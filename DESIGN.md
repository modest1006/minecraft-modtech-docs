# DESIGN.md — 設計メモと開発知見

MOD の設計方針・拡張手順・ハマりどころを蓄積するドキュメント。ビルド/コマンドなど「操作」は `CLAUDE.md` に、ここには「なぜ・どう作るか」を書く。

## このMODの内容（現状）

学習用のサンプル一式。「サファイア」テーマで、登録の基本形（アイテム／ブロック／BlockItem／クリエイティブタブ／レシピ／ドロップ／タグ／多言語）を最小構成で通している。

| 要素 | id | 説明 |
| --- | --- | --- |
| アイテム | `techlab:sapphire` | 素材アイテム |
| ブロック | `techlab:sapphire_block` | 適正ツール(鉄以上のツルハシ)で採掘、自身をドロップ |
| タブ | `techlab:techlab_tab` | 上記2つを並べる独自クリエイティブタブ |
| レシピ | 2種 | sapphire×9 ⇄ sapphire_block |

## パッケージ設計

```
com.example.techlab
├─ TechLab            エントリポイント(@Mod)。各レジストリを配線し、ライフサイクルを購読
├─ TechLabClient      クライアント専用(@Mod dist=CLIENT)。config画面登録など
├─ Config           ModConfigSpec のサンプル(techlab-common.toml を生成)
├─ block
│   └─ ModBlocks    ブロック定義。ブロック登録時にBlockItemも同時登録
└─ item
    ├─ ModItems            アイテム定義
    └─ ModCreativeModeTabs 独自クリエイティブタブ
```

**依存の向き**: `ModBlocks → ModItems`（ブロックが自分のBlockItemをItemsへ登録するため）。逆向き参照（ItemsからBlocks）を作ると静的初期化が循環し得るので禁止。タブは末端なので両方を参照してよい。

## 拡張手順（レシピ集）

### 新しいアイテムを追加

1. `ModItems` に 1 行:
   ```java
   public static final DeferredItem<Item> RUBY =
           ITEMS.registerSimpleItem("ruby", new Item.Properties());
   ```
2. `assets/techlab/models/item/ruby.json`（`minecraft:item/generated` + `layer0: techlab:item/ruby`）
3. `assets/techlab/textures/item/ruby.png`（16×16）
4. `lang/en_us.json` / `ja_jp.json` に `"item.techlab.ruby"` を追加
5. 見せたいなら `ModCreativeModeTabs` の `displayItems` に `output.accept(ModItems.RUBY.get());`

### 新しいブロックを追加

1. `ModBlocks` に `registerBlock("ruby_block", () -> new Block(...properties...))`（BlockItem は自動登録される）
2. `blockstates/ruby_block.json`（variants → `techlab:block/ruby_block`）
3. `models/block/ruby_block.json`（`cube_all` + texture）
4. `models/item/ruby_block.json`（`parent: techlab:block/ruby_block`）
5. `textures/block/ruby_block.png`
6. `data/techlab/loot_table/blocks/ruby_block.json`（採掘ドロップ。無いと壊してもドロップしない）
7. 採掘制限を付けるなら `data/minecraft/tags/block/mineable/<tool>.json` と ティアタグへ追加
8. `lang` に `"block.techlab.ruby_block"`

### 食べ物・道具・防具など

`Item.Properties()` に `.food(...)`、`SwordItem`/`PickaxeItem` などのサブクラスやコンポーネントで表現する。1.21 は Item コンポーネント(data component)体系なので、旧来の NBT ベースの記事は読み替えが必要。

## データ生成 (datagen) — 導入済み

コアの定型JSONは **datagen 化済み**（手書きから移行完了）。実装は `com.example.techlab.datagen`:

| Provider | 生成物 |
| --- | --- |
| `ModBlockStateProvider` | blockstates / block models / block item models（`simpleBlockWithItem`・LIT variantは`getVariantBuilder`）＋ 通常アイテムモデル（`itemModels().basicItem`） |
| `ModRecipeProvider` | クラフトレシピ（`ShapedRecipeBuilder`/`ShapelessRecipeBuilder`、`save(output, id)`でファイル名固定）＋ レシピ解放advancement |
| `ModBlockLootProvider`(+`LootTableProvider`) | ブロックのドロップ表（`dropSelf`） |
| `ModBlockTagsProvider` | `mineable/pickaxe`・`needs_iron_tool`・`c:storage_blocks` |
| `ModItemTagsProvider` | `c:gems`・`c:storage_blocks`（item側。ブロックタグの`contentsGetter()`を受け取る） |
| `ModEnUsProvider`/`ModJaJpProvider` | `lang/en_us.json`・`ja_jp.json` |

