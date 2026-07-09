# 動くブロック（エレベーター/自動ドア等）と当たり判定 — 各アプローチ

「動きがあって当たり判定もあるブロック」を、開発者がどう実現しているかの調査（NeoForge 1.21.1）。API名は実jar（vanilla merged）を `javap` で確認済み。実例は Create / Moving Elevators / OpenBlocks Elevator / vanilla piston。

> **根本問題**: Minecraft のブロックは**静的グリッド**。「連続的に動きながら当たり判定を持つブロック」の標準プリミティブは**ピストンだけ**。だから開発者は「①見た目の動き」と「②当たり判定」を**別々に用意して組み合わせる**。この組み方で流派が分かれる。

---

## 4つのアプローチ（俯瞰）

| 流派 | 見た目の動き | 当たり判定 | 代表例 | 難易度 |
| --- | --- | --- | --- | --- |
| **A. 動かさない/瞬間** | なし（テレポート・瞬間切替） | 常に静的ブロック（自明） | OpenBlocks Elevator, vanillaドア | 低 |
| **B. VoxelShape段階アニメ** | BERで補間 | blockstateごとの `getCollisionShape`（段階的だが本物） | スライドドア系 | 中 |
| **C. ピストン式 移動ブロック** | `PistonMovingBlockEntity` のoffset補間 | `MovingPistonBlock.getCollisionShape`（progressで動く） | vanillaピストン | 中〜高 |
| **D. エンティティ化コントラプション** | エンティティのレンダリング | エンティティの当たり判定（`canBeCollidedWith`）＋乗員/上載せ処理 | Create contraption, Moving Elevators | 高 |

---

## A. 動かさない（テレポート/瞬間切替）

**動く当たり判定を持たない**ことで問題自体を回避する。最も堅牢・軽量。

- **エレベーター（OpenBlocks Elevator方式）**: 各階の**同じ水平座標**にエレベーターブロックを置き、ジャンプ／スニークで**プレイヤーを上下の階へテレポート**。移動アニメも移動衝突も無い。
  - 長所: 実装容易・軽量・確実。短所: 「乗って動く」感が無い、他エンティティ/アイテムを一緒に運べない。
- **自動ドア（vanillaドア方式）**: `open`/`closed` blockstate を瞬間切替。当たり判定は blockstate ごとの `getCollisionShape`（開=`Shapes.empty()`、閉=板）。レッドストーン/センサー/近接検知で自動化。
  - 長所: 確実。短所: スライドの「動き」は無い。

→ **「ただ階を移動」「開け閉め」なら第一候補。**

---

## B. VoxelShape段階アニメ（ブロックのまま定位置で変形）

ブロックは動かさず、**blockstateに進捗プロパティ**（例 `PROGRESS 0..N`）を持たせ、tickごとに進める。当たり判定は進捗に応じて変える。

- **当たり判定**: `getCollisionShape(state, level, pos, ctx)` を進捗で切り替える。`Shapes.box(x1,y1,z1, x2,y2,z2)` で板を動かす/縮める。→ 段階的だが**本物の衝突**（プレイヤーが挟まれる・乗れる）。
- **見た目**: 段階の間を滑らかに見せたいなら `BlockEntity` ＋ `BlockEntityRenderer` で補間描画（衝突は段階、見た目は連続）。
- **用途**: スライド自動ドア、せり上がる床、絞り機構など「1〜数ブロックが定位置内で変形」する動き。
- 長所: ブロックのままなので周囲との整合が楽。短所: 「大きく移動」には不向き（中間位置ぶんの blockstate/衝突を用意するのは非現実的）。

---

## C. ピストン式 移動ブロック（vanilla唯一の“動く当たり判定ブロック”）

vanilla は `MovingPistonBlock`（一時ブロック）＋ `PistonMovingBlockEntity` で「滑らかに動いて人を押す当たり判定ブロック」を実現している。**自作の参考になる唯一の標準実装**。

- `PistonMovingBlockEntity.getProgress(partialTick)` / `getYOff(partial)` 等 … 補間オフセット（描画用）。
- `MovingPistonBlock.getCollisionShape(...)` … progress に応じて**動く衝突形状**を返す。乗ったプレイヤーはこの形状に押される。
- 仕組みは複雑（`sourceState`/`movedState`、`TICK_MOVEMENT`、移動先エンティティの押し出し処理）。
- **用途**: 短距離の押し出し・せり上がり。1マス移動が基本で、長距離エレベーターには不向き。

---

