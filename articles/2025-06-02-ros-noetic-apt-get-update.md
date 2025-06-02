---
title: "2025年6月以降ROS Noetic環境でapt-get updateするときにGPG errorが発生する場合の対処方法"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ros"]
published: true
---

2025年5月でROS1はEOLを迎えました。

https://www.ros.org/blog/noetic-eol/

2025年6月1日以降、ROS Noetic環境で`apt-get update`するときにGPG errorが発生する場合があります。

```
W: GPG error: http://packages.ros.org/ros/ubuntu focal InRelease: The following signatures were invalid: EXPKEYSIG F42ED6FBAB17C654 Open Robotics <info@osrfoundation.org>
E: The repository 'http://packages.ros.org/ros/ubuntu focal InRelease' is not signed.
```

:::message
ポイントは、**2025年5月28日以降にROS Noeticをインストールしたかどうか**です。
ほとんどの場合、2025年5月28日以前にROS Noeticをインストールしていると思いますので、影響を受けます。
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
5月28日以降にROS Noeticをインストールした場合は、この新しい公開鍵をダウンロードしているはずなのでこれ以降に紹介する手順は一切不要なはずですが、
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

以下のコマンドで公開鍵を差し替えます。

```sh
sudo curl -sS https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
```


## 確認

新しい公開鍵は2030年6月まで使えることが確認でき、`apt update`が成功することが確認できます。

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

### ShellScript

`apt-key add`で公開鍵を追加した場合も、Signed-Byでリポジトリと公開鍵を紐付けていた場合も、以下のコマンドでシェルスクリプトを使って公開鍵を差し替えることができます。

```sh
curl -SsL git.io/ros-gpg-key-update | sh
```

呼び出しているシェルスクリプトは以下です。CC-0ライセンスで公開しているので、ご自由にご利用ください。  
2019年に[ROSのbuild farmでのGPG Key差し替えになったとき](https://discourse.ros.org/t/new-gpg-keys-deployed-for-packages-ros-org/9454)に公開したものですが、まさかまた使うことになるとは思いませんでした。

https://github.com/Tiryoh/ros_gpg_key_update

余談ですが、ROS NoeticがEOLになったことから、ROS Noetic環境で`rosdep update`するときには今後は
`rosdep update --rosdistro=noetic`とするか、`rosdep update --include-eol-distros`とする必要があります。

## 参考

https://github.com/ros-infrastructure/ros-apt-source/pull/12
