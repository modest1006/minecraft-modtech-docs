# 工業系マシン 実装ノウハウ (NeoForge 1.21.1)

加工マシン（在庫＋エネルギー＋GUI＋加工レシピ）を作るための中核パターン集。
**全API名は本プロジェクトの `neoforge-21.1.235-merged.jar` を `javap` で確認済み**（2026-07-09）。
概念は [NeoForge公式docs](https://docs.neoforged.net/docs/1.21.1/) を参照。まず [DESIGN.md](../DESIGN.md) の連携方針を前提とする。

> このドキュメントは「知見の蓄積」であり、コードは動作する骨子（スケッチ）。実装時はこの型に沿って肉付けする。

---

## 0. 加工マシンの全体像

「入れたアイテムをFEを使って別アイテムへ加工する機械」を例に、部品の関係を示す。

```
Block (EntityBlock)
  └─ BlockEntity  ← 状態の本体。毎tickで加工を進める
       ├─ ItemStackHandler   入出力スロット（Item Capabilityで公開 → 自動化MOD搬送）
       ├─ ModEnergyStorage   FEバッファ（Energy Capabilityで公開 → ケーブル受電）
       ├─ (FluidTank)        任意。液体を使うなら
       ├─ progress/maxProgress  加工の進捗（int）
       └─ MenuProvider 実装   右クリックでGUIを開く
  └─ Menu (AbstractContainerMenu)  サーバ/クライアント両方に存在。スロットとint同期を仲介
  └─ Screen (AbstractContainerScreen) クライアントのみ。見た目（進捗バー・エネルギーバー）
  └─ Recipe / RecipeType / RecipeSerializer  加工レシピの定義とJSONロード
```

キモは **「サーバが真実、クライアントは表示」**。スロット中身はMenuのSlotが自動同期、int値(進捗/エネルギー)は `ContainerData` で同期する。

---

## 1. インベントリ (ItemStackHandler + Item Capability)

`net.neoforged.neoforge.items.ItemStackHandler` を BlockEntity のフィールドに持つ。

```java
private final ItemStackHandler inventory = new ItemStackHandler(2) { // 0=input, 1=output
    @Override protected void onContentsChanged(int slot) { setChanged(); }
    @Override public boolean isItemValid(int slot, ItemStack stack) {
        return slot == 0; // 出力スロットには手で入れさせない等
    }
};
```

- NBT: `tag.put("Inventory", inventory.serializeNBT(registries));` / `inventory.deserializeNBT(registries, tag.getCompound("Inventory"));`
  （1.21では `serializeNBT(HolderLookup.Provider)` / `deserializeNBT(HolderLookup.Provider, CompoundTag)`。引数にレジストリが要る点に注意）
- Capability公開（搬送MODと連携）: `RegisterCapabilitiesEvent` で
  ```java
  event.registerBlockEntity(Capabilities.ItemHandler.BLOCK, MY_BE.get(),
      (be, side) -> be.getInventoryForSide(side)); // 面ごとに入力/出力を分けても良い
  ```
- 面制御: 入力面はinsertのみ、出力面はextractのみにしたいなら `IItemHandlerModifiable` をラップした「向き別ビュー」を返す（AE2/パイプが正しく搬入出できる）。

---

## 2. エネルギー (FE)

既に実装済みの [`energy/ModEnergyStorage`](../src/main/java/com/example/techlab/energy/ModEnergyStorage.java) を使う。加工マシンでは毎tickで「レシピ進行に必要なFEを `consume()`」し、足りなければ止める。Capability公開は [`ModCapabilities`](../src/main/java/com/example/techlab/ModCapabilities.java) と同型（`Capabilities.EnergyStorage.BLOCK`）。

---

## 3. カスタム加工レシピ

vanillaと同じ仕組みでJSON駆動にする。1.21のレシピAPIは `RecipeInput` ベース。

### 3-1. RecipeInput（機械の入力を表す）
単一入力なら既製の `SingleRecipeInput(ItemStack)` が使える。複数スロットは自作:
```java
public record GrindingInput(ItemStack input) implements RecipeInput {
    @Override public ItemStack getItem(int i) { return input; }
    @Override public int size() { return 1; }
}
```

### 3-2. Recipe 実装
```java
public record GrindingRecipe(Ingredient input, ItemStack result, int energy) implements Recipe<GrindingInput> {
    @Override public boolean matches(GrindingInput in, Level level) { return input.test(in.input()); }
    @Override public ItemStack assemble(GrindingInput in, HolderLookup.Provider reg) { return result.copy(); }
    @Override public ItemStack getResultItem(HolderLookup.Provider reg) { return result; }
    @Override public boolean canCraftInDimensions(int w, int h) { return true; }
    @Override public RecipeSerializer<?> getSerializer() { return ModRecipes.GRINDING_SERIALIZER.get(); }
    @Override public RecipeType<?> getType() { return ModRecipes.GRINDING_TYPE.get(); }
}
```

### 3-3. RecipeType と RecipeSerializer の登録
```java
// DeferredRegister<RecipeType<?>>(Registries.RECIPE_TYPE) と DeferredRegister<RecipeSerializer<?>>(Registries.RECIPE_SERIALIZER)
public static final Supplier<RecipeType<GrindingRecipe>> GRINDING_TYPE =
    RECIPE_TYPES.register("grinding", () -> RecipeType.simple(ResourceLocation.fromNamespaceAndPath(MODID, "grinding")));

public static final Supplier<RecipeSerializer<GrindingRecipe>> GRINDING_SERIALIZER =
    RECIPE_SERIALIZERS.register("grinding", () -> new GrindingSerializer());
```
`RecipeSerializer<T>` は2つを返すだけ:
```java
public MapCodec<GrindingRecipe> codec() { return CODEC; }          // JSON用
public StreamCodec<RegistryFriendlyByteBuf, GrindingRecipe> streamCodec() { return STREAM_CODEC; } // ネット同期用
```
CODEC は `RecordCodecBuilder.mapCodec(...)` で `Ingredient.CODEC`・`ItemStack.CODEC`・`Codec.INT` を組む。STREAM_CODEC は `StreamCodec.composite(...)`。

### 3-4. JSON（`data/<modid>/recipe/xxx.json`）
```json
{ "type": "techlab:grinding", "input": { "item": "minecraft:cobblestone" }, "result": { "id": "minecraft:gravel" }, "energy": 2000 }
```
タグ入力にすれば他MOD素材も受けられる（`"input": {"tag": "c:ingots/iron"}`）。

### 3-5. マシンからの参照（サーバ側tick）
```java
Optional<RecipeHolder<GrindingRecipe>> match =
    level.getRecipeManager().getRecipeFor(ModRecipes.GRINDING_TYPE.get(), new GrindingInput(inputStack), level);
match.ifPresent(holder -> { /* holder.value().assemble(...) で進捗を進める */ });
```

---

## 4. GUI（Menu + Screen）

### 4-1. MenuType 登録（追加ネットワークデータ付き）
```java
public static final Supplier<MenuType<GrinderMenu>> GRINDER_MENU =
    MENUS.register("grinder", () -> IMenuTypeExtension.create(GrinderMenu::new));
// IContainerFactory.create(int id, Inventory inv, RegistryFriendlyByteBuf buf) 相当
```

### 4-2. Menu（サーバ/クライアント両対応の2コンストラクタ）
```java
public class GrinderMenu extends AbstractContainerMenu {
    private final GrinderBlockEntity be;
    private final ContainerData data; // 進捗・エネルギー等のint同期

    // クライアント側（buf からBEを引く）
    public GrinderMenu(int id, Inventory inv, RegistryFriendlyByteBuf buf) {
        this(id, inv, (GrinderBlockEntity) inv.player.level().getBlockEntity(buf.readBlockPos()), new SimpleContainerData(2));
    }
    // サーバ側
    public GrinderMenu(int id, Inventory inv, GrinderBlockEntity be, ContainerData data) {
        super(ModMenus.GRINDER_MENU.get(), id);
        this.be = be; this.data = data;
        addSlot(new SlotItemHandler(be.getInventory(), 0, 56, 35)); // input
        addSlot(new SlotItemHandler(be.getInventory(), 1, 116, 35)); // output
        addPlayerInventory(inv); // 27+9スロットを並べる定型ループ
        addDataSlots(data); // int同期を登録
    }
    @Override public boolean stillValid(Player p) {
        return stillValid(ContainerLevelAccess.create(be.getLevel(), be.getBlockPos()), p, ModBlocks.GRINDER.get());
    }
    @Override public ItemStack quickMoveStack(Player p, int index) { /* moveItemStackTo で定型実装 */ }
    public int getProgress() { return data.get(0); }
    public int getEnergy()   { return data.get(1); }
}
```
- **int同期**: BE側は `getContainerData()` として `ContainerData`(`SimpleContainerData` か自作)を持ち、`get/set` で progress/energy を読み書き。`addDataSlots` すれば毎tick自動でクライアントへ送られる。short制約(±32767)を超える大きなエネルギー値は上位/下位に分割する。
- **スロット中身**は `SlotItemHandler` を addSlot するだけで自動同期。

### 4-3. GUIを開く（右クリック）
Block の `useWithoutItem`/`use` でサーバ側:
```java
if (!level.isClientSide && player instanceof ServerPlayer sp) {
    sp.openMenu(new SimpleMenuProvider(
        (id, inv, pl) -> new GrinderMenu(id, inv, be, be.getContainerData()),
        Component.translatable("block.techlab.grinder")),
        buf -> buf.writeBlockPos(pos)); // ← IContainerFactory へ渡す追加データ
}
```
BE を `MenuProvider` にしてもよい。

### 4-4. Screen（クライアントのみ）
```java
public class GrinderScreen extends AbstractContainerScreen<GrinderMenu> {
    // renderBg でGUIテクスチャ、進捗バーは menu.getProgress() を幅に反映して blit
}
```
登録は **クライアント専用**イベント `RegisterMenuScreensEvent`（MODバス）:
```java
@SubscribeEvent static void onRegisterScreens(RegisterMenuScreensEvent e) {
    e.register(ModMenus.GRINDER_MENU.get(), GrinderScreen::new);
}
```
→ [`TechLabClient`](../src/main/java/com/example/techlab/TechLabClient.java) 側（`dist=CLIENT`）に置く。

---

## 5. BlockEntity ⇄ クライアント同期

- **GUI内のint** → `ContainerData`/`addDataSlots`（上記）。
- **ワールド描画に必要な状態**（BERで描く、モデル差し替え等）→ `getUpdateTag(reg)` と `getUpdatePacket()`（`ClientboundBlockEntityDataPacket.create(this)`）を実装し、変化時に `level.sendBlockUpdated(pos, state, state, Block.UPDATE_CLIENTS)`。
- 発光のON/OFFのように **blockstateで足りる表示** は、これまで通り `level.setBlock` でstateを変えるのが最小コスト（[`energy_machine`](../src/main/java/com/example/techlab/block/EnergyMachineBlock.java) 方式）。

---

## 6. 流体 (Fluid)

`net.neoforged.neoforge.fluids.capability.templates.FluidTank` を持つ。

```java
private final FluidTank tank = new FluidTank(8000) { // 8バケツ
    @Override protected void onContentsChanged() { setChanged(); }
};
```
- 操作: `fill(FluidStack, FluidAction)` / `drain(int|FluidStack, FluidAction)`。`FluidAction.SIMULATE`/`EXECUTE`。
- NBT: `tank.writeToNBT(reg, tag)` / `tank.readFromNBT(reg, tag)`。
- Capability公開: `Capabilities.FluidHandler.BLOCK` を `registerBlockEntity`。→ パイプ/タンクMODと自動連携。
- 液体の種類制限は `new FluidTank(cap, fs -> fs.is(...))` のバリデータで。

---

## 7. データ生成 (datagen)

要素が増えたら手書きJSONをやめて datagen 化する。`GatherDataEvent`（MODバス）で各Providerを登録:

```java
@SubscribeEvent static void gather(GatherDataEvent e) {
    DataGenerator gen = e.getGenerator();
    PackOutput out = gen.getPackOutput();
    ExistingFileHelper efh = e.getExistingFileHelper();
    CompletableFuture<HolderLookup.Provider> lookup = e.getLookupProvider();
    gen.addProvider(e.includeClient(), new MyBlockStateProvider(out, efh));
    gen.addProvider(e.includeClient(), new MyItemModelProvider(out, efh));
    gen.addProvider(e.includeClient(), new MyLanguageProvider(out, "en_us"));
    gen.addProvider(e.includeServer(), new MyRecipeProvider(out, lookup));
    gen.addProvider(e.includeServer(), MyLootProvider.create(out, lookup));
    gen.addProvider(e.includeServer(), new MyBlockTagsProvider(out, lookup, efh));
}
```
確認済みのProviderクラス（全て存在）:
- `net.neoforged.neoforge.client.model.generators.BlockStateProvider` / `ItemModelProvider`
- `net.minecraft.data.recipes.RecipeProvider`
- `net.minecraft.data.loot.LootTableProvider`（+ `BlockLootSubProvider`）
- `net.minecraft.data.tags.*`（`BlockTagsProvider`/`ItemTagsProvider` は NeoForge/vanilla 各基底）
- `net.neoforged.neoforge.common.data.LanguageProvider`

出力は `build.gradle` の run `data` が `src/generated/resources` に書き出し、sourceSet設定でビルドに取り込まれる（`.\gradlew.bat runData`）。**テクスチャPNGだけは自作**。移行時は手書きJSONを削除して二重管理を避ける。

> **このプロジェクトでは導入済み**（`com.example.techlab.datagen`、エントリ `DataGenerators`）。実際の Provider 実装・generated/手書きの切り分け・optional タグの扱いは `DESIGN.md`「データ生成」を参照。

---

## 8. AE2 ストレージ提供（次のAE2段の下調べ）

グリッド接続（[`compat/ae2/me_connector`](../src/main/java/com/example/techlab/compat/ae2)）の次段。自作インベントリをMEネットワークに載せる:

- ノードサービスとして `appeng.api.networking.storage.IStorageProvider` を実装し、`mountInventories(IStorageMounts)` で `MEStorage` を提供。中身が変わったら `IStorageProvider.requestUpdate()`。
- `MEStorage` は `AEKey`（`AEItemKey`/`AEFluidKey`）単位。`insert/extract/getAvailableStacks`。
- `IManagedGridNode` に `.addService(IStorageProvider.class, this)` で登録してからグリッドへ。
- 実APIシグネチャは実装直前に `javap` で確認する（jar: `~/.gradle/caches/.../maven.modrinth/ae2/19.2.17/ae2-19.2.17.jar`）。

---

## ハマりどころ

- `ItemStackHandler.serializeNBT/deserializeNBT` は **1.21で `HolderLookup.Provider` 引数が必須**。旧記事の無引数版はコンパイルエラー。
- `ContainerData` の同期は **short幅**。大きなエネルギー値は分割して2スロットで送る。
- Screen登録は `RegisterMenuScreensEvent`（**クライアント専用**）。共通クラスに書くと専用サーバでクラッシュ。
- Menuのクライアント/サーバ2コンストラクタで、クライアント側は `buf` からBEを引く。`stillValid` のクライアント側 access は `ContainerLevelAccess.NULL`。
- レシピは `RecipeManager.getRecipeFor(type, input, level)` が `Optional<RecipeHolder<T>>` を返す。`RecipeHolder.value()` が実レシピ、`.id()` がID。
- `Recipe.assemble` の結果は必ず `.copy()` を返す（元resultを共有しない）。
