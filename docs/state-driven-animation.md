# 状態駆動アニメーション（スタンバイ ⇄ 稼働）(NeoForge 1.21.1)

「同じ形成状態でも、スタンバイ/稼働で機構の動きがガラッと変わる」の実装ノウハウ。
実装は `assembler` で完了済み（2026-07-10）。関連: [multiblock-visual-formation.md](multiblock-visual-formation.md)。

---

## 核心パターン

「サーバは瞬時切替、クライアントは補間して滑らかに動く」の分離が最重要:

```
[Server]          サーバ側で ACTIVE(bool) を管理
   │ setBlock(...) で blockstate に反映
   ▼
[Blockstate]      ACTIVE=true/false（自動同期）
   │ BER が毎フレーム読む
   ▼
[Client / BER]    clientAnimActivity(0..1) を目標値へ毎フレーム lerp
   │             → 加速→定常→減速→停止の連続変化
   ▼
[描画]            回転速度・振幅・追加要素の可視性 = f(clientAnimActivity)
```

- **サーバ**: 状態は bool（`true`/`false`）で扱うのが楽（NBT保存もそのまま）。
- **クライアント**: bool を直に使うと**カクッと**切り替わる。BER側で 0..1 の連続値へ**補間**することで滑らかに。
- **単一の状態値で複数要素を同時制御**: 回転速度・スケール・オービットの出現・振幅…を全部 `t` の関数にすると設計が単純。

---

## 実装（assembler の場合）

### 1) blockstate プロパティを追加（サーバ→クライアント同期の器）

```java
public static final BooleanProperty ACTIVE = BlockStateProperties.POWERED;

@Override
protected void createBlockStateDefinition(StateDefinition.Builder<Block, BlockState> b) {
    b.add(FORMED, ACTIVE);
}
```

`registerDefaultState` にも初期値。既存プロパティを増やしたので **datagen の variant を全組合わせに更新**する必要あり（`ModBlockStateProvider` を参照）。

### 2) サーバ側の状態切替

```java
public boolean toggleActive() {
    if (level == null || level.isClientSide || !formed) return active;
    active = !active;
    level.setBlock(worldPosition,
            getBlockState().setValue(ACTIVE, active), Block.UPDATE_ALL);
    setChanged();
    return active;
}
```
- `setBlock` により blockstate がクライアントへ自動同期される。
- `unform()` では `active=false` にもリセット（ゴースト状態防止）。
- NBT に `active` を保存（chunk save/load で復元）。

### 3) 操作トリガー（用途例）

`useWithoutItem` で右クリック挙動を段階分岐:
- 未形成 → 形成トライ
- 形成中 + スニーク → 解除
- 形成中 → **ACTIVE トグル**

### 4) クライアント側の補間状態

BE に **transient フィールド**（永続化しない・サーバ未使用）を1つ:

```java
public float clientAnimActivity = 0.0F;
```

BER の render 毎に:

```java
float target = state.getValue(ACTIVE) ? 1.0F : 0.0F;
be.clientAnimActivity = Mth.lerp(0.08F, be.clientAnimActivity, target);
float t = be.clientAnimActivity; // 0..1
```

- `Mth.lerp(0.08F, current, target)` は毎フレーム 8% だけ目標へ寄せる指数移動平均。加速/減速が自然に見える。
- **注意**: このコードはフレームレート依存。厳密な時間制御が要る場合は Δt を測って係数を調整（60fps を基準に補正など）。デモ用途なら十分。

### 5) t を複数の描画パラメータに乗算

単一の `t` で複数の見た目要素が連動する:

| パラメータ | スタンバイ (t=0) | 稼働 (t=1) |
| --- | --- | --- |
| 回転速度 | 1.5°/tick | 13.5°/tick |
| 上下振幅 | 0.12 (ゆったり) | 0.07 (小刻み) |
| 炉心スケール | ×0.60 | ×0.75 |
| オービットの存在 | なし (`t > 0.02` で出現) | 2つが逆回転で周回 |
| オービットのサイズ | 0 | ×0.22 |
| ブロック光量(サーバ) | 8 | 15 |

t の中間値では自然にフェードイン/アウトする。**「サーバは true/false、クライアントは連続」** の分離が肝。

---

## 応用パターン

- **エネルギー連動**: サーバtickerを付け、FE残量≥閾値なら ACTIVE=true、枯渇でfalse。→ 外部から給電→自動起動、切れると自然停止。
- **多段階状態**: 単一 bool でなく IntegerProperty(0..N)（`OVEN_LEVEL` 等）にして、クライアントで複数レイヤーの補間。
- **音**: サーバで start/stop、BER で「動作音が徐々に大きく」を音量補間で表現。
- **パーティクル**: t が閾値を超えたら BE の `getLevel().addParticle(...)` で発煙・火花。BEでなくBERから `Minecraft.getInstance().level.addParticle(...)` でクライアント専用エフェクトを追加してもいい。

---

## ハマりどころ（実装時）

- **BEフィールドで判定するとクライアントで常にfalse**（bool `active` を直接 BER で見ない）。同期される blockstate を読むのが正解（[multiblock-visual-formation.md の落とし穴](multiblock-visual-formation.md) と同根）。
- **blockstateプロパティ増設時は datagen 全variant更新**（`getVariantBuilder` の partialState を FORMED×ACTIVE の 4 組で書く）。忘れると "missing blockstate variant" 系エラー。
- **BER の render() は毎フレーム呼ばれる**（tickではない）ので、lerp係数は「1フレームあたりの寄せ幅」で考える。tick単位の想定で書くと超スロー / 超高速になる。
- **transient フィールド**を BE に足すと server では初期値のまま残る（save/load 不要）。サーバtickで触らないよう注意。
- **見た目の"稼働感"は複数要素の重ね合わせ**で作る（回転だけだと単調）。振幅・スケール・追加要素・光量を同じ `t` で連動。