エントリは `DataGenerators.gatherData(GatherDataEvent)`（`TechLab` コンストラクタで `modEventBus.addListener`）。

**運用**:
- 生成: `.\gradlew.bat runData` → `src/generated/resources` に出力。`build.gradle` の sourceSet 設定でビルドに含まれる。要素を足したら **runData を再実行**。
- **テクスチャPNGは対象外**（自前で用意。`assets/techlab/textures/` に手動配置）。
- **二重管理禁止**: datagenが吐くファイルの手書きは削除する（同一パスが main と generated に両方あるとビルドで重複衝突する）。

**設計上のポイント（学び）**:
- **compat/ae2 の me_connector は datagen対象外**（datagenをAE2に依存させないため、blockstate/model/loot は手書き維持）。ただし共有ファイル（タグ・言語）だけは datagen 側でまとめて出す。
- me_connector のタグ登録は `addOptional(ResourceLocation)` で **optional エントリ**（`{"id":...,"required":false}`）にした。→ AE2 未導入時でもタグ読み込みエラーにならない（手書き時代の潜在バグも解消）。言語は文字列キー `add("block.techlab.me_connector", ...)` で追加（AE2非参照）。

## テクスチャ運用

- 現在の PNG は System.Drawing で生成したプレースホルダ（青系の単純パターン）。本格的なテクスチャは BlockBench 等で作成する。
- 仕様: 16×16 PNG、アイテムは透過背景可、ブロックは全面不透過。

## GameTest / テスト方針

- `build.gradle` で `neoforge.enabledGameTestNamespaces=techlab` 済み。`@GameTest` を書いて `.\gradlew.bat runGameTestServer` で実行、またはゲーム内 `/test` コマンド。
- 純ロジック（レジストリに依存しない計算等）は将来 `src/test` に JUnit を足す余地あり（現状は未設定）。

## 実装ノウハウ（詳細）

実APIシグネチャ（`javap`で確認済み）付きの詳細ノウハウ:

- [docs/machine-patterns.md](docs/machine-patterns.md) … 加工マシンの中核（**在庫 ／ エネルギー ／ カスタム加工レシピ ／ GUI(Menu+Screen) ／ クライアント同期 ／ 流体 ／ datagen ／ AE2ストレージ提供**）。新しいマシンを作るときはまずこれ。
- [docs/mekanism-integration.md](docs/mekanism-integration.md) … Mekanism連携（chemical/gas統一システム、`Capabilities.CHEMICAL`、`BasicChemicalTank`＋`IMekanismChemicalHandler`）。**FEは連携済み**（既存機械がそのままMekanism電力と繋がる）。
- [docs/immersive-engineering-integration.md](docs/immersive-engineering-integration.md) … IE連携。**FEは連携済み**＋**IE機械レシピはJSONで追加（コード不要）**。多ブロック/鉱脈は `api/*` を compat 隔離で。
- [docs/multiblock-architecture.md](docs/multiblock-architecture.md) … マルチブロック（複数ブロックの大型装置）のアーキテクチャ。コントローラ/パーツ方式、vanilla `BlockPattern` での形成検出、Capability委譲、向き・永続化・レンダリング、IEフレームワークの構造。
- [docs/moving-blocks-collision.md](docs/moving-blocks-collision.md) … 動くブロック（エレベーター/自動ドア）と当たり判定の各アプローチ。A:テレポート/瞬間、B:VoxelShape段階アニメ、C:ピストン式移動ブロック、D:エンティティ化コントラプション（Create/Moving Elevators方式）。

## 工業系MOD連携の設計方針

このMODの本題は「工業系MODと連携する機械・仕組み」。連携の設計原則は **"標準に寄せる"**。

### 1. 素材は共通タグ (`c:`) でやり取りする

- 自分が追加する素材（インゴット/鉱石/ダスト/ギア等）は必ず対応する `c:` タグに登録する → 他MODのレシピ・機械が拾える。
- 他MODの素材を使うレシピは、特定MODのアイテムIDではなく `{"tag": "c:ingots/steel"}` のようにタグで受ける → どのMODのsteelでも動く。
- 未対応の素材種別（独自の合金など）は `c:ingots/<myalloy>` を自分で切って規約に合わせる。

