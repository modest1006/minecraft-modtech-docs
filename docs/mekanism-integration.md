# Mekanism 連携 実装ノウハウ (NeoForge 1.21.1)

Mekanism `10.7.19.85` の API を対象にした連携調査。**全API名は実jar (`mekanism-10.7.19.85.jar`) を `javap` で確認済み**（2026-07-09）。
連携全体の考え方は [DESIGN.md](../DESIGN.md)「工業系MOD連携の設計方針」を前提とする。

> `mekanism.api.*` は API パッケージ（比較的安定）。Capability 定数は `mekanism.common.capabilities.Capabilities`（common）にある。連携コードは AE2 と同様に `compat/mekanism/` へ隔離し、`ModList.isLoaded("mekanism")` ガード＋`neoforge.mods.toml` の optional 依存で守る（[optional-compatパターン](../CLAUDE.md)）。

---

## 0. まず押さえる: エネルギー(FE)は連携済み

Mekanism は `Capabilities.ENERGY`（= NeoForge 標準の `IEnergyStorage`）を公開・受理する。
→ **既存の [`energy_machine`](../src/main/java/com/example/techlab/block/EnergyMachineBlock.java)（FE）は、追加コードなしで Mekanism のケーブル/発電機と電力をやり取りできる**。Mekanism独自の「Joule」を直接使いたい場合のみ `Capabilities.STRICT_ENERGY`(`IStrictEnergyHandler`) を実装する（通常は不要）。

Mekanism の主要 Capability（`mekanism.common.capabilities.Capabilities`）:

| 定数 | 型 | 用途 |
| --- | --- | --- |
| `ENERGY` | `MultiTypeCapability<IEnergyStorage>` | **FE**（標準・実質これでMekanism電力と繋がる） |
| `STRICT_ENERGY` | `MultiTypeCapability<IStrictEnergyHandler>` | Mekanism独自のJoule |
| `CHEMICAL` | `MultiTypeCapability<IChemicalHandler>` | **化学物質(chemical)** ← 本命 |
| `HEAT` | `BlockCapability<IHeatHandler, Direction>` | 熱 |

`MultiTypeCapability` は `.block()` / `.item()` / `.entity()` で NeoForge の `BlockCapability` 等を取り出せる。

---

## 1. Chemical（化学物質）システムの要点

**Mekanism 10.7 で chemical は統一済み**。以前の gas / infuse / pigment / slurry は単一の `mekanism.api.chemical.Chemical` / `ChemicalStack` に統合された（旧バージョンの「GasStack」等の記事はそのままでは通らない）。

- `Chemical` … 化学物質の定義（水素・酸素等）。レジストリは `MekanismAPI.CHEMICAL_REGISTRY`（`DefaultedRegistry<Chemical>`）。
- `ChemicalStack` … 量つきの実体。`new ChemicalStack(Holder<Chemical>, long)` または `(Chemical, long)`。量は **long**（mB）。`ChemicalStack.EMPTY`、`CODEC`/`STREAM_CODEC` あり。
- `mekanism.api.Action` … `EXECUTE` / `SIMULATE`（`FluidAction` と相互変換可）。

---

## 2. 機械で chemical を扱う（受け入れ/排出）

### 2-1. タンクを持つ
`mekanism.api.chemical.BasicChemicalTank` のファクトリで `IChemicalTank` を作る。第2引数以降の `IContentsListener` は変更通知（`this::setChanged` 相当）。

```java
// 入力専用タンク（8000mB、特定chemicalのみ許可も可）
private final IChemicalTank inputTank  = BasicChemicalTank.input(8_000L, this);
// 出力専用タンク
private final IChemicalTank outputTank = BasicChemicalTank.output(8_000L, this);
// 種類制限つき: BasicChemicalTank.input(cap, chemical -> chemical == MekanismChemicals.HYDROGEN.get(), listener)
```
`IChemicalTank` は `INBTSerializable<CompoundTag>`。NBT保存: `tag.put("InputChem", inputTank.serializeNBT(registries))` / `inputTank.deserializeNBT(registries, ...)`。

