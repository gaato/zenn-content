---
title: "LinuxのWineでMikuMikuDanceを動かす"
emoji: "💃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["linux", "wine", "mmd", "mikumikudance"]
published: true
---

## はじめに

MikuMikuDance（MMD）を Linux 上の Wine で動かそうとすると、左側のボーン操作パネルや下側のタイムラインが真っ黒になって操作できない問題に遭遇しました。この記事では解決策をまとめます。

## 結論

必要なのは以下の 2 点です。

1. **64bitOS 版の MMD を使う**
2. **DXVK をインストールする**

## 環境

- OS: openSUSE Tumbleweed (Kernel 6.19.8)
- デスクトップ: KDE Plasma (Wayland)
- GPU: NVIDIA GeForce RTX 3060 (ドライバ 580.142)
- Wine: 11.4
- winetricks: 20260125

## 手順

### 0. Wine と unar をインストール

お使いのディストリビューションのパッケージマネージャまたは [WineHQ](https://wiki.winehq.org/Download) からお好きな方法でインストールしてください。

筆者は openSUSE を使用しているので以下のようにインストールしました。

```bash
sudo zypper in wine winetricks unar
```

### 1. MMD (64bitOS Ver) をダウンロード・展開

[公式サイト](https://sites.google.com/view/vpvp/) から「1.4 MikuMikuDance(64bitOS Ver)」をダウンロードします。一番上にある「1.1 MikuMikuDance」は古いバージョンで Wine では UI が正しく表示されないため（筆者は最初これをダウンロードしてハマりました）、必ず 64bitOS Ver を選んでください。

コマンドでダウンロードする場合は以下のようになります。（記事執筆時点）

```bash
wget "https://drive.google.com/uc?id=1Iucxu0tDsD05Siyv8VBGgm9vjD-f-RhM&export=download" -O mmd_x64.zip
```

zip 内のファイル名が Shift-JIS でエンコードされているため、普通に `unzip` すると日本語ファイル名が文字化けします。`unar` を使うとエンコーディングを指定して展開できます。

```bash
unar -e shift-jis mmd_x64.zip
```

### 2. DXVK と CJK フォントをインストール

DXVK は Direct3D 9/10/11 の呼び出しを Vulkan に変換するレイヤーです。これがないと UI パネルが真っ黒になります。CJK フォントがないと UI が豆腐（□□□）になります。

```bash
winetricks dxvk cjkfonts
```

:::message
DXVK なしだと左側・下側のパネルが真っ黒になります。DXVK が必須です。
:::

### 3. 起動

```bash
cd MikuMikuDance_v932x64
wine MikuMikuDance.exe
```

## トラブルシューティング

### 起動時にエラーが出る

公式サイトでは Visual C++ 2008/2010 再頒布可能パッケージと DirectX エンドユーザーランタイムのインストールが案内されています。Wine では winetricks で代替できます。筆者の環境 (Wine 11.4) ではこれらなしでも動作しましたが、環境によっては必要になるかもしれません。

```bash
winetricks vcrun2008 vcrun2010 d3dx9
```

### UI が黒い

DXVK が正しくインストールされているか確認してください。`winetricks dxvk` を再実行するか、別のプレフィックスで試してみてください。

```bash
WINEPREFIX=~/.wine-mmd winetricks dxvk cjkfonts
WINEPREFIX=~/.wine-mmd wine MikuMikuDance.exe
```

## まとめ

- MMD を Wine で動かすなら **64bitOS 版 + DXVK** の組み合わせが必要

## 参考

- [VPVP（MMD 公式サイト）](https://sites.google.com/view/vpvp/)
- [WineでMMDを動かそうとしているのですが、操作する所が真っ黒にな... - Yahoo!知恵袋](https://detail.chiebukuro.yahoo.co.jp/qa/question_detail/q14325584354)
- [Running MikuMikuDance in Linux using Wine and DXVK - LearnMMD](https://learnmmd.com/http:/learnmmd.com/wine-mmd-mikumikudance-linux/)
- [DXVK - GitHub](https://github.com/doitsujin/dxvk)
