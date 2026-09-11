---
title: "ARM ボードの UEFI ブートで Device Tree はどう渡されるか"
emoji: "🌳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["linux", "arm", "devicetree", "systemdboot", "opensuse"]
published: true
---

## はじめに

こんにちは、がーとです。

Radxa ROCK 5 ITX+（Rockchip RK3588 を載せた mini-ITX の ARM ボード）で、openSUSE MicroOS を UEFI ファームウェアと systemd-boot の組み合わせで動かしています。

このボードの NPU を Immich の機械学習に使おうと mainline カーネルの `rocket` ドライバを試したところ、`modprobe rocket` しても `/dev/accel/accel0` が出てきませんでした。openSUSE が配布する `rk3588-rock-5-itx.dtb` で NPU のノードが `disabled` になっているのが原因で、NPU を有効にした DTB に差し替えようとしたのですが、そもそも OS 側の DTB は起動に使われていませんでした。

## Device Tree とは何か

x86 の PC では、ファームウェアが ACPI テーブルでハードウェア構成をカーネルに伝えます。多くの ARM シングルボードコンピュータ（SBC）にはそれがなく、代わりに Device Tree を使います。SoC の周辺機器、割り込み、ピン配置、電源などをツリー構造で記述したもので、コンパイル済みのバイナリが DTB です。

カーネルは起動時に DTB を受け取り、そこに書かれた内容を頼りにドライバを結びつけます。DTB が渡らなければ、あるいはボードに合わない DTB が渡れば、デバイスが認識されなかったり、そもそも起動しなかったりします。NPU のノードが `disabled` になっていれば、`rocket` ドライバは NPU を見つけられません。

DTB のソース（`.dts`）は Linux カーネルのソースツリーで保守されています。DTB はカーネルの更新と同時に更新されるべきものです。

## いま渡っている DTB は何者か

まず実機で、カーネルが受け取った DTB の出どころを調べました。

`dmesg` を見ると、ファームウェアは EDK II でした。

```
efi: EFI v2.7 by EDK II
```

`/proc/device-tree/chosen` には UEFI 経由で起動したときにカーネルの EFI スタブが書き込むプロパティが並んでいます。

```
/proc/device-tree/chosen/linux,uefi-system-table
/proc/device-tree/chosen/linux,uefi-mmap-start
/proc/device-tree/chosen/linux,uefi-mmap-size
/proc/device-tree/chosen/linux,uefi-mmap-desc-size
/proc/device-tree/chosen/linux,uefi-mmap-desc-ver
```

一方で、systemd-boot が読んでいる Boot Loader Specification（BLS）のエントリには `devicetree` 行がありませんでした。ESP のどこにも `.dtb` はなく、openSUSE の `dtb-rockchip` パッケージはそもそもインストールされていません。

```
linux  /opensuse-microos/7.0.10-2-default/linux-...
initrd /opensuse-microos/7.0.10-2-default/initrd-...
```

カーネルが受け取っていた DTB は OS 側のファイルではなく、ファームウェアが自分で持っているものでした。`dtb-rockchip` を入れても、自前で NPU を有効にした DTB を置いても、誰もそれを参照していないので起動には反映されません。

## ARM ボードはどう起動するか

ROCK 5 ITX+ を例に、電源投入から systemd が上がるまでを追います。

```
SoC BootROM
  └─ Rockchip のローダ / TF-A（DRAM 初期化、EL3 ファームウェア）
       └─ EDK II (edk2-rk3588)             … ファームウェア内蔵の DTB (1)
            └─ UEFI アプリとしてブートローダを起動
                 └─ systemd-boot / GRUB    … BLS エントリの devicetree (2)
                      └─ Linux (EFI スタブ)
                           └─ カーネル本体 … 受け取った DTB でデバイス初期化
                                └─ initrd → ルート → systemd
```

### SoC の BootROM からファームウェアまで

SoC に焼き込まれた BootROM が、SPI フラッシュや eMMC から最初のステージを読み込みます。RK3588 では DRAM 初期化のコードと、EL3 で動く TF-A（Trusted Firmware-A）を経て、UEFI ファームウェア本体が起動します。ここまでは DTB の受け渡しとは関係がないので、この記事では踏み込みません。

### ファームウェアが DTB を用意する

ROCK 5 ITX+ の SPI フラッシュに入っている UEFI ファームウェアは、edk2-rk3588 プロジェクトをベースにした EDK II です。`bootctl status` でも `Firmware: UEFI 2.70 (EDK II 1.00)` と表示されます。

https://github.com/edk2-porting/edk2-rk3588

