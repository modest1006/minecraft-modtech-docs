# アイテムカタパルト (NeoForge 1.21.1)

「アイテムを 45°放物線で FACING 方向へ発射する遊びブロック」の実装。
Phase 1（最小版）は `techlab:catapult` として実装済み（2026-07-11）。

関連: [belt-conveyor.md](belt-conveyor.md) — ベルトから流れてきたアイテムをそのまま飛ばすと物流アトラクションになる。

---

## 全体像（Phase 1）

```
[ベルト/ホッパー/隣接機械]
        │
        │  IItemHandler.insertItem(stack)
        ▼
   [CatapultBlock.LaunchHandler]  ← BE無しの Capability プロバイダ
        │  即座に launchStack() を呼ぶ
        ▼
   [CatapultBlock.launchStack]
        │  1. FACING 方向へ 45° 初速をつけた ItemEntity を生成
        │  2. addFreshEntity → 以降は vanilla 物理（重力 0.04/tick、ドラッグ 0.98）に任せる
        │  3. 発射音 + 煙パーティクル
        ▼
   [ItemEntity] 山なりに飛んで着弾
```

- **BlockEntity 不要**: `IBlockCapabilityProvider` を使って BE 無しでブロックに Capability を公開できる。
- **貫通しない設計**: `insertItem` は「即発射して EMPTY を返す（=全量受領）」。BEのように貯めない。
- **物理は完全に vanilla 任せ**: 初速だけ与える。

## 実装ポイント

### BE無しで IItemHandler を公開する

NeoForge の `RegisterCapabilitiesEvent#registerBlock` は、BE無しブロックにも Capability を紐付けられる。ラムダは呼ばれるたびに新しいハンドラを返せば良い（軽量オブジェクト）。

```java
event.registerBlock(
    Capabilities.ItemHandler.BLOCK,
    (level, pos, state, blockEntity, side) -> {
        Direction facing = state.getValue(HorizontalDirectionalBlock.FACING);
        if (side == facing) return null;  // 出力面からの投入は拒否
        return new CatapultBlock.LaunchHandler(level, pos, state);
    },
    ModBlocks.CATAPULT.get()
);
```

`LaunchHandler` の中身は超簡単:

```java
public ItemStack insertItem(int slot, ItemStack stack, boolean simulate) {
    if (stack.isEmpty()) return ItemStack.EMPTY;
    if (!simulate) {
        launchStack(level, pos, state, stack);  // ← ここで発射
    }
    return ItemStack.EMPTY;  // 全量受領（残り0）
}
```

- **simulate=true のときも EMPTY を返す**: `ItemHandlerHelper.insertItemStacked` は事前に simulate で確認してから本挿入する仕組み。simulate で拒否したら本挿入も来ない。
- **他のメソッド**: `getStackInSlot=EMPTY`, `extractItem=EMPTY`, `getSlots=1`, `isItemValid=true`。

### 発射物理（45°固定）

```java
double v = SPEED_TABLE[state.getValue(POWER)];    // 0.6 / 0.9 / 1.25
double vh = v * Math.sqrt(0.5);                    // 水平成分 = v·cos45°
double vx = vh * facing.getStepX();
double vz = vh * facing.getStepZ();
double vy = v * Math.sqrt(0.5);                    // 垂直成分 = v·sin45°

double x = pos.getX() + 0.5 + facing.getStepX() * 0.5;  // 銃口: 上面中央から半歩前
double y = pos.getY() + 1.0;
double z = pos.getZ() + 0.5 + facing.getStepZ() * 0.5;

ItemEntity ie = new ItemEntity(level, x, y, z, stack.copy());
ie.setDeltaMovement(vx, vy, vz);
ie.setPickUpDelay(20);       // 1秒は拾えない（自分の足元に落ちた分の誤回収防止）
level.addFreshEntity(ie);
```

- **速度単位はブロック/tick**。20 tick/秒。0.6 なら 12 blocks/sec 相当の初速。
- **飛距離目安（vanilla 物理・ドラッグ込み実測ベース）**:
  - LOW  (v=0.6)  → 水平 約 4〜5 blocks
  - MID  (v=0.9)  → 約 9〜10 blocks
  - HIGH (v=1.25) → 約 18〜20 blocks
- **PickupDelay** を付けないと、発射直後に足元でプレイヤーが拾ってしまうことがある。

### POWER プロパティ

`IntegerProperty.create("power", 0, 2)` の 3 段階。**モデルに反映しない**（見た目は変わらない）ので、blockstate は `forAllStatesExcept(POWER)` で除外する:

```java
getVariantBuilder(catapult).forAllStatesExcept(state -> {
    Direction dir = state.getValue(CatapultBlock.FACING);
    int yRot = switch (dir) { case NORTH -> 0; case EAST -> 90; case SOUTH -> 180; case WEST -> 270; default -> 0; };
    return ConfiguredModel.builder().modelFile(catapultModel).rotationY(yRot).build();
}, CatapultBlock.POWER);
```

これで variant 数は 4×3=12 でなく 4 になる（POWER 分は同じモデルを共用）。

### 操作 UX

