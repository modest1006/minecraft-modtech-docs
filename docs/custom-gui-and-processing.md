# カスタム GUI と加工機能 (NeoForge 1.21.1)

「独自のマシンに GUI と加工レシピを持たせる」実装。assembler で実装済み（2026-07-10）。
関連: [machine-patterns.md](machine-patterns.md) の Menu/Screen 節がベース。

---

## 全体の関係

```
   [BlockEntity]  ← 頭脳(在庫・進捗・エネルギー・レシピ検索)
        │
        ├─ ItemStackHandler(3)  スロット 入力A/入力B/出力
        ├─ ModEnergyStorage     FE(既存)
        ├─ progress / MAX_PROGRESS
        └─ ContainerData(4)     GUI に int を同期する経路(short幅 x4)
        │
   [Menu (AbstractContainerMenu)] ← サーバ/クライアント両方に存在
        │  SlotItemHandler で在庫を addSlot
        │  addDataSlots(ContainerData) で int同期
        │  quickMoveStack / stillValid を実装
        │
   [Screen (AbstractContainerScreen<Menu>)] ← クライアントのみ
        │  renderBg で背景 / スロット / 進捗矢印 / エネルギーバー
        │  renderLabels で文字
        │  renderTooltip でホバーツールチップ
```

- **キモ**: サーバが真実、クライアントは表示。スロット中身は `SlotItemHandler` が自動同期、int は `ContainerData` で送る。

## 実装

### 1) カスタムレシピ型

- `crafting/AssemblerRecipeInput`（`RecipeInput` 実装、2入力レコード）
- `crafting/AssemblerRecipe`（`Recipe<AssemblerRecipeInput>`）
  - `matches` を**順不同**にする（入力Aがどちらのスロットにあってもマッチ）
  - `getSerializer` / `getType`
  - **`Serializer`**: `MapCodec`（JSON用）＋ `StreamCodec`（ネット同期用）
    - `Ingredient.CODEC_NONEMPTY.fieldOf("input_a")` … 空`Ingredient`禁止
    - `ItemStack.STRICT_CODEC.fieldOf("result")` … `{"id":"...","count":1}` 形式
    - `Codec.INT.fieldOf("energy")`
  - ✅ **修正済み（2026-07-11）**: 旧実装は固定値 40 FE/tick でレシピの `energy` が無視されていた。現在は `energyCostForTick(totalEnergy, progress)` で **レシピの energy を tick 割で消費**（端数は最終 tick でまとめて消費 → 総消費が energy と厳密一致）。GameTest `assemblerProcessesRecipe` が「1000 FE ちょうど与えて完了時に残 0」で回帰を守っている（[review-2026-07-11.md](review-2026-07-11.md) 指摘1）。
- `crafting/ModRecipes`: `RecipeType.simple(id)` と Serializer を `DeferredRegister<RecipeSerializer<?>>`/`<RecipeType<?>>` に登録
- サンプル JSON: `data/techlab/recipe/*.json` に手書き（レシピは datagen 化する価値もあるが型を登録すれば手書きJSONで十分ロードされる）

### 2) BlockEntity 側

- `ItemStackHandler(3)` を持ち `onContentsChanged` で `setChanged`、`isItemValid(2, ..)` で出力挿入を拒否
- `progress` int
- `ContainerData` を **short幅制約**に注意して 4スロットに（progress / MAX_PROGRESS / energy上位16bit / 下位16bit）
- `serverTick` で:
  1. `level.getRecipeManager().getRecipeFor(TYPE, new AssemblerRecipeInput(...), level)`
  2. マッチ && 出力に空き && エネルギー十分 → `canWork=true`
  3. `canWork` を **blockstate ACTIVE に反映** → **状態駆動アニメが自動追随**
  4. `progress++`、`MAX_PROGRESS` に到達したら入力消費・出力生成・progress=0
  5. 中断時は progress=0
- NBT: `Inventory` `Progress` `Energy` `Formed` `Active` を保存/復元

### 3) Menu

- `IMenuTypeExtension.create(AssemblerMenu::new)` で MenuType 登録（`IContainerFactory` 経由なので**クライアントコンストラクタで `RegistryFriendlyByteBuf` から BlockPos を読める**）
- 2コンストラクタ:
  - client: `(id, playerInv, buf)` — buf から pos を読み、ダミーの `ItemStackHandler(3)` と `SimpleContainerData(4)` を作る
  - server: `(id, playerInv, pos, machineInv, data)` — 実 BE の在庫と ContainerData を渡す