このファームウェアは設定メニューで ACPI モードと Device Tree モードを切り替えられます。Device Tree モードでは、ファームウェアに埋め込まれたボード用の DTB を読み込み、設定に応じた修正を加えたうえで、EFI の設定テーブルに `EFI_DTB_TABLE` として登録します[^1]。これが図の (1) で、実機で見えていた DTB はこれでした。

ファームウェアの DTB は、ファームウェアをビルドした時点のカーネルソースから来ています。カーネルより古いのが普通で、NPU のように後から DTB で有効化された機能は含まれていません。

edk2-rk3588 には、ブートデバイス上の `\dtb`、`\dtb\base`、`\dtb\rockchip` に置いた `<ボード名>.dtb` を代わりに読む機能もあります。ただしこの方法では、パーティションに置いた一つの DTB をすべてのカーネルで共有することになり、カーネルのバージョンごとに切り替えることはできません。

U-Boot をファームウェアにしているボードでも同じです。U-Boot にも UEFI 実装があり、内蔵の DTB を設定テーブルに登録したうえで、`fdtfile` 環境変数をもとに ESP の `/dtb/` などから DTB を探します[^2]。

### ブートローダが DTB を差し替える

systemd-boot や GRUB は、BLS エントリに `devicetree` の指定があれば、そのファイルを読み込んで設定テーブルを上書きします。これが図の (2) です。

systemd-boot の実装は `src/boot/devicetree.c` にあります。ファイルを読み、ファームウェアが `EFI_DT_FIXUP_PROTOCOL` を提供していればそれを呼んで、ファームウェアと同じ修正を DTB に適用します。最後に `InstallConfigurationTable` で `EFI_DTB_TABLE` を置き換えます。

GRUB も同様で、`devicetree` コマンドが DTB を読み込み、設定テーブルを差し替えます。

ただし edk2-rk3588 の FdtPlatformDxe を読む限り、`EFI_DT_FIXUP_PROTOCOL` は実装されていません[^1]。README にも、ファームウェア設定に応じた修正（PCIe や SATA、USB の切り替え）は GRUB の `devicetree` コマンドなどで差し替えた DTB には適用されない、と書かれています[^3]。ブートローダで DTB を差し替えると、ファームウェアの設定メニューで選んだ内容は DTB に反映されなくなります。手元の構成では NVMe も SATA も問題なく動いていますが、ファームウェア側で PCIe と SATA を切り替えている場合は気をつけてください。

### カーネルの EFI スタブが受け取る

Linux の aarch64 カーネルは EFI アプリケーションとしても動くように作られていて、その入口が EFI スタブです。EFI スタブは設定テーブルから `EFI_DTB_TABLE` を取り出し、UEFI のメモリマップなどを `chosen` ノードに書き加えたうえで、ブートサービスを終了してカーネル本体に渡します[^4]。先ほど見た `linux,uefi-*` プロパティはここで書き込まれたものです。

どこから DTB を得たかはカーネルのログに記録されます。

```
EFI stub: Using DTB from configuration table
```

コマンドラインの `dtb=` で指定する経路もありますが、これはカーネル側でオプション扱いのうえ、Secure Boot が有効なときは無視されます[^4]。

### DTB の供給源を整理する

| 供給源 | 誰が登録するか | カーネルごとの切り替え |
| --- | --- | --- |
| ファームウェア内蔵の DTB | ファームウェア | できない |
| ブートデバイスに置いた差し替え用 DTB | ファームウェア | できない |
| BLS エントリの `devicetree` | ブートローダ | できる |
| UKI に埋め込んだ `.dtb` セクション | systemd-stub | できる（署名も可能） |

カーネルパッケージが配布する DTB をカーネルのバージョンに合わせて使いたいなら、BLS エントリで指定することになります。

## BLS エントリの `devicetree` フィールド

Boot Loader Specification の Type 1 エントリは、ESP や XBOOTLDR パーティションの `loader/entries/` に置く小さなテキストファイルです。`linux`、`initrd`、`options` などと並んで `devicetree` というキーが定義されています[^5]。

```
title      openSUSE MicroOS 20260605
version    47@7.0.11-1-default
options    root=UUID=... rootflags=subvol=@/.snapshots/47/snapshot
linux      /opensuse-microos/7.0.11-1-default/linux-9724a3b6...
devicetree /opensuse-microos/7.0.11-1-default/devicetree-555d2e83....dtb
initrd     /opensuse-microos/7.0.11-1-default/initrd-9fea4bac...
```

パスはブートパーティションのルートからの相対パスです。カーネルと同じディレクトリに置けば、エントリ単位でファイルを管理できます。

