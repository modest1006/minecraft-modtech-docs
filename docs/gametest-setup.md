# GameTest による機能自動テスト (NeoForge 1.21.1)

「プレイヤーが触らずに MOD の機能を検証する」自動テストの導入・書き方・落とし穴。

Phase 1（試作 4本）は 2026-07-11 に導入済み、いずれも `runGameTestServer` で **1.5 秒程度** で全 PASS することを確認済み。

- `techlab:catapult_launches_east` (`CatapultGameTest`)
- `techlab:belt_pushes_item_east` (`BeltConveyorGameTest`)
- `techlab:assembler_forms` (`AssemblerGameTest`)
- `techlab:assembler_pipe_routing` (`AssemblerGameTest`)

---

## 何ができて、何が無理か

**できる**:
- Level を伴うロジック検証（ブロック配置、entityInside、tickの進行）
- Capability の実挙動（`level.getCapability(...)` で本物のハンドラを引く）
- BlockEntity のロジック（`tryForm()` 等の直接呼び出し）
- 加工進行、エネルギー授受、アイテム輸送
- 数秒で完結するので CI (`.github/workflows/build.yml`) にそのまま乗せられる

**無理**:
- レンダリング/BER/GUI/テクスチャの視覚検証（そもそもクライアントが立ち上がらない）
- 音の再生確認
- プレイヤー入力（右クリック等）そのもの — ただし内部メソッドを直接呼ぶことで代替できる

視覚バグは `runClient` で目視、それ以外はほぼ GameTest で拾える。

## セットアップ（本プロジェクトで完了済み）

`build.gradle` にはあらかじめ `runGameTestServer` タスクが定義されている:

```gradle
runs {
    gameTestServer {
        type = "gameTestServer"
        systemProperty 'neoforge.enabledGameTestNamespaces', project.mod_id
    }
}
```

加えて、SNBT テンプレを `run/gameteststructures/` にコピーする補助タスクを追加:

```gradle
tasks.register('copyGameTestStructures', Copy) {
    from 'src/main/gametest/structures'
    into 'run/gameteststructures'
    include '**/*.snbt'
}
tasks.named('runGameTestServer').configure {
    dependsOn 'copyGameTestStructures'
}
```

**なぜ必要か**: vanilla の `StructureTemplateManager` は
- 通常: `data/<ns>/structure/<path>.nbt`（バイナリ NBT）
- **dev 環境のみ** (`SharedConstants.IS_RUNNING_IN_IDE=true`): `run/gameteststructures/<id.path>.snbt`（テキスト SNBT）

の 2 経路から読む。**SNBT ローダーは dev 限定**、しかも `id.getPath()` だけを見るので namespace 混入 (`techlab:xxx`) は使えない。バージョン管理は `src/main/gametest/` に置いて、テスト実行前にコピーして使う。

## 実行

```powershell
$env:JAVA_HOME = [System.Environment]::GetEnvironmentVariable('JAVA_HOME','Machine')
.\gradlew.bat runGameTestServer
```

ログ末尾に `All N required tests passed :)` が出れば成功。失敗すると各テストの詳細（アサーション文言＋落ちた位置）が表示される。

## テストクラスの書き方

```java
@GameTestHolder(TechLab.MOD_ID)     // namespace を設定
@PrefixGameTestTemplate(false)      // クラス名 prefix を無効化（テンプレ共有のため）
public class SomeGameTest {

    @GameTest(template = "empty_3x3x3", timeoutTicks = 60)
    public static void someScenario(GameTestHelper helper) {
        // 1. ブロックを配置（相対座標）
        helper.setBlock(new BlockPos(1, 1, 1), ModBlocks.FOO.get().defaultBlockState());

        // 2. エンティティ生成 or Capability クエリ
        BlockPos absPos = helper.absolutePos(new BlockPos(1, 1, 1));
        IItemHandler h = helper.getLevel().getCapability(
                Capabilities.ItemHandler.BLOCK, absPos, Direction.UP);

        // 3. アサーション
        helper.assertTrue(h != null, "capability missing");

        // 4. 成功宣言（即時 or 遅延）
        helper.succeed();
        //  または:
        //  helper.succeedWhen(() -> helper.assertTrue(condition, "..."));  // 毎tick評価
    }
}
```

### 命名規則で解決される名前

| annotation | 効果 |
|---|---|
| `@GameTestHolder("techlab")` | 全メソッドの test id / template id を `techlab:...` にする |
| `@PrefixGameTestTemplate(true)` （デフォルト） | クラス名(小文字)を dot-prefix する。例: `template="foo"` → `techlab:someclassname.foo` |
| `@PrefixGameTestTemplate(false)` | prefix しない。`techlab:foo` に解決 |
| `@GameTest(templateNamespace = "xxx")` | ホルダーの namespace を上書き |

**本プロジェクトは全テストクラスで `@PrefixGameTestTemplate(false)`** にし、`empty_3x3x3.snbt` 1本を共有している。

## テンプレ SNBT の書き方

```snbt
{
    DataVersion: 3953,               # MC 1.21.1 の DataVersion
    size: [3, 3, 3],                 # x, y, z
    palette: [
        {Name: "minecraft:air"},
        {Name: "minecraft:polished_andesite"}
    ],
    blocks: [
        {pos: [0, 0, 0], state: 1},  # state はパレット index
        {pos: [1, 0, 0], state: 1},
        // 3x3 の床
        ...
    ],
    entities: []
}
```

