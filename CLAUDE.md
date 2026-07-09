# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

Minecraft Java 版の MOD プロジェクト。**NeoForge** ローダー向け。

| 項目 | 値 |
| --- | --- |
| Minecraft | 1.21.1 |
| MOD ローダー | NeoForge `21.1.235` |
| ビルドツール | Gradle 9.2.1 (wrapper) + ModDevGradle `2.0.141` (`net.neoforged.moddev`) |
| 必須 JDK | **Java 21**（Mojang が 1.21.1 で配布する版に合わせる） |
| Mappings | Parchment `2024.11.17` |
| mod id | `techlab`（`gradle.properties` の `mod_id`） |
| ベースパッケージ | `com.example.techlab` |

バージョン座標はすべて `gradle.properties` に集約されている。ここを直接編集して更新する（`build.gradle` にハードコードしない）。

## 環境

- JDK 21: `C:\Program Files\Microsoft\jdk-21.0.11.10-hotspot`（Microsoft OpenJDK、winget 導入）。マシン環境変数 `JAVA_HOME` に設定済み。
- 新しいシェルで `JAVA_HOME` が空の場合: `$env:JAVA_HOME = [System.Environment]::GetEnvironmentVariable('JAVA_HOME','Machine')`
- Gradle は wrapper 同梱のため別途インストール不要。必ず `.\gradlew.bat`（PowerShell）/ `./gradlew`（bash）を使う。

## よく使うコマンド

PowerShell から実行する場合、まず `JAVA_HOME` を通してからカレントをプロジェクト直下にする。

```powershell
$env:JAVA_HOME = [System.Environment]::GetEnvironmentVariable('JAVA_HOME','Machine')
.\gradlew.bat <task>
```

| タスク | 用途 |
| --- | --- |
| `.\gradlew.bat build` | コンパイル＋jar生成（`build/libs/techlab-<ver>.jar`）。CI相当のフルビルド |
| `.\gradlew.bat runClient` | 開発用 Minecraft クライアントを起動（MOD 適用済み）。動作確認の主力 |
| `.\gradlew.bat runServer` | 開発用専用サーバを起動 |
| `.\gradlew.bat runData` | **データ生成 (datagen)**。`src/generated/resources` にモデル/レシピ等を出力 |
| `.\gradlew.bat runGameTestServer` | 登録済み GameTest を実行して終了（自動テスト用） |
| `.\gradlew.bat --refresh-dependencies build` | 依存を強制再取得（バージョン変更後に不整合が出たら） |
| `.\gradlew.bat clean` | `build/` を削除 |

- 初回の `build` / `runClient` は NeoForge・Minecraft のダウンロードと復号のため数分かかる（以降はキャッシュされ高速）。
- 実行時の作業ディレクトリは `run/`（gitignore 済み）。ワールドデータやログはここに出る。ログは `run/logs/latest.log`。
- 単体の GameTest だけ回す仕組みは `runGameTestServer`。JUnit 的な純ロジックテストは現状なし（`src/test` は未使用）。

## デバッグ

ModDevGradle は IntelliJ / Eclipse 用の run 構成（`runClient` 等）を Gradle sync 時に自動生成する。

- **IntelliJ IDEA**: プロジェクトを Gradle として import → 右上の run 構成に `runClient` が出る → **Debug 実行**でブレークポイントが効く。ホットスワップ（メソッド本体の変更）も可能。
- **CLI からアタッチ**: `.\gradlew.bat runClient` はそのままだと通常起動。IDE を使わずリモートデバッグしたい場合は JVM 引数 `-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005` を run に付与し、任意のデバッガを 5005 にアタッチする。
- ログレベルは `build.gradle` の `runs { configureEach { logLevel = DEBUG } }` で DEBUG。レジストリの発火は `forge.logging.markers=REGISTRIES` で追える。

## アーキテクチャ

### 登録の流れ（最重要）

Minecraft の要素（ブロック/アイテム/タブ等）は**レジストリイベント発火時**に登録する必要がある。本 MOD は NeoForge の `DeferredRegister` パターンで「宣言」と「登録」を分離している。

1. 各 `Mod*` クラスが `static final DeferredRegister` を持ち、`static final Deferred{Block,Item}` フィールドとして要素を**宣言**する（この時点では未登録）。
2. 各クラスの `register(IEventBus)` を **`TechLab` コンストラクタ**が呼び、DeferredRegister を MOD イベントバスに繋ぐ。
3. 実際の登録は後で NeoForge がイベントを発火したときに行われる。

```
TechLab (@Mod, エントリポイント)
  └─ コンストラクタで以下を modEventBus に登録:
       ModItems.register()            → item/ModItems.java        (ITEMS)
       ModBlocks.register()           → block/ModBlocks.java      (BLOCKS + 各ブロックのBlockItem)
       ModBlockEntities.register()    → block/entity/ModBlockEntities.java (BlockEntityType)
       ModMultiblock.register()       → multiblock/ModMultiblock.java (マルチブロック assembler、隔離登録＋Capability委譲)
       ModCreativeModeTabs.register() → item/ModCreativeModeTabs.java (独自クリエイティブタブ)
     + addListener(ModCapabilities::registerCapabilities) → ModCapabilities.java (FE等のCapability公開)
```