systemd-boot も、BLS エントリを読む GRUB（openSUSE では grub2-bls）も、このフィールドを解釈します。`bootctl status` の Features に `Support Type #1 devicetree field` と出ていれば対応しています。

## Secure Boot と DTB

systemd-boot は Secure Boot が有効なとき、Type 1 エントリの `devicetree` を黙って読み飛ばします。

```c
/* DTBs are loaded by the kernel before ExitBootServices(), and they can be used to map and
 * assign arbitrary memory ranges, so skip them when secure boot is enabled as the DTB here
 * is unverified. */
if (entry->devicetree && !secure_boot_enabled()) {
        err = devicetree_install(&dtstate, image_root, entry->devicetree);
```

https://github.com/systemd/systemd/blob/v260.2/src/boot/boot.c#L2832-L2839

GRUB も同じ判断をしています。`devicetree` コマンドはロックダウンの対象として登録されていて、Secure Boot が有効で GRUB がロックダウンモードに入っていると実行できません[^6]。前述のとおり、カーネルの EFI スタブも `dtb=` を Secure Boot 下では無視します。

理由はコメントにあるとおりです。DTB はハードウェアの接続を定義するデータで、任意のメモリ範囲をデバイスに割り当てさせることができます。署名されていない DTB を読み込むのは、署名されていないコードを実行するのとほぼ同じです。

署名された DTB を使いたいなら、systemd の Unified Kernel Image（UKI）に `.dtb` セクションとして埋め込み、カーネルと一緒に署名する方法があります。systemd-stub は `.dtb` に加えて、ファームウェアの DTB の `compatible` プロパティを見て複数の候補から自動選択する `.dtbauto` セクションもサポートしています[^7]。

## Measured Boot と DTB

TPM で PCR 値を測り、その値でディスクを自動アンロックする構成（openSUSE のフルディスク暗号化など）では、ブート経路に入力を一つ足すと PCR が変わり得ます。

### GRUB は測る

GRUB は実行したコマンド文字列を PCR 8 に、読み込んだファイルの内容を PCR 9 に測定します[^8]。BLS エントリに `devicetree` 行があれば、GRUB は `devicetree` コマンドを実行して DTB ファイルを読むので、両方の PCR が変わります。

### systemd-boot は測らない

systemd-boot の `devicetree_install()` は、DTB を読んでファームウェア向けの修正を適用し、設定テーブルに登録するだけで、TPM への測定は行いません。

https://github.com/systemd/systemd/blob/v260.2/src/boot/devicetree.c#L65-L107

ソースを読むだけでは不安だったので、aarch64 の QEMU/libvirt 仮想マシンに swtpm で TPM2 を付け、systemd-boot 260.2 で確認しました。外部 DTB には、その VM が通常起動したときに `/sys/firmware/fdt` から取り出した実物の FDT を使っています。`devicetree` 行の有無だけが異なる 2 つの Type 1 エントリで起動を比べたところ、TPM のイベントログと PCR 9、12、15 の値は完全に一致し、DTB のハッシュはイベントログのどこにも現れませんでした。

なお UKI の `.dtb` セクションは話が別で、systemd-stub は埋め込まれた DTB や DTB アドオンを PCR に測定します[^7]。

### 測定されないことの意味

Secure Boot 無効で systemd-boot の外部 DTB を使う構成では、DTB は署名検証も TPM 測定もされません。ESP に書き込める人がハードウェア定義を差し替えても、PCR には痕跡が残りません。

署名と測定の両方を効かせたいなら、UKI に埋め込む方向になります。

## 実例: sdbootutil に devicetree 対応を入れる

openSUSE の DTB パッケージが置くファイルを、カーネルごとに BLS エントリの `devicetree` で指定すればよいことは分かりました。

### sdbootutil とは

sdbootutil は openSUSE で systemd-boot と grub2-bls を管理するツールです。Btrfs のスナップショットと snapper を前提に、カーネルパッケージのインストール時にカーネルと initrd を ESP にコピーして BLS エントリを生成し、TPM の PCR 予測を更新します。

この生成処理に DTB を扱う経路がありませんでした。生成されたエントリに手で `devicetree` 行を足しても、次のカーネル更新でエントリが作り直されて消えてしまいます。そこで sdbootutil 自体に機能を追加することにしました。

https://github.com/openSUSE/sdbootutil/pull/394

### 追加した設定

`/etc/default/sdbootutil` に `DEVICETREE_SOURCE` を書くか、コマンドラインで `--devicetree-source` を渡します。

```bash
# /etc/default/sdbootutil
DEVICETREE_SOURCE="/boot/dtb-%V/rockchip/rk3588-rock-5-itx.dtb"
```