- `size` は [x, y, z]。テスト時にこの大きさの矩形領域が確保される
- `palette[0]` は air 相当。実際に置くブロックは `palette[1]` 以降
- `blocks[i].state` はパレット index（**BlockState 文字列ではない**）
- 相対座標: (0,0,0) が template の南西下角、(size-1) が北東上角

## `GameTestHelper` の主要 API

| メソッド | 用途 |
|---|---|
| `setBlock(BlockPos, BlockState)` | 相対座標にブロック設置 |
| `spawnItem(Item, float x, float y, float z)` | 相対座標に ItemEntity を生成（返り値で参照を保持） |
| `absolutePos(BlockPos)` / `absoluteVec(Vec3)` | 相対 → 絶対座標変換（Level API に渡す時） |
| `getLevel()` | 実 ServerLevel。`getCapability`, `getBlockEntity` 等はここから |
| `assertTrue(cond, msg)` | 失敗時に msg を含めて例外化。テストが失敗扱いになる |
| `succeed()` | 即成功宣言 |
| `succeedWhen(Runnable check)` | check が assert を throw しなくなったら成功。毎tick評価、timeoutTicks 超過で失敗 |
| `fail(String)` / `fail(msg, BlockPos)` | 明示的な失敗 |

## パターン集

### Capability 経由の実挙動テスト
```java
BlockPos abs = helper.absolutePos(new BlockPos(1, 1, 1));
IItemHandler h = helper.getLevel().getCapability(
        Capabilities.ItemHandler.BLOCK, abs, Direction.UP);
helper.assertTrue(h != null, "no capability");
ItemStack rem = h.insertItem(0, new ItemStack(Items.APPLE), false);
helper.assertTrue(rem.isEmpty(), "insert rejected: " + rem);
```

### BlockEntity 内部メソッドを直接叩く
```java
BlockPos abs = helper.absolutePos(new BlockPos(1, 1, 1));
var be = (MyBlockEntity) helper.getLevel().getBlockEntity(abs);
helper.assertTrue(be != null && be.tryForm(), "form failed");
```

### エンティティの時系列変化（差分ベース）
```java
ItemEntity it = helper.spawnItem(Items.APPLE, 0.3f, 1.5f, 1.5f);
it.setDeltaMovement(0, 0, 0);
double startX = it.getX();                // 開始位置をキャプチャ
helper.succeedWhen(() -> {
    double dx = it.getX() - startX;
    helper.assertTrue(dx > 1.5, "did not move: dx=" + dx);
});
```

## ハマりどころ

1. **SNBT が読まれない** — dev 環境（`IS_RUNNING_IN_IDE=true`）でしか SNBT フォールバックローダーは有効にならない。本番jarでは binary NBT のみ。runGameTestServer は dev なので OK。
2. **テンプレ id path の間違い** — `@PrefixGameTestTemplate(false)` を付けないと `<classname>.<template>` になる。SNBT ファイル名もそれに合わせないと `RuntimeException: Failed to load structure info` が出る。
3. **`template = "techlab:xxx"` は使えない** — テンプレ値に `:` を入れると id が `techlab:...techlab:xxx` に化けて `ResourceLocationException`（"Non [a-z0-9/._-] character in path"）。namespace は `@GameTestHolder` で指定するのが正解。
4. **`import net.minecraft.gametest.framework.PrefixGameTestTemplate`** はコンパイルエラー — 正しくは **`net.neoforged.neoforge.gametest.PrefixGameTestTemplate`**。vanilla に無く NeoForge 固有。
5. **`helper.absolutePos(BlockPos.ZERO)` で origin を取れるが、テスト間で origin が違う** — テスト内で startX 等を capture して差分で判定する方が安全（絶対座標に依存しない）。
6. **エンティティを spawn した位置と VoxelShape の関係** — ItemEntity の AABB がブロックの VoxelShape と重なった tick で `entityInside` が発火する。薄板（例: ベルコン）の場合、spawn 高さを少し上げて重力で「落として着地させる」パターンだと安定する。
7. **`succeedWhen` の中で最終アサーションだけ書くと通り抜ける** — succeedWhen は「throw しなくなったら成功」。条件が満たされない限り毎tick評価→timeoutで失敗。逆に「条件を満たしたら成功」を書くには assertTrue 系を使う。
8. **クラス名は必ず小文字化される** — `MyGameTest` → `mygametest`。ドット区切りで template と結合されるので、SNBT ファイル名側も小文字で合わせる。

## Phase 2 以降

| Phase | やること |
|---|---|
| **2** | GitHub Actions `.github/workflows/build.yml` に `runGameTestServer` を追加、PR 毎に自動実行 |
| 3 | 「加工完了までフル進行」テスト（アセンブラに材料+電力→100tick→出力に完成品） |
| 4 | エラー再現用のテスト（過去バグの回帰防止テスト） |
| 応用 | GameTest レポート出力 (`--report` オプション) を CI アーティファクトとして残す |

## 参考

- vanilla `net.minecraft.gametest.framework.GameTestHelper`（豊富なアサート API）
- NeoForge `net.neoforged.neoforge.gametest.GameTestHooks`（prefix / namespace 制御ロジック）
- 上級パターン: `@GameTestGenerator` で 1 メソッドから複数テストを動的生成、`@BeforeBatch` / `@AfterBatch` でセットアップ・後処理
