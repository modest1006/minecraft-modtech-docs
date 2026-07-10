# ベルトコンベア (NeoForge 1.21.1)

「上に乗ったエンティティ／ItemEntity を一方向に流し、端で機械のインベントリに投入する」ブロックの実装。
Phase 1（最小版）は `techlab:conveyor` として実装済み（2026-07-10）。

関連: [moving-blocks-collision.md](moving-blocks-collision.md) — 「動くブロックと当たり判定」の4アプローチ。ベルトはその中で「静的当たり判定＋entityInside で押す」という最軽量の路線。

---

## 全体像（Phase 1）

```
[プレイヤー/MOB/ItemEntity]  ← 「乗る」= 薄板の上に接地
        │
        │  entityInside(state, level, pos, entity)
        ▼
   [ConveyorBlock]  FACING（水平4方向）を持つ
        │  1. 進行方向へ motion 加算（BELT_SPEED 上限）
        │  2. 中心線への引き戻し（センタリング）
        │  3. ItemEntity のみ端点判定 → 次ブロックの IItemHandler へ insertItemStacked
        │
   [端点の機械]  IItemHandler capability を公開しているならアイテムを吸収
```

## 実装ポイント

### 薄い VoxelShape が全ての鍵

- **完全な `Shapes.empty()` にすると entityInside が呼ばれない**（衝突検出の対象外になる）
- **完全な `Shapes.block()` にするとエンティティが「上に乗る」でなく「横に立つ」形になる**
- **正解: 高さ 2/16 の薄い板** (`Block.box(0,0,0, 16,2,16)`)。乗る側は薄板の上面に接地しつつ AABB がわずかに重なり、entityInside が発火する。

```java
private static final VoxelShape SHAPE = Block.box(0, 0, 0, 16, 2, 16);
```

### 進行方向は `HorizontalDirectionalBlock.FACING`

- 水平4方向のみで OK（上り/下りは Phase 3 で）
- `getStateForPlacement` で `ctx.getHorizontalDirection()` を採用 → プレイヤーの向いた向きに流れる
- `rotate` / `mirror` を FACING に対して実装（vanilla の Furnace 等と同流儀）

### entityInside の中身

```java
Direction dir = state.getValue(FACING);
Vec3 vel = entity.getDeltaMovement();

// 進行方向速度が BELT_SPEED を下回っていれば差分だけ加算（上限型・「押し」でなく「加速」）
double curForward = vel.x * dir.getStepX() + vel.z * dir.getStepZ();
if (curForward < BELT_SPEED) {
    double diff = BELT_SPEED - curForward;
    addX += diff * dir.getStepX();
    addZ += diff * dir.getStepZ();
}

// センタリング: 進行と直交する方向にゆるく戻す
if (dir.getAxis() == Axis.Z) {
    addX += (centerX - entity.getX()) * CENTERING;
} else {
    addZ += (centerZ - entity.getZ()) * CENTERING;
}
```

- **上限型加速**が正解。「毎tick一定量加算」だと乗ってる時間に応じて青天井で吹き飛ぶ
- **センタリング係数**は `0.10` 前後が体感良好。大きいと横方向に振動する

### 端点投入（自MODの機械/他MODの機械のどちらでも動く）

```java
IItemHandler handler = level.getCapability(
    Capabilities.ItemHandler.BLOCK, targetPos, dir.getOpposite());
if (handler != null) {
    ItemStack remaining = ItemHandlerHelper.insertItemStacked(handler, stack.copy(), false);
    if (remaining.isEmpty()) itemEntity.discard();
    else itemEntity.setItem(remaining);
}
```

- 相手側の受口方向は `dir.getOpposite()`（ベルトの向く方向の反対から入る）
- `Capabilities.ItemHandler.BLOCK` は NeoForge の統一 Capability。Mekanism / IE / AE2 の受口や vanilla のホッパー・チェスト全部と通る
- 中間ベルト経由では引き込まない（`instanceof ConveyorBlock` で判定）

### 「端まで進んだかどうか」の判定

前縁 (`front edge`) を FACING に応じて計算し、ItemEntity の座標が超えたら投入試行:

```java
if (dir.getAxis() == Axis.X) {
    entityPos = itemEntity.getX();
    frontEdge = dir == EAST ? pos.getX() + 0.9 : pos.getX() + 0.1;
}
// SOUTH/EAST は上限を超えたら、NORTH/WEST は下限を下回ったら投入
```

- 前縁近傍でのみ投入することで、ベルトを通過中のアイテムを誤って途中で吸い込まない
- 誤検出のリスク: `frontEdge` を近すぎる位置に設定すると1tick跨いで超過することがあるので余裕を持たせる