パスは絶対パスで、2 つのプレースホルダが使えます。

| プレースホルダ | 展開結果 | 例 |
| --- | --- | --- |
| `%K` | フレーバー付きのカーネルバージョン | `7.0.11-1-default` |
| `%V` | フレーバーなしのカーネルパッケージバージョン | `7.0.11-1` |

openSUSE の `dtb-rockchip` などの DTB パッケージは `/boot/dtb-<カーネルパッケージバージョン>/` にファイルを置くので、`%V` を使えばカーネル更新に追従できます。最初は自前ビルドの NPU 有効 DTB を `%K` 付きのパスで指定して使い、後から公式の `dtb-rockchip` に切り替えました。

設定があると、カーネルインストール時に対象スナップショット内で DTB を探し、チェックサム付きの名前で ESP のカーネルと同じディレクトリにコピーし、BLS エントリに `devicetree` 行を書きます。`sdbootutil list-entries` は DTB の存在も検査し、`sdbootutil show-entry` は `devicetree` フィールドを表示します。

### 実機での結果

ROCK 5 ITX+ で `sdbootutil show-entry` を実行した結果です。

```console
# sdbootutil show-entry 7.0.11-1-default 47
title      openSUSE MicroOS 20260605
version    47@7.0.11-1-default
sort-key   opensuse-microos
options    root=UUID=421c75b1-... splash=silent ... rootflags=subvol=@/.snapshots/47/snapshot
linux      /opensuse-microos/7.0.11-1-default/linux-9724a3b65a78f9e24f56caf12324083831fa5a1d
devicetree /opensuse-microos/7.0.11-1-default/devicetree-555d2e834784f2148159da1fee8c0c2f1b499fcd.dtb
initrd     /opensuse-microos/7.0.11-1-default/initrd-9fea4bacb118fc4bf3b35cdd987763fc28d89be4
```

このエントリで起動したあと、パッケージの DTB と ESP にコピーされた DTB のハッシュが一致し、`/proc/device-tree` の内容もパッケージの DTB と一致することを確認しました。ファームウェアの DTB ではなく、OS が管理する DTB でカーネルが動いています。

この機能は openSUSE Tumbleweed の `sdbootutil-1+git20260625.7fa275e` 以降に含まれています。

### 方針とレビューでの変更

ボードの自動検出はしないことにしました。同じ SoC でもボードのリビジョンやファームウェアで必要な DTB が変わって、間違えると起動しないので、管理者がパスを書く形にしています。ボード固有の設定パッケージが `DEVICETREE_SOURCE` を置く、という形が妥当だと思っています。DTB が見つからなかったら警告ではなくエラーで、カーネルインストールごと止めます。

Secure Boot が有効なときも、設定があればカーネルインストールを止めます。メンテナからは「DTB に署名はできないのか」と聞かれたのですが、BLS エントリで外部ファイルを渡す経路には署名を検証する仕組みがそもそもなく、systemd-boot 自体が Secure Boot 下では読み飛ばすので、この組み合わせは拒否のままにしました。

最初の実装はエントリを生成するだけで、TPM のことは考えていませんでした。そうしたら最初に来たレビューが「DTB は測定されるのではないか、この PR で予測も実装できるはず」で、grub2-bls の PCR 予測に `devicetree` コマンド行を PCR 8、DTB ファイルの中身を PCR 9 として足しました。これを忘れると、DTB 付きのカーネルを入れた次の起動で自動アンロックが失敗します。続けて「systemd-boot 側の予測も可能なら追加してほしい。外部 DTB が測定されないことは確認したか」と言われて、前述のソース確認と QEMU の実験はこのときにやったものです。測定されていないと判断して systemd-boot の予測は変えず、代わりに `DEVICETREE_SOURCE` が設定された systemd-boot 環境で予測を生成するときは未検証である旨の警告を出す、という形に落ち着きました。手元の ROCK 5 ITX+ には TPM2 がないので、実機のファームウェアがどうするかは見られていません。TPM2 付きの実機で試せた方がいれば教えてください。

DTB の探し方も一回直しています。最初は ESP 配下のパスだけファームウェアのブートパーティションから解決する特別扱いをしていて、それを「カーネルのソースが ESP にあるのを認めないのと同じで、DTB も認めるべきではない」と言われて、カーネルと同じく対象スナップショットの中から探す形に揃えました。ESP へのコピーもカーネルの `linux-<チェックサム>` と同じ流儀で、同じ中身の DTB は複数のカーネルで共有されて、`bootctl cleanup` で一緒に掃除されます。

