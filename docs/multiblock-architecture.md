# マルチブロック構造 アーキテクチャ (NeoForge 1.21.1)

「複数ブロックを組んで1つの大型装置にする」仕組みの設計知見。実例は **vanilla `BlockPattern`**（構造検出）と **Immersive Engineering の multiblock フレームワーク**（実jarで確認、2026-07-09）を参照。

> **大前提: Minecraft/NeoForge に“標準のマルチブロック枠組み”は無い。** 自作するか、MODの枠組み（IE / GregTech / Modular Machinery 等）に乗る。ただし vanilla は**形の検出**を助ける `BlockPattern` を持つ（ネザーポータル・ビーコン・ゴーレム召喚で使用）。

---

## 1. 基本アーキテクチャ: コントローラ / パーツ方式

ほぼ全てのマルチブロックはこの構造。

```
        ┌─ Controller (master) ブロック
        │    └─ BlockEntity に「頭脳」を集約: 在庫/エネルギー/加工ロジック/GUI/tick
        │
構造 ───┤─ Part (dummy/slave) ブロック × 複数
        │    └─ 自分では何もしない。コントローラの BlockPos を保持し、全要求を委譲
        │
        └─ 状態: formed(bool)。パーツ=controllerPos、コントローラ=向き＋メンバー範囲
```

- **単一の真実 = コントローラ**。N個のBEがそれぞれtick/加工するのを避け、コントローラ1つが処理する。
- パーツは「マーカー＋窓口」。外から来た搬入出やGUI要求をコントローラへ転送する。
- 加工マシンの中身（在庫・エネルギー・レシピ）は [machine-patterns.md](machine-patterns.md) と同じものをコントローラに載せるだけ。マルチブロックは「それを複数ブロックで包む」レイヤー。

---

## 2. 形成 (formation) の流れ

### いつ検証するか
- ブロック設置時 / 近傍変化 `neighborChanged` / プレイヤーの右クリック（組み立てトリガー）。
- 毎回フル検証は重い → **dirtyフラグ or デバウンス**（同tick内で1回等）。

### どう検証するか
**(a) vanilla `BlockPattern`（回転も自動で試せる・おすすめの検出手段）**
```java
BlockPattern pattern = BlockPatternBuilder.start()
    .aisle("CCC", "CAC", "CCC")   // 各文字が1ブロック、行が縦、文字列が横、aisleが奥行き
    .aisle("CCC", "C C", "CCC")   // ' '（空白）は「何でも良い/空気」
    .where('C', bw -> bw.getState().is(ModBlocks.CASING.get()))
    .where('A', bw -> bw.getState().is(ModBlocks.CONTROLLER.get()))
    .build();

BlockPattern.BlockPatternMatch match = pattern.find(level, pos); // 見つかれば向き・原点入り
```
`BlockInWorld` の述語で各セルの条件を書く。`find` は近傍を走査して一致（向き込み）を返す。

**(b) 手動チェック**: コントローラから相対座標をループし各ブロックを確認（完全制御・実装は重い）。

### 成立/崩壊
- **成立**: 各ブロックを `formed=true` に。パーツBEへ controllerPos を書く。IE方式は設置時に**専用ダミーブロックへ置換**（`MultiblockPartBlock`）＋一括設置アイテム（`MultiblockItem`）。
- **崩壊**: どれか破壊/近傍変化で再検証 → 不成立なら `formed=false`、パーツを通常ブロックへ戻す。`onRemove`/`neighborChanged` で漏れなく検出する（ゴースト状態を残さない）。

---

## 3. 向き (orientation)

- 4方向に向けられるので、パターン検証は各回転で試す（`BlockPattern.find` は回転を試行）。
- **相対→ワールド座標変換**が要る。「マルチブロックのローカル座標系」を持ち、向きに応じて回す。IEは `MultiblockOrientation` / `RelativeBlockFace` / `MultiblockFace` でこれを体系化。
- どの面が「正面/入力面」かも向きで変わる点に注意。

---

## 4. Capability の委譲（最重要）

外部のパイプ/ケーブルは**マルチブロックのどの面からでも**繋ぎたい。→ **パーツの各面が Capability を公開し、コントローラへ委譲**する。

```java
// RegisterCapabilitiesEvent: パーツBEにも item/fluid/energy capability を登録
event.registerBlockEntity(Capabilities.ItemHandler.BLOCK, PART_BE.get(), (partBe, side) -> {
    var master = partBe.getController();          // controllerPos から master BE を取得
    if (master == null || !master.isFormed()) return null;
    return master.getItemHandlerFor(partBe.getBlockPos(), side); // 面/位置ごとに入出力を出し分け
});
```
- **面/位置ごとの役割分担**: ある面はアイテム入力、別の面はエネルギー入力…とできる。IEは `CapabilityPosition`（相対位置＋面→capability）でこれを表現。
- 未形成時は `null` を返す（外部が「繋がっていない」と判断できる）。

---

## 5. 永続化・チャンクロード

- **パーツ**: controllerPos を NBT 保存。**コントローラ**: formed・向き・メンバー範囲を保存。
- **チャンク境界問題**: 構造が境界をまたぐと片側だけロードされ得る。定石は「コントローラが全メンバーを管理し、必要ブロックが揃うまで休止 → 揃ったら再開」。堅牢にするなら再ロード時に再検証。
- BE参照は `level.getBlockEntity(controllerPos)` で都度引く（直接参照を持ち続けるとアンロードでリーク/古参照になる）。

