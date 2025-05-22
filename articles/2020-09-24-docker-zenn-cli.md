---
title: "Zenn CLIのDockerfileとその使い方の紹介"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zenncli", "docker"]
published: true
---

Zenn CLIはNode.js製のZennのローカル編集をサポートするためのツールです[^1][^3][^4]。
ZennはGitHubリポジトリを連携してコンテンツを管理することができ[^2]、そのコンテンツを作成、プレビューするのに用いられます。

このZenn CLIをPCにインストールせずにDockerから扱えるようにするために、Dockerfileを書いたのでその使い方を紹介します。

PCにZenn CLIをインストールしなくてよいので、Dockerさえインストールされていれば、Zenn CLIを使用することができます。
Node.jsをインストールする必要もありませんし、すでにインストールされているNode.jsのバージョンについて気にする必要もありません。

なお、私はZennの中の人ではないので、Zenn CLI本体についてはよくわかっていない部分もあります。

[^1]: GitHubリポジトリはこちらです。
https://github.com/zenn-dev/zenn-editor

[^2]: 2020年9月24日現在、GitHubと連携する機能についてはベータ版とのことです。
https://zenn.dev/zenn/articles/connect-to-github

[^3]: Zenn CLIのインストール方法はこちらの公式記事に書かれています。  
https://zenn.dev/zenn/articles/install-zenn-cli

[^4]: Zenn CLIの使用方法についてはこちらの公式記事に書かれています。  
https://zenn.dev/zenn/articles/zenn-cli-guide

## 作成したもの

GitHub上で公開しています。

[![Tiryoh/docker-zenn-editor - GitHub](https://gh-card.dev/repos/Tiryoh/docker-zenn-editor.svg?fullname=)](https://github.com/Tiryoh/docker-zenn-editor)

## 実行例

[![Image from Gyazo](https://i.gyazo.com/ecac97b023646f69acc884ab1f3e6bc6.gif)](https://gyazo.com/ecac97b023646f69acc884ab1f3e6bc6)

コマンドを毎回入力するのは面倒なのでシェルの入力保管に全力で頼っていますが、`create-new.sh`や`preview.sh`としてスクリプトを書いても同様に楽できるかと思います。

## ビルド方法

DockerがインストールされているPC上で[Tiryoh/docker-zenn-editor](https://github.com/Tiryoh/docker-zenn-editor)をダウンロードし、Dockerイメージを作成します。

```
$ git clone https://github.com/Tiryoh/docker-zenn-editor.git
$ cd docker-zenn-editor
$ docker build -t tiryoh/zenn:latest .
```

※Zenn CLIは頻繁にアップデートされているようです。Zenn CLIで何かうまく行かないことがあればDockerイメージを再作成してみてください。
[![Image from Gyazo](https://i.gyazo.com/ca4fbd4a33189277b52848c3668d40d0.png)](https://gyazo.com/ca4fbd4a33189277b52848c3668d40d0)
https://www.npmjs.com/package/zenn-cli

## 使い方

基本的にはZenn公式の📘[Zenn CLIを使ってコンテンツを作成する](https://zenn.dev/zenn/articles/zenn-cli-guide)で書かれているコマンドの
`npx zenn`を`docker run --rm -v $PWD:/work tiryoh/zenn`に読み替えて使用します。


### 記事を作成する

```
$ docker run --rm -v $PWD:/work tiryoh/zenn new:article
```

PCで使用しているOSがUbuntu等Linuxの場合は、ファイルパーミッションの問題で保存できなくなってしまう場合があるのでオプションでユーザを指定します。

```
$ docker run --rm -u $(id -u):$(id -g) -v $PWD:/work tiryoh/zenn new:article
```

### プレビュー

```
$ docker run --rm -p 8000:8000 -v $PWD:/work tiryoh/zenn
```
