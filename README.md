# TechLab（仮称）

Minecraft Java 版の 工業系MOD 開発プロジェクト（**NeoForge** / Minecraft **1.21.1** / Java **21**）。
`TechLab` は仮の名前です（`gradle.properties` の `mod_name` / `mod_id` で変更可能。手順は CLAUDE.md「mod id / パッケージのリネーム」）。

現在は**サンプルで知見を高める段階**。各機能は「実装パターンの実例」として作っており、対応するドキュメントが `docs/` にあります。

## 実装済みの機能

| 機能 | 見どころ（実装パターン） | ドキュメント |
| --- | --- | --- |
| サファイア素材・ブロック | DeferredRegister、共通タグ `c:` | DESIGN.md |
| FEエネルギー機械 | BlockEntity + `IEnergyStorage` Capability 公開（他MODのケーブルと自動連携） | DESIGN.md |
| マルチブロック「アセンブラ」 | コントローラ/パーツ方式、Capability 委譲（FE+アイテム）、形成検出 | [docs/multiblock-architecture.md](docs/multiblock-architecture.md) |
| アセンブラの GUI と加工 | カスタム Recipe（順不同マッチ）+ Menu/Screen + ContainerData 同期 | [docs/custom-gui-and-processing.md](docs/custom-gui-and-processing.md) |
| 状態駆動アニメーション | サーバ blockstate 切替 + クライアント lerp 補間 | [docs/state-driven-animation.md](docs/state-driven-animation.md) |
| ベルトコンベア | 薄板 VoxelShape + `entityInside` 搬送 + 端点で `IItemHandler` 投入 | [docs/belt-conveyor.md](docs/belt-conveyor.md) |
| アイテムカタパルト | **BlockEntity 無しで Capability を公開**、挿入即 45° 放物線発射 | [docs/catapult.md](docs/catapult.md) |
| AE2 連携 (ME Connector) | optional-compat 隔離パターン | DESIGN.md |
| datagen | blockstate/model/loot/tags/lang/recipe の自動生成 | DESIGN.md |
| **GameTest 自動テスト** | 機能テスト 4 本を ~1.5 秒で実行、mutation testing で検出力実証済み | [docs/gametest-setup.md](docs/gametest-setup.md) |

## クイックスタート

```powershell
# JAVA_HOME を通す（初回のみ / 新しいシェルごと）
$env:JAVA_HOME = [System.Environment]::GetEnvironmentVariable('JAVA_HOME','Machine')

# 開発クライアントを起動して動作確認
.\gradlew.bat runClient

# 機能自動テスト（~1.5秒、コミット前に推奨）
.\gradlew.bat runGameTestServer

# 配布用 jar をビルド（build/libs/ に出力）
.\gradlew.bat build
```

初回はNeoForge/Minecraftのダウンロードと復号で数分かかります。

## ドキュメント

- **[CLAUDE.md](CLAUDE.md)** — 環境・コマンド・アーキテクチャ・デバッグ手順
- **[DESIGN.md](DESIGN.md)** — 設計方針・要素の追加手順・ハマりどころ・docs 索引
- **[docs/review-2026-07-11.md](docs/review-2026-07-11.md)** — 直近の総括レビュー（既知の問題と負債の優先順位）

## 参考

- NeoForge ドキュメント: https://docs.neoforged.net/
- NeoForged Discord: https://discord.neoforged.net/
- マッピングのライセンス: https://github.com/NeoForged/NeoForm/blob/main/Mojang.md
