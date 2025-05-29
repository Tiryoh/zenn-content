---
title: "ament_cmake_autoでのincludeディレクトリのパスの扱い変更"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ros2", "ament"]
published: true
---

ament_cmake_autoを使ったROS 2のパッケージをcolcon buildする際に、最近（[2.5.4](https://github.com/ament/ament_cmake/blob/2.5.4/ament_cmake_auto/CHANGELOG.rst)以降）includeディレクトリのパス周りが更新されていることに気づき、その背景と対応について調べたのでまとめます。

https://x.com/Tiryoh/status/1921799222820561200

## rcl/rclcppでのincludeディレクトリのパスの扱い

ROS 2 Humble以降とそれより前ではincludeディレクトリのパスが更新されています。

※変更前後どちらでも `include <rclcpp/node.hpp>`で使用できます。

https://github.com/ros2/ros2/issues/1150

https://github.com/ros2/ros2_documentation/pull/3703

**before**
```
    /opt/ros/galactic/include/
    ├── rcl
    │   ├── node.h
    ├── rclcpp
    │   ├── node.hpp
```

**after**
```
    /opt/ros/humble/include
    ├── rcl
    │   └── rcl
    │       ├── node.h
    ├── rclcpp
    │   └── rclcpp
    │       ├── node.hpp
```


このパスの修正により`colcon build --merge-install`などでmerge installする場合にunderlay/overlayに同名のパッケージがあると衝突してしまう問題を解決できるようになったようです。

https://hans-robo.hatenablog.com/entry/2022/12/01/235528

Humbleリリース時にこの話を聞いたことがあったような気がしますが、そもそもmerge installをしていなかったり、単一ワークスペースのみで使っている場合はあまり影響がなかったので、完全に忘れていました。

## ament_cmake_autoでの対応

このincludeディレクトリのパスの変更ですが、実は最近までament_cmake_autoではこのパスの変更が反映されていなかったようでした。  
（ament_cmake_autoを使わずにament_cmakeで`CMakeLists.txt`を書いている場合は新しいincludeディレクトリのパスに変更されています）

反映されていなかった理由については、細かくは確認できていませんが、 ament_cmake_autoがあまり使われておらず影響範囲が少ないのが要因の1つではないかと思います。
※詳細未調査です。詳しい経緯をご存知の方、是非教えてください。

先日rollingとkiltedについてはデフォルトの挙動を変更する形で反映されました。

https://github.com/ament/ament_cmake/pull/540


humbleとjazzyについては、オプションを有効化することで反映できるようになり、有効化していない場合はcolcon build時にメッセージが表示されるようになりました。

> In this package, headers install destination is set to `include` by ament_auto_package. It is recommended to install `include/ros2_build_test_package` instead and will be the default behavior of ament_auto_package from ROS 2 Kilted Kaiju. On distributions before Kilted, ament_auto_package behaves the same way when you use USE_SCOPED_HEADER_INSTALL_DIR option.

https://github.com/ament/ament_cmake/pull/574

## 今後のament_cmake_autoでのCMakeLists.txtの書き方

※ament_cmake_autoを使わずにament_cmakeで`CMakeLists.txt`を書いている場合は、 影響がないので、特に対応は必要ありません。

### Rolling/Kilted

標準で新しいスタイル（`include/<package_name>/<package_name>` ディレクトリ）にインストールされるようになっています。

### Humble/Jazzy

標準ではGalacticと同じスタイルなので、新しいスタイル（`include/<package_name>/<package_name>` ディレクトリ）にインストールする場合は、`ament_auto_package`の`USE_SCOPED_HEADER_INSTALL_DIR`オプションを有効化する必要があります。

```
ament_auto_package(
    USE_SCOPED_HEADER_INSTALL_DIR
    INSTALL_TO_SHARE launch param
)
```

ただし、このオプションはament_cmake_autoの[2.5.4](https://github.com/ament/ament_cmake/blob/2.5.4/ament_cmake_auto/CHANGELOG.rst)以降（バイナリ版は2025年4月末にリリースされています）でしか対応しておらず、それ以前のバージョンでは colcon build時にエラーとなる可能性があります。

### 参考

https://github.com/ros2/ros2/issues/1150

https://github.com/ament/ament_cmake/pull/574

https://hans-robo.hatenablog.com/entry/2022/12/01/235528
