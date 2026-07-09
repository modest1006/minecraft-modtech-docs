> 📄 これは Minecraft 工業系MOD 学習プロジェクト（コードネーム **TechLab**・仮称）の**ドキュメントのみ**を切り出したレビュー用リポジトリです。
> ソースコードは含めていないため、本文中の `src/...` へのリンクは解決しません（MD同士のリンクは有効）。

# TechLab（仮称）

Minecraft Java 版の 工業系MOD 開発プロジェクト（**NeoForge** / Minecraft **1.21.1** / Java **21**）。
`TechLab` は仮の名前です（`gradle.properties` の `mod_name` / `mod_id` で変更可能。手順は CLAUDE.md「mod id / パッケージのリネーム」）。

現在は**サンプルで知見を高める段階**。サファイアの基本サンプルと、他MOD連携のFEエネルギー機械を同梱しています。

## クイックスタート

```powershell
# JAVA_HOME を通す（初回のみ / 新しいシェルごと）
$env:JAVA_HOME = [System.Environment]::GetEnvironmentVariable('JAVA_HOME','Machine')

# 開発クライアントを起動して動作確認
.\gradlew.bat runClient

# 配布用 jar をビルド（build/libs/ に出力）
.\gradlew.bat build
```

初回はNeoForge/Minecraftのダウンロードと復号で数分かかります。

## ドキュメント

- **[CLAUDE.md](CLAUDE.md)** — 環境・コマンド・アーキテクチャ・デバッグ手順
- **[DESIGN.md](DESIGN.md)** — 設計方針・要素の追加手順・ハマりどころ

## 参考

- NeoForge ドキュメント: https://docs.neoforged.net/
- NeoForged Discord: https://discord.neoforged.net/
- マッピングのライセンス: https://github.com/NeoForged/NeoForm/blob/main/Mojang.md