BlockEntityを持つ機械の実例が `energy_machine`（FE受電で発光）。BlockEntity(`block/entity/`)・エネルギー(`energy/`)・Capability公開(`ModCapabilities`)の連携パターンは `DESIGN.md` の「実装例」を参照。

**加工マシンの中核パターン**（在庫/エネルギー/カスタム加工レシピ/GUI(Menu+Screen)/クライアント同期/流体/datagen/AE2ストレージ）は [docs/machine-patterns.md](docs/machine-patterns.md) に実APIシグネチャ付きでまとめてある。マシンを作る前に読む。

- **ブロックと BlockItem の関係**: `ModBlocks.registerBlock()` がブロック登録と同時に `ModItems.ITEMS` へ BlockItem を登録する。よって `ModItems` は `ModBlocks` を参照してはならない（静的初期化の循環を避けるため、依存は Blocks→Items の一方向）。
- 新要素を足すときは、対応する `Mod*` クラスに `Deferred*` フィールドを 1 行足すだけ。`register()` の呼び出しは既に TechLab にあるので追加不要。

### クライアント / 共通（サーバ）の分離

- `TechLab`（共通）: 両サイドでロードされる。レンダリング等クライアント専用 API を**参照してはいけない**。
- `TechLabClient`（`@Mod(dist = Dist.CLIENT)`）: 専用サーバでは読み込まれない。クライアント専用処理はここに置く。
- この分離を破ると専用サーバ起動時に `NoClassDefFoundError` 系でクラッシュする。

### リソースの構成と 1.21 特有の落とし穴

- `src/main/resources/assets/<modid>/` … クライアント資産（`blockstates`, `models`, `textures`, `lang`）。
- `src/main/resources/data/<ns>/` … サーバ資産（`loot_table`, `recipe`, `tags`, `advancement` 等）。
- **MC 1.21 でデータパックのフォルダ名が単数形化された**。旧版の記事をそのまま使うと読み込まれない:
  - `recipes/` → **`recipe/`**、`loot_tables/` → **`loot_table/`**、`advancements/` → **`advancement/`**
  - `tags/blocks/` → **`tags/block/`**、`tags/items/` → **`tags/item/`**（`entity_type`, `fluid` 等も同様に単数）
  - ただしブロックの loot table のパスは慣習上 `loot_table/blocks/<name>.json`（この "blocks" はフォルダ改名対象外）。
- `src/main/templates/META-INF/neoforge.mods.toml` はテンプレート。`${mod_id}` 等のプレースホルダは `build.gradle` の `generateModMetadata` タスクが `gradle.properties` の値で置換して生成する。**このファイルを直接いじるときはプレースホルダを壊さないこと**。翻訳キーではなくメタデータ。
- ブロックが `requiresCorrectToolForDrops()` を持つ場合、`minecraft:mineable/<tool>` タグ **と** ティアタグ（例 `needs_iron_tool`）の両方に入れないと「適正ツールでもドロップしない」状態になる。

### データ生成 (datagen) — 導入済み

コアのモデル/ブロックステート/レシピ/ドロップ表/タグ/言語 JSON は **datagen で自動生成**する（`com.example.techlab.datagen`、エントリ `DataGenerators`）。

- **要素を足したら `.\gradlew.bat runData` を再実行** → `src/generated/resources` に出力され、`build.gradle` の sourceSet 設定でビルドに含まれる。
- datagenが吐くファイルは**手書きしない**（同一パスが `src/main/resources` と `src/generated/resources` に両方あるとビルドで重複衝突する）。
- **テクスチャPNGだけは手動**（`assets/techlab/textures/`）。
- 例外: `compat/ae2` の `me_connector` は datagen をAE2に依存させないため blockstate/model/loot を手書き維持（タグ・言語は datagen 側で optional 扱い）。詳細は `DESIGN.md`「データ生成」。

## 工業系連携・テスト環境

### テスト用MODの入れ方（開発ランタイム）

`build.gradle` の `repositories` に BlameJared(JEI) / CurseMaven / Modrinth を追加済み。MODは**スコープ**で使い分ける:

| 用途 | 設定 | 例 |
| --- | --- | --- |
| 相手MODのAPIに対してコードを書く | `compileOnly` | JEIプラグイン、Mekanism連携 |
| 開発中だけゲームに入れて動作確認（配布物に含めない） | `localRuntime` | JEI本体、テスト用の工業MOD |
| 実行時に必要だがコンパイルはしない（配布依存にする） | `runtimeOnly` | ハード依存する前提のMOD |