- **素手で右クリック**: POWER を 0 → 1 → 2 → 0 とサイクル（レバー音、パワーが上がるほどピッチ↑）。
- **アイテム入力**: ベルト/ホッパー/他 MOD 機械の出力を隣接させれば `insertItem` 経由で自動発射。
- **アイテム持って右クリック**: 何もしない（`useItemOn` を override していないので、通常のアイテム使用挙動になる。要注意）。

## datagen 設定

- **blockstate**: `forAllStatesExcept(state -> ..., POWER)` で FACING → Y回転（NORTH:0, EAST:90, SOUTH:180, WEST:270 = vanilla furnace 準拠）
- **model**: `models().orientable(name, side, front, top)` を使う。orientable のデフォルト front は **NORTH**、down/up は #top を共用。
- **loot**: `dropSelf`
- **tag**: `mineable/pickaxe` に追加。`requiresCorrectToolForDrops()` は付けていないので `needs_iron_tool` には**入れない**（＝素手でも壊せる）
- **recipe**: 鉄×6 + スライム×2 + レッドストーン×1 + ピストン×1 → `catapult ×1`

## テクスチャ

- `assets/techlab/textures/block/catapult_top.png` — 上面。矢印 "^" が画像トップ（=v−方向 = UP面の -Z = NORTH）を指す。**FACING=NORTH の時に前方向を指す**ように配置しているので、Y回転 (NORTH:0/EAST:90/…) と自然に整合する。
- `assets/techlab/textures/block/catapult_side.png` — 側面。金属バンド＋4隅のリベット。
- `assets/techlab/textures/block/catapult_front.png` — 正面。中央に丸い発射口。

Phase 1 は手書きプレースホルダ（PowerShell で 16×16 生成）。Phase 2 でアームアニメを付けるときに差し替える予定。

## ブロックの Properties

```java
BlockBehaviour.Properties.of()
    .mapColor(MapColor.COLOR_GRAY)
    .strength(2.0F, 4.0F)
    .sound(SoundType.METAL)
```

`requiresCorrectToolForDrops()` は付けていない → 素手でも壊せる (`mineable/pickaxe` タグに入れるだけ)。

## ハマりどころ

- ✅ **修正済み（2026-07-11）**: 旧実装は `LaunchHandler` が生成時の `BlockState` を保持しており、パイプMODがハンドラをキャッシュすると POWER/FACING 変更が反映されない問題があった。現在は `insertItem` 内で `level.getBlockState(pos)` を読み直し、撤去済みブロックの stale ハンドラは受け取りを拒否する（[review-2026-07-11.md](review-2026-07-11.md) 指摘3）。
- **`setPickUpDelay(0)` にすると足元で即回収される**: 20 tick（1秒）以上の delay を付ける。
- **POWER 段階を blockstate に反映すると variant が 12 個になり冗長**: 見た目に絡まないプロパティは `forAllStatesExcept(POWER)` で必ず除外する。
- **上面テクスチャの矢印方向 vs Y 回転**: [belt-conveyor.md](belt-conveyor.md) と同じ落とし穴。今回は orientable モデルを使い、front テクスチャを **NORTH** 面に置いてある。それに合わせて上面矢印は「画像トップ = -Z = 北」を向ける。Y回転は vanilla furnace と同じ (NORTH:0, EAST:90, SOUTH:180, WEST:270)。
- **`insertItem` の simulate 忘れ**: 呼び出し側は simulate → 本挿入の 2 段で来るので、両方で「全量受領」を返さないと1回で終わらない事象になる（simulate で false を返すと、実際に発射する本挿入が呼ばれない）。
- **側面全部から入力を受ける**: 現状は FACING 面（出力側）だけを弾いている。TOP/BOTTOM も受け付けるので、上にホッパーを載せれば普通に流し込める。
- **エンティティ数の暴発**: ベルト+ホッパーで大量供給されると 20 発射/秒 になる。Phase 2 でクールダウン（POWER に応じた 5-10 tick 程度）を入れると自然。
- **重いスタックを飛ばすと着弾で散らばる**: 64 個スタックがそのまま1エンティティで飛ぶ。着弾時にプレイヤーがまとめて拾えるので実害はない。

## Phase 2 以降のロードマップ

| Phase | 追加 |
| --- | --- |
| 2 | BER でアームの跳ね上げアニメ、発射クールダウン、パーティクル強化 |
| 3 | ターゲットブロック指定（リンカーで座標登録 → 距離から必要初速を自動算出、任意角度） |
| 4 | 上下方向ターゲット対応（垂直発射・斜め上ブロック狙い）、着弾検知でリレー |
| 応用 | 発射エフェクトの拡張（火薬追加で爆発弾化）、プレイヤー射出モード（危険） |

## 使い方（ベルコン連携例）

1. カタパルトを配置（プレイヤーの向いた方向へ発射する）
2. 素手で右クリック → POWER を目的の距離に合わせる（LOW/MID/HIGH のクリック音でわかる）
3. カタパルトの後ろ（=FACING の逆向き）に **ベルトコンベア** を接続、その先にドロッパーやチェスト
4. ベルトにアイテムを流す → カタパルトから山なりに飛んでいく
5. 着弾地点に **チェスト** や別の機械の入力面を置けば、そのまま連鎖搬送になる（ItemEntity は水面/ホッパー面に入ればフックされる）