### 2. エネルギー/液体/搬送は NeoForge Capability で繋ぐ

相手MODのAPIに依存しないのが基本。BlockEntity に Capability を実装し `RegisterCapabilitiesEvent` で公開する。

| 連携 | Capability | 型 | 効果 |
| --- | --- | --- | --- |
| エネルギー(FE/RF) | `Capabilities.EnergyStorage.BLOCK` | `IEnergyStorage` | ケーブル/発電機/機械とエネルギー授受 |
| 液体 | `Capabilities.FluidHandler.BLOCK` | `IFluidHandler` | パイプ/タンクと液体授受 |
| アイテム | `Capabilities.ItemHandler.BLOCK` | `IItemHandler` | パイプ/ホッパー/搬入出 |

- これらを実装した機械は、Mekanism/IE/Thermal 等と**追加コードなしで相互運用**できる（各MODがこの標準に対応しているため）。
- 独自エネルギー単位を作らない。FE(Forge Energy)/mB(液体)の標準に合わせる。

### 3. 相手MOD固有APIに触るのは最後の手段

Mekanism の gas/chemical、AE2 の ME ネットワーク等、標準 Capability で表現できない独自システムに関与する時だけ、そのMODを `compileOnly` で追加してAPIを叩く。ハード依存になるので `neoforge.mods.toml` の dependencies も更新し、optional/required を明示する。

### 実装例: FEエネルギー機械（`energy_machine`）

統一Capabilityによる連携の最初の実装。他MODのケーブル/発電機からFEを受け取る「消費専用」機械。

- 構成:
  - `energy/ModEnergyStorage` … NeoForge `EnergyStorage` を拡張。受電時に `onChanged`(=`setChanged`) を呼び、内部消費 `consume()` と NBT復元 `setEnergy()` を追加。
  - `block/entity/EnergyMachineBlockEntity` … 蓄電(容量10万FE, 受電上限2000FE/t)を持ち、毎tickわずかに消費。蓄電>0なら `LIT=true`。NBTで蓄電を永続化。
  - `block/EnergyMachineBlock`（`BaseEntityBlock`）… `LIT` プロパティ、`RenderShape.MODEL`（BaseEntityBlockの既定INVISIBLEを上書き）、サーバ側tickerを`getTicker`で供給。
  - `block/entity/ModBlockEntities` … `BlockEntityType` 登録。
  - `ModCapabilities` … `RegisterCapabilitiesEvent` で `Capabilities.EnergyStorage.BLOCK` に BlockEntity を登録（`TechLab`コンストラクタで `modEventBus.addListener` 経由）。
- 動作確認: クリエイティブで `energy_machine` を設置 → 隣に発電機＋ケーブル(IE/Mekanism等)を繋ぐ → 給電されると発光(LIT, 光量13)。電源を外すと蓄電を消費して消灯。
- ここに `maxExtract>0` にすれば送電もできる。液体/アイテムも同様に `Capabilities.FluidHandler.BLOCK` / `ItemHandler.BLOCK` を `registerBlockEntity` すれば連携が広がる。
- **注意**: `BaseEntityBlock` は `codec()` 実装必須、`getRenderShape` を MODEL に上書きしないと見えない。ticker はサーバ側のみ返す。

### AE2連携（調査結果）