---

## 6. レンダリング

- 形成時にパーツのモデルを **非表示 or 別モデル**へ（blockstate `formed`）。
- 全体を1つの見た目にするなら、コントローラに `BlockEntityRenderer` を付けて構造全体を描く（IE `MultiblockRenderer`）。または各ブロックが自分の部分モデルを持つ。
- 任意でゴーストプレビュー（設置前に半透明で完成形を表示）。

---

## 7. 実装アプローチの選択

| 方法 | 向き | コスト |
| --- | --- | --- |
| **vanilla `BlockPattern` + 自作コントローラ/パーツ** | 小〜中規模の独自マルチブロック（推奨の入口） | 中：検出は楽、状態管理は自前 |
| 完全自作（相対座標ループ） | 学習・完全制御 | 高 |
| **IE multiblock API** に乗る | IEエコシステム前提の大型機械 | 中：枠組みは強力だが学習コスト。compat隔離必須 |
| Modular Machinery 等の専用ライブラリ | データ駆動で大量の機械を量産 | 低〜中：ライブラリ依存 |

このプロジェクトの方針（[[structure-modular-not-monolith]] 準拠）なら、まず **vanilla `BlockPattern` + 自作コントローラ/パーツ** を `multiblock/`（または機能別パッケージ）に隔離して試すのが素直。

---

## 8. 参考: IE フレームワークの構造（実クラス）

`blusunrize.immersiveengineering.api.multiblocks.*`（本体jar同梱）:
- **registry**: `MultiblockBlockEntityMaster`（コントローラBE）/ `MultiblockBlockEntityDummy`（パーツBE）/ `MultiblockPartBlock` / `MultiblockItem`（一括設置アイテム）。
- **logic**: `IMultiblockLogic` / `IMultiblockState` / `IMultiblockBE` … **機械ロジックをブロック配線から分離**（テスト・再利用しやすい）。
- **env**: `IMultiblockContext` / `IMultiblockBEHelperMaster|Dummy` / `IMultiblockLevel`（構造を1つの仮想Levelとして扱うビュー）。
- **util**: `CapabilityPosition`・`MultiblockOrientation`・`RelativeBlockFace`・`MultiblockFace`・`MultiblockRenderer`。
- **shape**: `TemplateMultiblock`（構造テンプレート .nbt で形を定義）＋ `BlockMatcher`。

→ 「master/dummy＋logic分離＋相対座標＋CapabilityPosition＋テンプレート形状」という、本ドキュメントの設計をそのまま体系化したもの。自作の設計指針として非常に参考になる。

---

## 実装済みサンプル: `assembler`（multiblock パッケージ）

本ドキュメントの設計を最小構成で実装したもの（`com.example.techlab.multiblock`）。

- 形: **3×3 の平面**（中心=`assembler_controller`、周囲8マス=`assembler_casing`、同じ高さ）。
- 操作: コントローラを**右クリック**で形成/解除トグル（アクションバーに結果表示）。
- 形成すると `FORMED`（発光）になり、コントローラ/ケーシングが光る。どれか壊すと自動で解除。
- **Capability委譲**: コントローラはFEバッファを公開、ケーシングは各面のFE要求をコントローラへ委譲（`ModMultiblock` の `RegisterCapabilitiesEvent`）。→ 構造のどの面にケーブルを繋いでも受電できる。
- 構成: `AssemblerControllerBlock(Entity)` / `AssemblerCasingBlock(Entity)` / `ModMultiblock`（登録＋Capability）。検出は相対座標の手動チェック（`MEMBER_OFFSETS`）、回転が要る大型構造では §2 の `BlockPattern` に置き換える。
- リソースはすべて datagen 生成（`FORMED` は光量のみ変えるので `simpleBlockWithItem` の空variantで1モデル）。

**実機テスト手順**（帰宅後）:
1. クリエイティブで「アセンブラ・ケーシング」8個を 3×3 の枠（中央を空ける）に置く。
2. 中央に「アセンブラ・コントローラ」を置く。
3. コントローラを**右クリック** → 「アセンブラを形成しました！」と表示され、9ブロックが発光。
4. 隣接するケーシングに Mekanism/IE の発電機＋ケーブルを繋ぐ → コントローラのFEバッファに受電（委譲）。
5. どれか1つを壊す → 自動で解除・消灯。もう一度コントローラ右クリックで再形成。

（コンパイル・全MODロード・ビルド・データ生成まで検証済み。ゲーム内の見た目/操作は要確認。）

## ハマりどころ
- **チャンク境界での分割ロード**（片側だけロード）→ 揃うまで休止する設計に。
- **向きの座標変換ミス**（回転の当て方）→ 相対座標系を1箇所に集約。
- **Capability委譲の null 処理**（未形成/コントローラ未ロード時）。
- **崩壊検出漏れ**でゴースト状態が残る → `onRemove`/`neighborChanged` を必ず塞ぐ。
- **再検証が重い** → dirtyフラグ/デバウンスで毎tickフル検証を避ける。
- BE直接参照を持ち続けない（`getBlockEntity(controllerPos)` で都度取得）。