### 2-2. ハンドラを実装して公開
`mekanism.api.chemical.IMekanismChemicalHandler` を実装すると、`getChemicalTanks(Direction)` を返すだけで `IChemicalHandler` の各メソッド（insert/extract/getChemicalInTank…）が default で埋まる。

```java
public class MyMachineBE extends BlockEntity implements IMekanismChemicalHandler, IContentsListener {
    private final List<IChemicalTank> tanks = List.of(inputTank, outputTank);
    @Override public List<IChemicalTank> getChemicalTanks(@Nullable Direction side) { return tanks; }
    @Override public void onContentsChanged() { setChanged(); } // IContentsListener
}
```

### 2-3. Capability として公開（Mekanismのパイプと自動接続）
`RegisterCapabilitiesEvent` で `Capabilities.CHEMICAL.block()` を登録する。

```java
event.registerBlockEntity(
    Capabilities.CHEMICAL.block(),           // BlockCapability<IChemicalHandler, Direction>
    MyCompat.MY_MACHINE_BE.get(),
    (be, side) -> be);                        // be は IMekanismChemicalHandler(=IChemicalHandler)
```
これで隣接する Mekanism の Pressurized Tube 等がこの機械の chemical を搬入出できる。面ごとに入力/出力を分けたいなら `getChemicalTanks(side)` を面で切り替える。

---

## 3. 独自 Chemical を追加する（任意）

自分の化学物質を足すなら `DeferredRegister<Chemical>`（`MekanismAPI.CHEMICAL_REGISTRY_NAME`）に `ChemicalBuilder` で登録:

```java
public static final DeferredRegister<Chemical> CHEMICALS =
    DeferredRegister.create(MekanismAPI.CHEMICAL_REGISTRY_NAME, TechLab.MOD_ID);
public static final DeferredHolder<Chemical, Chemical> MY_GAS =
    CHEMICALS.register("my_gas", () -> ChemicalBuilder.builder().build());
```
（色・アイコン・属性は `ChemicalBuilder` のメソッドで設定。既存chemicalを使うだけなら不要。）

---

## 4. Mekanism レシピ連携（発展）

Mekanism 形式の加工レシピ（例: 電解・化学反応）に絡めるなら `mekanism.api.recipes.*` と `mekanism.api.recipes.ingredients.chemical.ChemicalIngredient`（レジストリ `MekanismAPI.CHEMICAL_INGREDIENT_TYPE_REGISTRY_NAME`）を使う。自作機械で独自加工を組むだけなら、[machine-patterns.md](machine-patterns.md) のカスタム `Recipe`＋chemicalタンクの組み合わせで十分。

---

## 5. 実装方針まとめ

1. **電力連携だけなら何もしなくてよい**（FE=`Capabilities.ENERGY` で既存機械がそのまま繋がる）。
2. **chemical を扱う機械**を作るなら: `BasicChemicalTank` を持ち → `IMekanismChemicalHandler` 実装 → `Capabilities.CHEMICAL.block()` を `RegisterCapabilitiesEvent` で公開。
3. コードは `compat/mekanism/` に隔離し、`ModList.isLoaded("mekanism")` ガード＋optional依存。別 DeferredRegister。→ 将来 別mod（Mekanismアドオン）へ切り出し可能。
4. API名に迷ったら実jarを `javap`：
   `jar = ~/.gradle/caches/.../maven.modrinth/mekanism/10.7.19.85/mekanism-10.7.19.85.jar`

## ハマりどころ

- **chemical は統一済み**（gas/infuse/pigment/slurryの区別は無い）。旧`GasStack`/`IGasHandler`前提の古い記事は読み替える。
- 量は **long**（intではない）。
- Capability定数は `mekanism.common.capabilities.Capabilities`（api ではなく common パッケージ）。`compat` 隔離必須。
- FEで足りるならFEを使う（Joule=STRICT_ENERGYは基本不要）。AE2と違い、Mekanismは電力・液体で標準Capabilityに乗ってくれる部分が多い。