AE2は標準Capabilityで表せない独自のMEネットワークを持つため、`appeng.api.*`（本体jar同梱、`compileOnly` 済み）を直接使う。公式: [API.md](https://github.com/AppliedEnergistics/Applied-Energistics-2/blob/main/API.md) / [javadoc](https://appliedenergistics.org/javadoc/)。

- **グリッド接続**（機械をMEに繋ぐ／連携の入口）:
  - BlockEntity が `appeng.api.networking.IInWorldGridNodeHost` を実装（`getGridNode(Direction)`）。
  - ノードは `GridHelper.createManagedNode(this, listener)` で `IManagedGridNode` を作り、`.setInWorldNode(true)`・`.setVisualRepresentation(item)`・`.setIdlePowerUsage(x)` を設定。
  - ライフサイクル厳守: ロード時 `loadFromNBT()` → **初tickで** `create(level,pos)`、保存時 `saveToNBT()`、破棄/アンロードで `destroy()`。
  - 隣接AE2ケーブルとの接続はAE2が自動。NeoForgeでは node host を API Lookup(Capability) で公開する必要がある。
- **ストレージ提供**: ノードサービス `IStorageProvider`（`mountInventories()` / `requestUpdate()`）＋統一 `MEStorage`(`AEKey`)。
- **エネルギー**: `IGrid#getEnergyService()`(`IEnergyService`)。ノードは `idlePowerUsage` を消費。AE2はFEを直接出さない（Energy AcceptorがFE→AE変換）。
- **オートクラフト**: `ICraftingProvider`(パターン提供) / `ICraftingService`。
- **構造方針**: AE2連携は独立性が高いので `compat/ae2` パッケージ（将来は別モジュール/別mod=アドオン構成）に隔離する。AE2をハード依存にするなら `neoforge.mods.toml` に optional/required 依存を明示。
- **推奨初手**: グリッド接続するBlockEntity（"hello ME network"）。接続状態を可視化 → 次段でストレージ提供 or オートクラフトへ拡張。

#### 実装済みサンプル: `me_connector`（compat/ae2）

MEネットワークに接続し、グリッドが有効になると発光するサンプル機械。

- `compat/ae2/MeConnectorBlockEntity` … `IInWorldGridNodeHost` を実装。`GridHelper.createManagedNode(this, listener).setInWorldNode(true).setIdlePowerUsage(1.0)` でノード生成。ライフサイクル: `loadAdditional`→`loadFromNBT`、初tick→`create`、`saveAdditional`→`saveToNBT`、`setRemoved`→`destroy`、`clearRemoved`→再生成フラグ。`getGridNode(dir)` は `mainNode.getNode()` を返す。
- `compat/ae2/MeConnectorBlock` … `BaseEntityBlock`、`CONNECTED` プロパティ、サーバtickで `mainNode.isActive()` を見て CONNECTED/発光を更新。
- `compat/ae2/Ae2Compat` … 専用 DeferredRegister（本体と分離）でブロック/アイテム/BEを登録。`RegisterCapabilitiesEvent` で **`AECapabilities.IN_WORLD_GRID_NODE_HOST`** に BE を登録（隣接ケーブルが `getGridNode()` を見つけられるようにする）。タブ追加は `BuildCreativeModeTabContentsEvent`。

**オプショナル連携パターン（重要）**: `compat/ae2` は `appeng.api.*` を直接参照するので、AE2 が無い環境でクラスロードされると `NoClassDefFoundError` になる。これを避けるため:
1. AE2依存コードを `compat/ae2` パッケージに隔離し、本体（core）から直接参照しない。
2. `TechLab` コンストラクタで `ModList.get().isLoaded("ae2")` を確認してから `Ae2Compat.register()` を呼ぶ（初めてクラスロードされる箇所を守る）。ガード条件で使う mod id は `static final String` 定数（コンパイル時にインライン化され、クラスロードを誘発しない）。
3. `neoforge.mods.toml` に AE2 を `type="optional"`, `ordering="AFTER"` で明示。
- これで「AE2があればAE2連携サンプルが有効、無ければ本体だけ動く」状態になり、将来そのまま別mod（AE2アドオン）へ切り出せる。
- 実機確認: `me_connector` を設置 → AE2コントローラ＋エネルギー(Energy Acceptorで給電)＋ケーブルで繋ぐ → グリッドが起動しチャンネルが通ると発光。

### JEI 連携（任意）

独自の機械・加工レシピを追加したら、JEIプラグイン（`@JeiPlugin` 実装 + `compileOnly` のJEI API）でカテゴリ表示・使用先/入手元を出せる。まずは動作優先で、レシピ表示は後追いでよい。

## 既知のハマりどころ（追記していく）

- **1.21 のデータパック単数形フォルダ**: `recipe/`, `loot_table/`, `advancement/`, `tags/block/`, `tags/item/`。旧バージョンの記事をコピペすると読み込まれず、エラーも出にくいので要注意。
- **BlockItem を登録し忘れる**とインベントリに持てない（`ModBlocks.registerBlock` が自動でやる設計）。
- **共通クラスからクライアント専用 API を触る**と専用サーバでクラッシュ。`TechLabClient` 側へ。
- **`neoforge.mods.toml` はテンプレート**。`${...}` を壊さない。実体は `build/generated/sources/modMetadata` に生成される。
- **バージョン更新**は `gradle.properties` の `neo_version` 等を変え、`--refresh-dependencies` で再取得。NeoForge の最新版は maven-metadata（`https://maven.neoforged.net/releases/net/neoforged/neoforge/maven-metadata.xml`）で確認できる。