- **JEI は導入済み**: `localRuntime "mezz.jei:jei-1.21.1-neoforge:${jei_version}"` + API を `compileOnly`。`runClient` するとアイテム/レシピ確認・チートが使える。
- **工業系テストMODも導入済み**（`localRuntime` で投入、API連携用に `compileOnly` も併用）。バージョンは `gradle.properties`:
  - Mekanism (`mekanism`) / Immersive Engineering (`immersiveengineering`) / Applied Energistics 2 (`ae2`)
  - **Mekanism サブモジュール**（すべて別mod・基盤と同期`${mekanism_version}`・`localRuntime`のみ）: Generators (`mekanism-generators`, 発電機) / Tools (`mekanism-tools`, 合金ツール防具) / Additions (`mekanism-additions`, 追加要素)
  - GuideME (`guideme`) … AE2 の必須依存のため `localRuntime` のみ
  - 補足: FE電源のテストは Generators 無しでも IE の Creative Capacitor / Mekanism の Creative Energy Cube で可能
  - 実IDは Modrinth API で 1.21.1/neoforge 向け最新を確認して固定（更新時も同手順）。
- MODの座標:
  - CurseForge: `localRuntime "curse.maven:<slug>-<projectId>:<fileId>"`（`content { includeGroup 'curse.maven' }` で絞り込み済み）
  - Modrinth: `localRuntime "maven.modrinth:<slug>:<versionId>"`
- ModDevGradle は runtime クラスパス上のNeoForge MODを**自動ロード**する。特別な設定は不要。
- バージョンは `gradle.properties` に集約（`jei_version` 等）。工業MODの実IDは導入時に確定させる。

### 鉱石辞書 = 共通タグ (`c:`)

MC 1.21 に旧来の Ore Dictionary は無い。**NeoForge が標準で持つ共通タグ規約 `c:` がその代替**。他MODが追加する素材も同じ `c:` タグに入るため、タグ経由で横断的に扱える。

- 代表例: `c:ingots/<name>`, `c:ores/<name>`, `c:raw_materials/<name>`, `c:dusts/<name>`, `c:gems/<name>`, `c:storage_blocks/<name>`、液体は `c:` の fluid タグ。umbrella タグ（`c:ingots` 等）＋サブ（`c:ingots/copper`）の二層構造。
- コードからは NeoForge の `Tags.Items` / `Tags.Blocks` / `Tags.Fluids` 定数、または `TagKey` を自作して参照する。
- **本MODの素材はすでに共通タグへ登録済み**: `techlab:sapphire` → `c:gems`(+`c:gems/sapphire`)、`techlab:sapphire_block` → `c:storage_blocks`(+`/sapphire`)。これで他MODやレシピからタグで拾える。
- レシピで他MODの素材を受ける時は `{"tag": "c:ingots/iron"}` のようにタグを材料にする（特定MODに依存しない）。

### 連携で最重要: NeoForge の統一 Capability

エネルギー・液体・アイテム搬送は**NeoForge が標準化**しており、多くの場合**相手MODのAPIに依存せず連携できる**。

- **エネルギー (FE/RF)**: `IEnergyStorage` capability。これを実装/消費すれば Mekanism・IE・Thermal 等とエネルギー授受できる。
- **液体**: `IFluidHandler` + `FluidStack`。
- **アイテム搬送**: `IItemHandler`。
- これらを `RegisterCapabilitiesEvent` で BlockEntity に公開すれば、パイプ/ケーブル系MODと自動で繋がる。特定MODの `compileOnly` が要るのは、その独自システム（例: Mekanism の chemical/gas）を直接触るときだけ。

### 特定MOD連携の隔離（optional-compat パターン）

AE2 のように標準Capabilityで表せない独自API（`appeng.api.*`）を使う連携は、**AE2が無い環境で本体が壊れないよう隔離する**:

1. 連携コードを `compat/<modid>/` パッケージに閉じ込め、本体(core)から直接参照しない。
2. `TechLab` コンストラクタで `ModList.get().isLoaded("<modid>")` を確認してから `<Modid>Compat.register()` を呼ぶ（そこで初めて連携クラスがロードされる）。ガードに使う mod id は `static final String` 定数にする（インライン化され、条件評価でクラスロードを誘発しない）。
3. `neoforge.mods.toml` に `type="optional"` 依存を明示。
4. 登録は本体とは別の DeferredRegister に隔離しておくと、将来まるごと別mod（アドオン）へ切り出しやすい。

実装例が `compat/ae2`（`me_connector`＝MEネットワークに接続して発光するサンプル）。詳細は `DESIGN.md`「AE2連携」。

## mod id / パッケージのリネーム

`techlab` を独自名に変える場合は次を**すべて**揃える必要がある（1 つでもズレると登録失敗やテクスチャ欠けになる）:

1. `gradle.properties`: `mod_id`, `mod_name`, `mod_group_id`
2. Java パッケージ `com/example/techlab/` のディレクトリ名とパッケージ宣言、`TechLab.MOD_ID` 定数
3. `src/main/resources/assets/<modid>/` と `data/<modid>/` のフォルダ名
4. `lang/*.json` の翻訳キー（`item.<modid>.*`, `block.<modid>.*`, `itemGroup.<modid>`）と、JSON 内の `techlab:` 名前空間参照

`neoforge.mods.toml` はテンプレート置換なので基本は自動追従する。