- `addSlot(new SlotItemHandler(machineInv, i, x, y))` × 3、プレイヤーインベントリ (27+9) を定型ループで
- `addDataSlots(data)` で int同期を登録
- `stillValid(Player)`：ここでは距離チェックのみ（`ContainerLevelAccess.create` で厳密化も可）
- `quickMoveStack`：スロット index で機械↔プレイヤーの移動先を出し分ける

### 4) Screen

**テクスチャなし版**（この実装）: `GuiGraphics.fill(x1,y1,x2,y2, ARGB)` で全部矩形描画。GUIテクスチャ探しが不要でお手軽:
- 背景パネル（枠付き）
- スロット窪み（枠 + 塗り）を各スロット位置に描く（`SlotItemHandler` が上からアイテムを描く）
- 進捗矢印: `menu.getProgress() / getMaxProgress()` の比率で矩形を伸ばす
- エネルギーバー: `menu.getEnergy() / getMaxEnergy()` の比率で下から積み上げる
- タイトル/インベントリ名は `renderLabels`
- ツールチップ: `renderTooltip` に加えて手動 `g.renderComponentTooltip` でホバー時のエネルギー数値表示

**本格版**では 256×256 の PNG を `assets/<modid>/textures/gui/` に置いて `g.blit(texture, x, y, u, v, w, h, 256, 256)`。テクスチャに背景・矢印(埋)・バー(埋) が入っていて、進捗に応じて `w` を狭めて描く。

登録は `TechLabClient.onRegisterMenuScreens`（`RegisterMenuScreensEvent`, クライアント専用）→
`event.register(ModMenus.ASSEMBLER_MENU.get(), AssemblerScreen::new)`。

### 5) GUI を開く

Block の `useWithoutItem` サーバ側で:
```java
sp.openMenu(new SimpleMenuProvider(
        (id, inv, p) -> new AssemblerMenu(id, inv, pos, be.getInventory(), be.getContainerData()),
        Component.translatable("container.techlab.assembler")),
    buf -> buf.writeBlockPos(pos));
```
- 第2引数の `Consumer<RegistryFriendlyByteBuf>` は `IMenuTypeExtension.create` を使ったときの追加ネットワーク引数。クライアント側 Menu コンストラクタが同じ buf を読む。

---

## 状態駆動アニメとの連動

前段の [state-driven-animation.md](state-driven-animation.md) で作った ACTIVE トグルを、`serverTick` の `canWork` に**乗っ取らせる**:
- 「レシピ材料入っている＋出力に余裕＋エネルギー有」→ `ACTIVE=true` → **自動起動＋炉心が加速＋オービット出現**
- 材料が尽きる/出力が満杯 → `ACTIVE=false` → 自動停止＋炉心減速＋オービット消滅

→ GUI に材料を入れるだけでマシンが自動で稼働アニメを始める。切り分けたモジュール同士が状態駆動でつながる好例。

## 応用パターン

- **エネルギー可視化**: バーだけでなく `x FE/tick` の消費速度も同期して表示
- **加工時間の可変化**: レシピごとに `MAX_PROGRESS` を持たせる（Recipe の追加フィールド）
- **バッチ加工**: 入力から複数個消費・出力に複数個生成
- **音・パーティクル**: `canWork` 中に一定間隔で `level.playSound` / `level.addParticle`
- **Item Capability の公開**: `ItemStackHandler` を `Capabilities.ItemHandler.BLOCK` に登録 → パイプで自動搬入出（`ModMultiblock` の面別 view で入力面/出力面を分離）

## ハマりどころ

- **`ContainerData` は short 幅**。int(0..2^31)を送るときは上位/下位16bitに分割する定石。
- **Menu の2コンストラクタ**: `IMenuTypeExtension.create(...)` は「buf を読むクライアント側」を要求。書き忘れると `NoSuchMethodException`。
- **Screen 登録はクライアントイベント** `RegisterMenuScreensEvent`。共通側で `MenuScreens.register` すると専用サーバでクラッシュ。
- **`Recipe.assemble` は必ず `.copy()`**（result を共有すると壊す）。
- **`isSameItemSameComponents`**: 1.20.5 以降は NBT ではなく **data component** で比較。旧記事の `ItemStack.matches`/`isSameItemSameTags` は 1.21 では別物。
- **`Ingredient.CODEC_NONEMPTY`** / **`ItemStack.STRICT_CODEC`** … 名前を間違えると Codec 解決失敗で無言でロードされない。javap でシグネチャ確認が確実。
