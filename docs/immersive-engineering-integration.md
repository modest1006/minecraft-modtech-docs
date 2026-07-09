# Immersive Engineering 連携 実装ノウハウ (NeoForge 1.21.1)

Immersive Engineering `12.4.2-194` を対象にした連携調査。API・レシピ形式は実jar (`immersiveengineering-12.4.2-194.jar`) を `javap`／同梱データで確認済み（2026-07-09）。

> 結論から言うと、IE連携の多くは **(1) FE（コード不要）** と **(2) JSONレシピ追加（コード不要）** で足りる。IE固有APIの `compileOnly` が要るのは多ブロック追加など踏み込んだ場合のみ。

---

## 1. エネルギー(FE)は連携済み（コード不要）

IE の `blusunrize.immersiveengineering.api.energy.MutableEnergyStorage` は
`net.neoforged.neoforge.energy.EnergyStorage`（FE）を継承。IE の LV/MV/HV ワイヤコネクタは
隣接する `IEnergyStorage` へ FE を供給する。

→ **既存の [`energy_machine`](../src/main/java/com/example/techlab/block/EnergyMachineBlock.java)（FE）は、IEのコネクタから追加コードなしで受電できる**。Mekanism と同じ。
表示を滑らかにしたいときだけ IE の `AveragingEnergyStorage` を参考にする（必須ではない）。

---

## 2. IE機械にレシピを追加する（★最も実用的・コード不要）

IE の各多ブロック機械のレシピは **データ駆動（JSON）**。自分のMODの `data/<modid>/recipe/…json` に
IEのレシピ型で書くだけで、IEの機械がそれを認識する（`compileOnly` 不要）。

確認済みの実例（IE同梱 `data/immersiveengineering/recipe/crusher/amethyst.json`）:
```json
{
  "type": "immersiveengineering:crusher",
  "energy": 3200,
  "input": { "item": "minecraft:amethyst_block" },
  "result": { "count": 4, "id": "minecraft:amethyst_shard" }
}
```
→ 例えば自分の鉱石を Crusher で砕けるようにするレシピを、自MODのデータパックに置ける。`input` をタグ（`{"tag":"c:ores/..."}`）にすれば他MOD素材にも効く。

主なレシピ型（`type`）:

| 機械 | type |
| --- | --- |
| Crusher（粉砕） | `immersiveengineering:crusher` |
| Metal Press（金型プレス） | `immersiveengineering:metal_press` |
| Arc Furnace（アーク炉） | `immersiveengineering:arc_furnace` |
| Blast Furnace（高炉） | `immersiveengineering:blast_furnace` |
| Alloy（合金化・高炉） | `immersiveengineering:alloy` |
| Mixer / Squeezer / Fermenter / Refinery / Bottling / Sawmill / Coke Oven / Cloche | 各 `immersiveengineering:<machine>` |

各型の正確なフィールドは **IE本体の同梱JSONをテンプレにするのが確実**（jar内 `data/immersiveengineering/recipe/<machine>/*.json` を開いて真似る）。Metal Press は `mold`（金型）指定、Arc Furnace は `additives`/`time` 等が要る。JSONなので datagen 化も可能だが、まずは手書きで十分。

---

## 3. IE固有API（踏み込む場合のみ・compileOnly）

`blusunrize.immersiveengineering.api.*`（本体jar同梱）を使う。主な領域:

- `api/multiblocks` … IEスタイルの多ブロック機械を**自作**する枠組み（大掛かり）。
- `api/wires` … IEのワイヤ網に直接参加する（通常はFE公開で足りるので不要）。
- `api/excavator` … Excavator多ブロックの鉱脈(mineral vein)に自分の鉱石を追加。
- `api/crafting` … 上記レシピ型のコード側定義（JSONで足りるので普通は触らない）。
- `api/tool` … Engineer's Hammer/Wirecutter 等の相互作用。

これらは AE2/Mekanism と同じく `compat/immersiveengineering/` に隔離し、`ModList.isLoaded("immersiveengineering")` ガード＋optional依存で守る。

---

## まとめ（IE連携の優先順）

1. **電力**: 何もしなくてよい（FE＝`IEnergyStorage`で繋がる）。
2. **IE機械で自分の素材を加工**: `data/<modid>/recipe/` に JSON を置く（コード不要）。タグ入力で汎用化。
3. **踏み込んだ連携**（多ブロック自作・鉱脈追加）: `api/multiblocks`・`api/excavator` を `compat/` 隔離で。
4. 迷ったら実jar/同梱JSONを確認：
   `jar = ~/.gradle/caches/.../maven.modrinth/immersiveengineering/12.4.2-194/immersiveengineering-12.4.2-194.jar`

## ハマりどころ
- IE機械レシピは **vanillaのcrafting型ではなくIE独自type**。`data/<modid>/recipe/` 直下でよい（1.21の単数形 `recipe/`）。
- レシピ型ごとに必須フィールドが違う。**IE同梱JSONをコピーして改変**が最短・最確実。
- IEのエネルギーはFEなので、Mekanism・Thermal 等とも同じ土俵。独自エネルギー実装は不要。
