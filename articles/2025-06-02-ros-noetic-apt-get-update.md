---
title: "2025年6月以降ROS環境でapt-get updateするときにGPG errorが発生する場合の対処方法"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ros"]
published: true
---

2025年5月でROS1はEOLを迎えました。

https://www.ros.org/blog/noetic-eol/

2025年6月1日以降、ROSインストール済みの環境で`apt-get update`するときにGPG errorが発生する場合があります。

```
W: GPG error: http://packages.ros.org/ros/ubuntu focal InRelease: The following signatures were invalid: EXPKEYSIG F42ED6FBAB17C654 Open Robotics <info@osrfoundation.org>
E: The repository 'http://packages.ros.org/ros/ubuntu focal InRelease' is not signed.
```

:::message
当初ROS Noeticに限定して書いていましたが、ポイントは、**2025年5月28日以前にROSをインストールしたかどうか**です。
ROS 1なのかROS 2なのかに関わらず、2025年5月28日以前にインストールしている場合、2025年6月1日以降に影響を受けます。
https://discourse.ros.org/t/ros-signing-key-migration-guide/43937
:::

## 原因と対処方法

原因は今まで使用されていた公開鍵が2025年5月末で期限切れになっていることです。

```sh-session
$ gpg --show-keys /usr/share/keyrings/ros-archive-keyring.gpg
pub   rsa4096 2019-05-30 [SC] [expired: 2025-06-01]
      C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654
uid                      Open Robotics <info@osrfoundation.org>
```

5月28日にros/rosdistroリポジトリに新しい公開鍵が追加されています。
5月28日以降にROS/ROS 2をインストールした場合は、この新しい公開鍵をダウンロードしているはずなのでこれ以降に紹介する手順は一切不要なはずです。
それ以外の場合は、この新しい公開鍵への差し替えが必要になります。

https://github.com/ros/rosdistro/pull/46048


### `apt-key add`で公開鍵を追加していた場合

apt-keyで公開鍵を追加していた場合は、以下のコマンドで公開鍵を差し替えます。

```sh
sudo apt-key del C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654
curl -sS https://raw.githubusercontent.com/ros/rosdistro/master/ros.key | sudo apt-key add -
```

余談ですが、apt-keyは非推奨になっており、後述のSigned-Byでリポジトリと公開鍵を紐付ける方法が推奨されています。

https://www.clear-code.com/blog/2021/5/5/best-practice-about-deprecated-apt-key.html

https://github.com/ros2/ros2_documentation/pull/1189

### Signed-By でリポジトリと公開鍵を紐付けていた場合

Signed-By で下記のようにリポジトリと公開鍵を紐付けていた場合は、

```txt:/etc/apt/sources.list.d/ros1-latest.list
deb [arch=amd64 signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros/ubuntu focal main
```

```txt:/etc/apt/sources.list.d/ros2.list
deb [arch=amd64 signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu jammy main
```

以下のコマンドで公開鍵を差し替えます。

```sh
sudo curl -sS https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
```


## 確認

新しい公開鍵は2030年6月まで使えることと、`apt update`が成功することが確認できます。

```sh-session
$ gpg --show-keys /usr/share/keyrings/ros-archive-keyring.gpg
pub   rsa4096 2019-05-30 [SC] [expires: 2030-06-01]
      C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654
uid                      Open Robotics <info@osrfoundation.org>
```

## その他

あとからコピペできるように実際の使用例をメモしておきます。

### Dockerfile

新規でROSをインストールする場合はビルドしなおせばOKです。  
既存のDockerイメージをベースにそのままDockerfileに追記する場合は、以下のようにします。


**ROS 1**

```dockerfile
# Update gpg key
RUN \
  rm -rf /etc/apt/sources.list.d/ros1-latest.list && \
  apt-get update -q && \
  apt-get install -yq --no-install-recommends curl && \
  curl -sS https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg && \
  apt-key del C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654 && \
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros/ubuntu $( . /etc/os-release && echo $UBUNTU_CODENAME ) main" > /etc/apt/sources.list.d/ros1-latest.list && \
  rm -rf /var/lib/apt/lists/*
```

**ROS 2**

```dockerfile
# Update gpg key
# ROS 2 Documentationに記載されている方法では /etc/apt/sources.list.d/ros2.list
# 古い ros:jazzy-ros-base などのDockerイメージでは /etc/apt/sources.list.d/ros2-latest.list
# 新しい ros:jazzy-ros-base などのDockerイメージでは /etc/apt/sources.list.d/ros2.sources
# 新しい Dockerイメージでは更新された公開鍵がとりこまれているので、本来は更新不要
# https://github.com/ros-infrastructure/ros-apt-source/pull/8
RUN \
  rm -rf /etc/apt/sources.list.d/ros2-latest.list && \
  rm -rf /etc/apt/sources.list.d/ros2.list && \
  rm -rf /etc/apt/sources.list.d/ros2.sources && \
  apt-get update -q && \
  apt-get install -yq --no-install-recommends curl && \
  curl -sS https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg && \
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $( . /etc/os-release && echo $UBUNTU_CODENAME ) main" > /etc/apt/sources.list.d/ros2.list && \
  rm -rf /var/lib/apt/lists/*
```

### ShellScript

`apt-key add`で公開鍵を追加した場合も、Signed-Byでリポジトリと公開鍵を紐付けていた場合も、以下のコマンドでシェルスクリプトを使って公開鍵を差し替えることができます。ROS 1/2両方対応しています。

```sh
curl -sS https://raw.githubusercontent.com/Tiryoh/ros_gpg_key_update/refs/heads/master/run.sh -o ros-gpg-key-update.sh
sh ros-gpg-key-update.sh
```

```sh
# 一応こちらでも実行可能です
curl -sSL git.io/ros-gpg-key-update | sh
```

呼び出しているシェルスクリプトは以下です。[CC-0ライセンス](https://creativecommons.org/publicdomain/zero/1.0/)で公開しているので、ご自由にご利用ください。  
2019年に[ROSのbuild farmでのGPG Key差し替えになったとき](https://discourse.ros.org/t/new-gpg-keys-deployed-for-packages-ros-org/9454)に公開したものですが、まさかまた使うことになるとは思いませんでした。

https://github.com/Tiryoh/ros_gpg_key_update

これもまた余談ですが、ROS NoeticがEOLになったことから、ROS Noetic環境で`rosdep update`するときには今後は
`rosdep update --rosdistro=noetic`とするか、`rosdep update --include-eol-distros`とする必要があります。

## 参考

https://github.com/ros-infrastructure/ros-apt-source/pull/12

https://discourse.ros.org/t/ros-signing-key-migration-guide/43937

## 更新履歴

### 2025年6月2日（1）

- DockerfileとShellScriptのコードを追加
- ROS 1/2両方影響するのでNoetic限定についての記述を削除

### 2025年6月2日（2）

- Xで教えていただいたROS Discourseの記事を追記

https://x.com/ReMIX28395705/status/1929487380991332451

### 2025年6月10日

- 新しいDockerイメージでは`/etc/apt/sources.list.d/ros2.sources`になっているので、その旨を追記
