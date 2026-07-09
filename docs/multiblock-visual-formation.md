# マルチブロック形成時に見た目が激変する原理 (NeoForge 1.21.1)

「バラバラの素材ブロックを組んで形成した瞬間、1つの精密な機械の見た目に一変する」——IE等のあの仕組みの原理。実クラス（IE / vanilla）を `javap` で確認。関連: [multiblock-architecture.md](multiblock-architecture.md)。

## 核心

見た目の激変 = **形成が「データ(blockstate/ブロック)」を切り替え、描画がそのデータに追従する**、というだけ。追従のさせ方に2つの主要手法があり、IE級の“別物になる”見た目は主に**手法B**。

---

## 手法A: blockstate バリアント切替（各ブロックが自分のモデルを変える）

- 各ブロックが `formed`（＋構造内の位置/向き）プロパティを持つ。blockstate JSON で値ごとに**別モデル**へマップ。
- 各パーツが「完成機械の“自分の担当部分”」モデルを表示 → 集まると1つの機械に見える。
- **静的モデル・軽量・実装容易**。ただし「複数ブロックにまたがる大きな連続造形」は各セルに分割したモデルを用意する必要があり、複雑な形ほど手間。
- 我々の `assembler` の `FORMED`→lightLevel(発光) はこの簡易版。テクスチャ/モデルを formed で差し替えれば“見た目”も変わる。

## 手法B: マスターの BlockEntityRenderer で完成形を丸ごと描画（IEの本命）

- マスターBEに **`BlockEntityRenderer`(BER)** を付け、**完成機械の大きなモデルを1回で描画**（マスター位置基準）。
- 構成ブロックは自分のモデルを描かない → `getRenderShape` を **`RenderShape.INVISIBLE`** に。
- 結果、「バラバラの素材ブロック(unformed)」が「1つの精密な機械(formed)」へ一変して見える。
- BERなので**動的**: 稼働アニメ・回転パーツ・発光・状態別の差分描画ができる。
- **当たり判定は各パーツブロックの `getShape`/`getCollisionShape` が担う**（描画はBER、衝突はブロック形状、と役割分離）。

---

## IEの実装（実クラスで確認）

`blusunrize.immersiveengineering.api.multiblocks.*`:

- **パーツ** = `MultiblockPartBlock`（`Block implements EntityBlock`）。全パーツがBlockEntityを持ち、`getShape`/`getCollisionShape` が**状態依存で機械の実形状**を返す。形成時に構成ブロックを **dummyパーツ**（`MultiblockBlockEntityDummy`）へ置換。
- **マスター** = `MultiblockBlockEntityMaster`。完成機械を描く BER = **`MultiblockRenderer`**（`extends BlockEntityRenderer<MultiblockBlockEntityMaster>`）。`IModelOffsetProvider` でモデル位置を微調整。
- 形状定義 = `TemplateMultiblock`（構造テンプレート .nbt）。

→ IEの激変の正体は **「素材ブロックの個別モデル(unformed)」→「マスターのBERが描く完成モデル＋パーツは INVISIBLE(formed)」への切替**。そしてパーツの `VoxelShape` が当たり判定を保証するので、見た目はBER1枚でも“ちゃんとぶつかる/乗れる”。

---

## 技術メモ（実API）

- **`RenderShape`**（`getRenderShape` の戻り値）:
  - `MODEL` … 通常のブロックモデルを描く。
  - `INVISIBLE` … ブロックモデルを描かない（＝BER任せ / 完全透明）。BERで丸描きするブロックはこれ。
  - `ENTITYBLOCK_ANIMATED` … 旧式のBER描画指定（現在はINVISIBLE＋BERが主）。
- **BER**: `BlockEntityRenderer.render(be, partialTick, PoseStack, MultiBufferSource, packedLight, packedOverlay)`。大型は `getRenderBoundingBox` を広げないとカリングで消える。
- **登録**: クライアント専用イベント **`EntityRenderersEvent.RegisterRenderers`**（MODバス）で
  `event.registerBlockEntityRenderer(MY_BE.get(), MyRenderer::new)`。→ `TechLabClient`（`dist=CLIENT`）側に置く。
- **描画内容**: BER内で baked model（json/obj）を `BlockRenderDispatcher` 等で描くか、`PoseStack`＋頂点で直描き。完成形の見た目は baked model を用意することが多い（複雑な機械は .obj）。

---

## 我々の `assembler` に応用（手法A：実装済み）

**手法A を実装済み**（2026-07-09）。`ModBlockStateProvider` で `getVariantBuilder` を使い、`FORMED`(=LIT) の値で描画モデルを切り替える:
- 未形成 (`lit=false`) → `assembler_controller` / `assembler_casing`（素の金属）。
- 形成後 (`lit=true`) → `assembler_controller_on` / `assembler_casing_on`（発光パネル/配線が光る稼働テクスチャ）。
- 併せて lightLevel で発光もするので、右クリック形成の瞬間に**テクスチャが一変＋光る**。
- アイテムの手持ち見た目は未形成モデル（`simpleBlockItem(block, offModel)`）。

→ blockstate variant を2種出して別テクスチャに割り当てるだけ。**最小コストで「形成で見た目が変わる」を体験できる**構成。

- **本格（手法B・未実装）**: コントローラBEに BER を付け、完成形の大きなモデルを描画。コントローラ/ケーシングは formed 時に `RenderShape.INVISIBLE`。当たり判定は各ブロックの `getCollisionShape`/`getShape` を formed に応じて機械形状に。IE級の“別物になる”見た目はこちら。

---

## まとめ

- 「見た目が激変」= **形成でデータを切替 → 描画が追従**。魔法ではない。
- 手のかからない差分は **A（blockstate variant）**、IE級の“別物になる”見た目は **B（マスターBERで完成形を丸描き＋パーツ INVISIBLE）**。
- **当たり判定は常にブロックの `getCollisionShape` が担い、描画(BER)と分離する**のがIEの設計の勘所。だからBER1枚のド派手な見た目でも物理は破綻しない。
