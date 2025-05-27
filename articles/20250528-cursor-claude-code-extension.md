---
title: "miseを使っている環境でClaude CodeのVSCode拡張機能をインストールする"
emoji: "🔌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mise", "cursor", "claude", "vscode"]
published: true
---
2025年5月22日のCode with Claudeで発表され、正式リリースされた[Claude Code](https://www.npmjs.com/package/@anthropic-ai/claude-code)では、VSCodeの拡張機能が提供されるようになりました。

miseを使っているためか、VSCodeの拡張機能をインストールするまでに手間取ったので、インストールするまでの手順をまとめます。

## 動作確認環境

- macOS Sequoia 15.2
    - mise 2025.3.11
    - Node.js 22.14.0
    - Cursor 0.50.6
- Ubuntu 22.04.5
    - mise 2025.4.4
    - Node.js 22.14.0
    - Cursor 0.50.2

## インストール方法

miseはインストールして環境変数まで設定してある前提です。  
下記ドキュメントなどが参考になりそうです。

https://zenn.dev/takamura/articles/dev-started-with-mise

miseでnodeをインストールし、グローバルで使えるようにします。

```bash
mise use -g node@22.14.0
```

Claude Codeをインストールします。

```bash
npm install -g @anthropic-ai/claude-code
```

VSCode（Cursor）の拡張機能としてインストールします。miseを使っている場合は、Node.jsをグローバルで直接インストールしている場合とパスが異なるようです。

```bash
cursor --install-extension ~/.local/share/mise/installs/node/latest/lib/node_modules/@anthropic-ai/claude-code/vendor/claude-code.vsix
```

`successfully installed`と表示されればインストール成功です。

```bash
Installing extensions...
Extension 'claude-code.vsix' was successfully installed.
```

公式ドキュメントに従って、Claude Codeの設定を済ませておきます。

https://docs.anthropic.com/en/docs/claude-code/overview

Cursorを再起動すると、タブの右側に「Run Claude Code」のアイコンが追加されて、拡張機能が利用できるようになります。

![](/images/cursor-claude-code-extension.png)

## 参考文献

https://zenn.dev/schroneko/articles/code-with-claude#claude-code-%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6

https://zenn.dev/kesu/articles/mise-backends

https://gist.github.com/sotayamashita/3da81de9d6f2c307d15bf83c9e6e1af6