このほか `devicetree` の引数の並びを `initrd` の後ろにするとか、生成するエントリの雛形を既存の heredoc に寄せるとか、いくつかの指摘に対応して、最後は「一つの変更としてマージしたい」というメンテナの希望で 1 コミットに squash してマージ、という感じでした。

## その後: NPU はどうなったか

NPU を有効化する上流のコミット[^9]は Linux 7.2 に収録され、7.2 は 2026 年 8 月 16 日にリリースされました。Tumbleweed の aarch64 リポジトリでも 2026 年 8 月 30 日のスナップショットから `kernel-default` と `dtb-rockchip` の 7.2.2 が配信されています。

`DEVICETREE_SOURCE` を `%V` で書いてあるので、カーネルを更新すれば DTB も `/boot/dtb-<バージョン>/` のものに自動で切り替わります。7.2.3 に更新した実機では、BLS エントリの `devicetree` が 7.2.3 のものを指し、NPU のノードが `okay` になって `rocket` ドライバが 3 コアを認識しています。

```console
# sdbootutil show-entry 7.2.3-1-default 153
title      openSUSE MicroOS 20260908
version    153@7.2.3-1-default
sort-key   opensuse-microos
options    root=UUID=421c75b1-... splash=silent ... rootflags=subvol=@/.snapshots/153/snapshot
linux      /opensuse-microos/7.2.3-1-default/linux-763a5d505058d75cce11d1b5877d51eb787adfcb
devicetree /opensuse-microos/7.2.3-1-default/devicetree-41b4459da6ad7f94314d0eede5d7c040793baa11.dtb
initrd     /opensuse-microos/7.2.3-1-default/initrd-8787eeec80e6935572322b8b38acc8de3d686865
# cat /proc/device-tree/npu@fdab0000/status
okay
# dmesg | grep rocket
[   26.574852] [drm] Initialized rocket 0.0.0 for rknn on minor 0
[   26.575535] rocket fdab0000.npu: Rockchip NPU core 0 version: 1179210309
[   26.577456] rocket fdac0000.npu: Rockchip NPU core 1 version: 1179210309
[   26.581999] rocket fdad0000.npu: Rockchip NPU core 2 version: 1179210309
# ls /dev/accel/
accel0
```

## まとめ

- ARM ボードを UEFI で起動するとき、DTB は EFI 設定テーブルを介してファームウェアからカーネルの EFI スタブへ渡る。ブートエントリで何も指定しなければ、ファームウェア内蔵の DTB が使われる
- ファームウェアの DTB はカーネルより古いことが多く、カーネルごとに切り替えたいなら BLS エントリの `devicetree` フィールドを使う
- systemd-boot も GRUB も Secure Boot 有効時は外部 DTB を読まない。署名したいなら UKI に埋め込む
- GRUB は DTB を PCR 8 と 9 に測定するが、systemd-boot は Type 1 エントリの DTB を測定しない
- この理解をもとに openSUSE の sdbootutil に `DEVICETREE_SOURCE` を追加し、ディストリビューションの DTB がカーネル更新に追従して起動に使われるようにした

[^1]: [edk2-rk3588 の FDT 処理 (FdtPlatformDxe.c)](https://github.com/edk2-porting/edk2-rk3588/blob/master/edk2-rockchip/Silicon/Rockchip/RK3588/Drivers/FdtPlatformDxe/FdtPlatformDxe.c)
[^2]: [U-Boot の EFI ブートメソッド (boot/bootmeth_efi.c)](https://github.com/u-boot/u-boot/blob/master/boot/bootmeth_efi.c)
[^3]: [edk2-rk3588: Device Tree configuration](https://github.com/edk2-porting/edk2-rk3588#device-tree-configuration)
[^4]: [Linux EFI スタブの DTB 処理 (drivers/firmware/efi/libstub/fdt.c)](https://github.com/torvalds/linux/blob/master/drivers/firmware/efi/libstub/fdt.c)
[^5]: [Boot Loader Specification](https://uapi-group.org/specifications/specs/boot_loader_specification/)
[^6]: [GRUB の devicetree コマンド (grub-core/loader/efi/fdt.c)](https://git.savannah.gnu.org/cgit/grub.git/tree/grub-core/loader/efi/fdt.c)
[^7]: [systemd-stub の UKI セクション処理 (src/boot/stub.c)](https://github.com/systemd/systemd/blob/main/src/boot/stub.c)
[^8]: [GRUB Manual: Measured Boot](https://www.gnu.org/software/grub/manual/grub/html_node/Measured-Boot.html)
[^9]: [arm64: dts: rockchip: Enable the NPU on rk3588-rock-5-itx](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e4f7054e819eece6fd83072ff2dcefc7a36224c0)