## datagen 設定

- **blockstate**: `forAllStates` で FACING → Y回転にマッピング
  - モデルのアロー方向を +Z (SOUTH) にした場合、`SOUTH:0°, WEST:90°, NORTH:180°, EAST:270°`
- **model**: `parent: minecraft:block/thin_block` + `element` で `from=[0,0,0] to=[16,2,16]`。3面テクスチャ（top/side/bottom）を割り当て
- **loot**: `dropSelf`
- **tag**: `mineable/pickaxe` に追加。ただし `requiresCorrectToolForDrops()` は付けないので `needs_iron_tool` には**入れない**（＝素手でも壊せる）
- **recipe**: 鉄インゴット×3 + レッドストーン×1 で `conveyor ×4`

## テクスチャ

- `assets/techlab/textures/block/conveyor_top.png` （上面、進行方向を示すアロー）
- `assets/techlab/textures/block/conveyor_side.png` （側面/底面、金属バンド）

Phase 1 はプレースホルダ（PowerShell で程手続きに生成した16x16 PNG）。Phase 2 でUVスクロールを付ける時に、アロー付きの正しいテクスチャに差し替える。

## ブロックの Properties

```java
BlockBehaviour.Properties.of()
    .mapColor(MapColor.COLOR_GRAY)
    .strength(1.5F, 3.0F)
    .sound(SoundType.METAL)
    .noOcclusion()  // ← 薄板なので側面の隣接cullを止める（下や横のブロックが消えないように）
```

- `.noOcclusion()` が地味に重要。隣接ブロックのカリング判定に「これは不透明の立方体だ」と誤解されると隣の面が消える。薄板・非フル形状のブロックでは必須。
- `requiresCorrectToolForDrops()` は付けていない → 素手でも壊せる (`mineable/pickaxe` タグに入れるだけでよい)

## ハマりどころ

- **`Shapes.empty()` にすると entityInside が呼ばれない**。触ってほしいなら薄板必須。
- **`.noOcclusion()` を忘れると周囲のブロック面が消える**（描画は正しいが下の土ブロック等の面カリングが誤発動）
- **プレイヤーが吹き飛ぶ**: 差分加算ではなく上限到達型を使う（`if (curForward < BELT_SPEED) addSpeed = BELT_SPEED - curForward`）
- **⚠ 上面テクスチャの矢印方向とblockstate Y回転の一致（超注意）**: `UP` face のデフォルトUV は `u+ = +X (east)`, `v+ = +Z (south)`。**Y回転は上から見て時計回り (CW)**（vanilla furnace で検証: 前面が北デフォルト、y=90で東に回る）。テクスチャの矢印が画像内で `+u` に向いている（=eastを指す）なら、blockstateは `EAST:0°, SOUTH:90°, WEST:180°, NORTH:270°`。これを90°ずらすと**視覚と物理FACINGが食い違い**、「見た目は縦に流れるベルトなのに乗ると横に押される」という混乱が起きる（実体験）。
- **アイテムが端で機械に入らない**: 相手側 face が `dir.getOpposite()` 側であることを確認。ホッパーなど「上面から入れる」機械を隣に置いても水平に投入は成立しないので注意
- **中間ベルトで途中に機械があるとき**: 現状は「隣がベルトでなければ端点」判定なので、途中に機械を挟むと吸われる（意図通り）
- **多量のItemEntity で重い**: Phase 1 は生の ItemEntity なので、100個並べるとFPSに影響。Create 式（BE 内で仮想ストレージ管理）に寄せると重量減。Phase 4 以降のテーマ。

## Phase 2 以降のロードマップ

| Phase | 追加 |
| --- | --- |
| 2 | BER で UV スクロールアニメ、テクスチャ改良 |
| 3 | 上り／下り形状（SHAPE プロパティ、右クリックで切替、斜面 VoxelShape） |
| 4 | 速度違い（低/中/高速）、アイテムマージ最適化、機械からの extractItem 側の実装 |
| 応用 | Create 風の TransportedItemStack（BEフィールドで管理・レンダリング補間） |

## 使い方（アセンブラや energy_machine との連携例）

1. 「テックラボ」タブから **ベルトコンベア** を4本取り出す
2. 進行方向を向いて設置（プレイヤーの向いた方向にアローが流れる）
3. 端の位置に **エネルギー機械** や **アセンブラ・コントローラ**（マルチブロック形成済み）を隣接配置
4. コンベアの手前で **アイテムをドロップ**（Q キー等）
5. アイテムが押されて流れ、端で機械のインベントリに吸い込まれる（機械が `IItemHandler` を公開していれば）