## D. エンティティ化コントラプション（大型・長距離・任意形状の本命）

動く構造を**エンティティ**として表現する。Create の Contraption や Moving Elevators のやり方。**最も強力だが最も重い**。

- **見た目**: エンティティが構造のブロック群を描画（小さければ `Display.BlockDisplay`＝blockstateを描く軽量表示エンティティ、大型は独自レンダラ）。
- **当たり判定（ここが核心）**:
  1. エンティティを **hard-collidable** にする: `Entity.canBeCollidedWith()` を `true`（boat/shulker と同じ）。プレイヤーの移動処理がこのエンティティの AABB に衝突するようになる（`makeBoundingBox()` で箱を定義）。
  2. **上に乗せて運ぶ**: プレイヤーはエンティティ上面に「立てる」が、**エンティティが動いても自動では運ばれない**。毎tick、エンティティの上に居る対象を検出し、エンティティの移動分だけ**手動で translate** する（Create/Moving Elevators が実装しているのはここ）。
     - 別解: **passenger（乗員）** にする（boat方式の `positionRider`）＝プレイヤーを固定して運ぶ。歩き回れないが実装は確実。
  3. **押しのけ**: `push()` で周囲エンティティを押す。
- **性能対策（Moving Elevators）**: 「**床レベルのブロックだけ判定**」など、毎tick数百ブロックを走査しない工夫。性能が最大の敵。
- **Create の方針**: contraption は entity、entity は contraption に衝突、contraption は特定の world-interaction ブロック経由でのみ world と衝突（無制限な相互衝突を避けて破綻を防ぐ）。
- 長所: 大型・長距離・任意形状・滑らか。短所: 衝突/乗員/チャンク/同期/レンダリングを全部自作。**Create API 等の専用ライブラリに乗るのも現実的**。

---

## 当たり判定 技術メモ（実API）

- **ブロック側（可変衝突）**:
  - `BlockBehaviour#getCollisionShape(state, level, pos, CollisionContext)` … 物理衝突。`getShape` … 選択枠。
  - `VoxelShape` は `Shapes.box(x1,y1,z1,x2,y2,z2)` / `Shapes.block()` / `Shapes.empty()` / `Shapes.or(a,b...)` / `Shapes.join(a,b,BooleanOp)` で合成。**blockstateやtickで返す形状を変える**＝可変当たり判定（B/Cの土台）。
- **エンティティ側**:
  - `Entity#canBeCollidedWith()`（true で硬い衝突＝乗れる/ぶつかる）、`isPushable()`、`push(...)`、`makeBoundingBox()`、`positionRider(...)`（乗員配置）、`noPhysics`。
  - EntityType を `DeferredRegister`(`Registries.ENTITY_TYPE`) で登録し、`EntityRenderer` をクライアントで紐づける。
- **見た目だけ動かしたい**: `Display.BlockDisplay`（blockstateを描画、当たり判定なし）＋衝突は別途 B などで用意、という組み合わせも可。

---

## 選び方（要件別）

| やりたいこと | 推奨 |
| --- | --- |
| 階を移動するだけのエレベーター | **A テレポート**（最速・堅牢） |
| 自動ドア | **A 瞬間** or **B スライド**（VoxelShape段階＋BER） |
| 短距離の押し出し・せり上がる床 | **C ピストン式** or **B** |
| 乗って動く大型プラットフォーム・カスタム乗り物 | **D エンティティ**（最強・最重、Create API検討） |

---

## 難所（D系で特に）
- **性能**: 毎tickの衝突判定・ブロック走査は重い → 動く部分/床だけに絞る。
- **クライアント同期**: 位置・進捗をスムーズに補間（サーバtickは20Hz、描画は補間必須）。カクつき対策。
- **乗員の巻き込み**: 上のプレイヤー/mob/アイテムを取りこぼさず運ぶ（AABB上面判定＋translate）。
- **チャンク境界/アンロード**をまたぐ移動。
- **他MODブロックの持ち運び**（BlockEntity・tick・Capabilityの引き継ぎ）は Create 級の難題。無理をせず「自前ブロックだけ運ぶ」に絞るのが現実的。

## まとめ
- Minecraft に「動く当たり判定ブロック」の汎用機能は無い。**見た目と衝突を分けて組む**のが全流派共通。
- 迷ったら **A（動かさない）** から。動きが要るなら **B（VoxelShape段階）**、本格的な移動は **D（エンティティ）**。C はピストン相当の特殊用途。
- D の大型は自作より **Create 等のフレームワーク採用**も強く検討する。
