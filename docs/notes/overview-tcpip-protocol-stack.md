# TCP/IP プロトコルスタックの機能概要

## TCP/IP プロトコルスタックとはなにか？

TCP/IP プロトコルスタックは、インターネット上での通信を可能にする一連のネットワークプロトコルの実装です。Linux カーネルでは、TCP/IP スタックは階層化されたアーキテクチャとして実装されており、各層が特定の役割を担います。

### 4層モデルの構成

1. **アプリケーション層**: HTTP、FTP、SSH などのプロトコル
2. **トランスポート層**: TCP、UDP プロトコル
3. **インターネット層**: IP プロトコル（IPv4、IPv6）
4. **データリンク層**: Ethernet、WiFi などのネットワークインターフェース

**関連ソースコード:**
- `net/` - TCP/IP スタック全体の実装
- `include/net/` - ネットワーク関連のヘッダファイル

## TCP/IP プロトコルスタックの OS 内での位置づけ

Linux カーネル内で TCP/IP スタックは、ユーザー空間のアプリケーションとネットワークハードウェア間の橋渡しを行います。

```
ユーザー空間アプリケーション
        ↓ (システムコール)
カーネル空間 TCP/IP スタック
        ↓
ネットワークデバイスドライバ
        ↓
ネットワークハードウェア (NIC)
```

**関連ソースコード:**
- `net/socket.c` - ソケット API の実装
- `net/core/sock.c` - ソケット構造体の管理
- `include/linux/socket.h` - ソケット関連の定義

## TCP/IP という用語の整理：TCP/IP プロトコル

TCP/IP は以下の主要プロトコルで構成されます：

### IP (Internet Protocol)
- パケットのルーティングとアドレッシングを担当
- IPv4 と IPv6 をサポート

**関連ソースコード:**
- `net/ipv4/` - IPv4 実装
- `net/ipv6/` - IPv6 実装
- `net/ipv4/ip_input.c` - IPv4 パケット受信処理
- `net/ipv4/ip_output.c` - IPv4 パケット送信処理

### TCP (Transmission Control Protocol)
- 信頼性のあるコネクション指向通信
- フロー制御、輻輳制御、エラー回復

**関連ソースコード:**
- `net/ipv4/tcp.c` - TCP の主要実装
- `net/ipv4/tcp_input.c` - TCP パケット入力処理
- `net/ipv4/tcp_output.c` - TCP パケット出力処理
- `net/ipv4/tcp_timer.c` - TCP タイマー管理

### UDP (User Datagram Protocol)
- シンプルなコネクションレス通信

**関連ソースコード:**
- `net/ipv4/udp.c` - UDP 実装

## TCP/IP プロトコルスタックによる、HTTP over TCP の処理の流れ

### 2台のホストでの通信流れ

#### Host 1: 送信側の処理
1. **アプリケーション層**
   - HTTP リクエストメッセージを作成
   - `write()` システムコールでソケットに送信

2. **トランスポート層 (TCP)**
   - メッセージを TCP セグメントに分割
   - シーケンス番号、チェックサムを付加
   - TCP ヘッダーを追加

3. **インターネット層 (IP)**
   - TCP セグメントを IP パケットにカプセル化
   - ルーティングテーブルを参照して次のホップを決定
   - IP ヘッダーを追加

4. **データリンク層**
   - IP パケットをフレームにカプセル化
   - Ethernet ヘッダーを追加してネットワークに送信

**関連ソースコード:**
- `net/socket.c:sock_write_iter()` - ソケット書き込み
- `net/ipv4/tcp.c:tcp_sendmsg()` - TCP メッセージ送信
- `net/ipv4/ip_output.c:ip_queue_xmit()` - IP パケット送信キュー
- `net/core/dev.c:dev_queue_xmit()` - デバイス送信キュー

#### Host 2: 受信側の処理
1. **データリンク層**
   - NIC がフレームを受信
   - Ethernet ヘッダーを除去

2. **インターネット層 (IP)**
   - IP ヘッダーをチェック
   - 宛先アドレスが自身かチェック
   - フラグメントの再組み立て（必要に応じて）

3. **トランスポート層 (TCP)**
   - TCP ヘッダーをチェック
   - セグメントの順序付けと再組み立て
   - ACK パケットの送信

4. **アプリケーション層**
   - 完成したメッセージをアプリケーションに配信
   - `read()` システムコールでデータを読み込み

**関連ソースコード:**
- `drivers/net/*/` - NIC ドライバ
- `net/core/dev.c:netif_receive_skb()` - パケット受信処理
- `net/ipv4/ip_input.c:ip_rcv()` - IP パケット受信
- `net/ipv4/tcp_input.c:tcp_v4_rcv()` - TCP パケット受信

## アプリケーション実行中に NIC によるハードウェア割り込みが入ったときの処理の流れ

### 割り込み処理の流れ
1. **ハードウェア割り込み発生**
   - NIC がパケット受信を検出
   - IRQ (Interrupt Request) を CPU に送信

2. **割り込みハンドラ実行**
   - カーネルが割り込みハンドラを呼び出し
   - 最小限の処理を実行（パケットを受信バッファに移動）

3. **ソフト IRQ スケジューリング**
   - より重い処理をソフト IRQ に委譲
   - `NET_RX_SOFTIRQ` を発動

4. **ネットワークスタック処理**
   - プロトコル別の処理を実行
   - アプリケーションのソケットバッファに配信

**関連ソースコード:**
- `drivers/net/ethernet/*/` - NIC ドライバの割り込みハンドラ
- `net/core/dev.c:net_rx_action()` - ネットワーク受信ソフト IRQ
- `kernel/softirq.c` - ソフト IRQ 機構
- `net/core/dev.c:process_backlog()` - パケット処理バックログ

### NAPI (New API) による効率化
Linux は NAPI (New API) を使用して割り込みの頻度を制御：

- **ポーリングモード**: 高負荷時に割り込みを無効化してポーリングに切り替え
- **Weight システム**: 一度の処理で処理するパケット数を制限

**関連ソースコード:**
- `include/linux/netdevice.h` - NAPI 構造体定義
- `net/core/dev.c:napi_poll()` - NAPI ポーリング処理

## 各レイヤー間のインタフェースの処理の流れ、ソースコードの整理

### アプリケーション層とトランスポート層
**インターフェース**: システムコール API（socket、bind、listen、accept、send、recv など）

**主要な処理**:
- ソケットの作成と管理
- データの送受信バッファリング
- ブロッキング/ノンブロッキング I/O

**関連ソースコード:**
```c
// ソケット API の実装
net/socket.c
  ├── sys_socket()      // socket() システムコール
  ├── sys_bind()        // bind() システムコール  
  ├── sys_listen()      // listen() システムコール
  ├── sys_accept()      // accept() システムコール
  └── sock_read_iter()  // read/recv システムコール

// ソケット構造体管理
net/core/sock.c
  ├── sk_alloc()        // ソケット割り当て
  ├── sock_init_data()  // ソケット初期化
  └── sk_free()         // ソケット解放
```

### トランスポート層とインターネット層
**インターフェース**: `ip_queue_xmit()`, `ip_local_deliver()` 関数群

**主要な処理**:
- TCP/UDP ヘッダーの付加/除去
- IP パケットへのカプセル化/デカプセル化
- ポート番号によるソケットの特定

**関連ソースコード:**
```c
// TCP から IP への送信
net/ipv4/ip_output.c
  └── ip_queue_xmit()   // TCP から呼び出される IP 送信関数

// IP から TCP への配信
net/ipv4/ip_input.c
  ├── ip_local_deliver()     // ローカル配信処理
  └── ip_local_deliver_finish()  // プロトコル別ハンドラ呼び出し

// プロトコル登録テーブル
net/ipv4/protocol.c
  └── inet_protos[]     // プロトコル番号とハンドラの対応表
```

### インターネット層とデータリンク層
**インターフェース**: `dev_queue_xmit()`, `netif_receive_skb()` 関数群

**主要な処理**:
- ルーティングテーブルの参照
- ARP による MAC アドレス解決
- Ethernet フレームのカプセル化/デカプセル化

**関連ソースコード:**
```c
// ルーティング処理
net/ipv4/route.c
  ├── ip_route_output_key()  // 送信用ルート検索
  └── ip_route_input()       // 受信用ルート検索

// ARP (Address Resolution Protocol)
net/ipv4/arp.c
  ├── arp_send()        // ARP 要求送信
  └── arp_process()     // ARP 応答処理

// デバイス層との接続
net/core/dev.c
  ├── dev_queue_xmit()       // デバイスへの送信
  ├── netif_receive_skb()    // デバイスからの受信
  └── __netif_receive_skb_core()  // 受信コア処理
```

### データリンク層と物理層
**インターフェース**: ネットワークデバイスドライバ API

**主要な処理**:
- DMA によるデータ転送
- 割り込み処理
- フレームの送受信

**関連ソースコード:**
```c
// ネットワークデバイス構造体
include/linux/netdevice.h
  └── struct net_device    // ネットワークデバイス構造体

// デバイスドライバインターフェース
drivers/net/ethernet/intel/e1000e/  // Intel Ethernet ドライバ例
  ├── e1000_main.c
  │   ├── e1000_xmit_frame()    // 送信関数
  │   └── e1000_intr()          // 割り込みハンドラ
  └── e1000_hw.c                // ハードウェア制御

// 一般的なドライバフレームワーク
net/core/dev.c
  ├── register_netdev()    // デバイス登録
  └── unregister_netdev()  // デバイス登録解除
```

## TCP/IP プロトコルスタックを実装するときのポイント

### 1. 効率的なバッファ管理
- **sk_buff 構造体**: Linux の高効率パケットバッファ
- **ゼロコピー**: メモリコピーを最小限に抑制
- **バッファ共有**: 複数の処理で同一バッファを共有

**関連ソースコード:**
```c
include/linux/skbuff.h
  └── struct sk_buff      // パケットバッファ構造体

net/core/skbuff.c
  ├── alloc_skb()         // バッファ割り当て
  ├── skb_clone()         // バッファ複製
  └── kfree_skb()         // バッファ解放
```

### 2. 並行処理とロッキング
- **Per-CPU データ構造**: CPU ごとのデータでロック競合を回避
- **RCU (Read-Copy-Update)**: 読み込み性能を重視したロックメカニズム
- **ファインラッシュドロック**: 粒度の細かいロック制御

**関連ソースコード:**
```c
// RCU 機構
include/linux/rcupdate.h
net/core/dev.c
  └── rcu_read_lock()     // RCU 読み込みロック

// Per-CPU 統計
include/linux/percpu.h
net/ipv4/tcp.c
  └── __TCP_INC_STATS()   // Per-CPU 統計更新
```

### 3. メモリ効率とキャッシュ最適化
- **オブジェクトキャッシュ**: slab/slub アロケータの活用
- **ローカリティ**: データアクセスパターンの最適化

**関連ソースコード:**
```c
net/core/sock.c
  └── sock_init()         // ソケットキャッシュ初期化

net/ipv4/tcp.c
  └── tcp_init()          // TCP キャッシュ初期化
```

### 4. エラーハンドリングと回復
- **チェックサム**: データ整合性の検証
- **再送制御**: パケット紛失の検出と回復
- **輻輳制御**: ネットワーク輻輳の回避

**関連ソースコード:**
```c
net/ipv4/tcp_input.c
  ├── tcp_ack()           // ACK 処理
  └── tcp_retransmit_timer()  // 再送タイマー

net/ipv4/tcp_cong.c
  └── tcp_cong_control()  // 輻輳制御
```

## Linux カーネルの TCP/IP プロトコルスタックを弄ってデバッグするにはどうしたらよいか？

### 1. カーネルデバッグ設定
**カーネル設定オプション:**
```bash
CONFIG_DEBUG_KERNEL=y
CONFIG_NET_DEBUG=y
CONFIG_DEBUG_NET=y
CONFIG_DYNAMIC_DEBUG=y
```

### 2. デバッグツールとトレーシング

#### ftrace によるカーネルトレーシング
```bash
# TCP 関数のトレーシング
echo 'tcp_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

**関連ソースコード:**
```c
// トレーシングポイントの定義
include/trace/events/tcp.h
include/trace/events/net.h
```

#### perf によるパフォーマンス解析
```bash
# TCP 関連イベントの監視
perf record -e net:netif_rx -e net:net_dev_xmit
perf report
```

#### eBPF によるプログラマブルトレーシング
```c
// BPF プログラム例: TCP パケット監視
SEC("kprobe/tcp_sendmsg")
int trace_tcp_send(struct pt_regs *ctx) {
    // パケット情報をログ出力
    return 0;
}
```

### 3. ネットワーク統計の確認
```bash
# 詳細な TCP 統計
cat /proc/net/tcp
cat /proc/net/netstat
cat /proc/net/snmp

# ソケット統計
ss -s
```

**関連ソースコード:**
```c
net/ipv4/proc.c
  ├── tcp4_seq_show()     // /proc/net/tcp の実装
  └── netstat_seq_show()  // /proc/net/netstat の実装
```

### 4. パケットキャプチャとダンプ
```bash
# tcpdump でカーネルレベルでのパケット監視
tcpdump -i any -XX

# Wireshark での詳細解析
wireshark &
```

### 5. カーネルモジュールによる独自デバッグ
**サンプル debug モジュール:**
```c
// tcp_debug.c
#include <linux/module.h>
#include <linux/net.h>
#include <net/tcp.h>

// TCP パケット送信時のフック
static int tcp_send_hook(struct sock *sk, struct msghdr *msg, size_t size) {
    printk(KERN_INFO "TCP send: size=%zu\n", size);
    return 0;  // 正常処理を継続
}

static int __init tcp_debug_init(void) {
    // フックの登録
    return 0;
}

static void __exit tcp_debug_exit(void) {
    // フックの解除
}

module_init(tcp_debug_init);
module_exit(tcp_debug_exit);
```

### 6. 開発環境の準備
```bash
# カーネル開発環境
make menuconfig
make -j$(nproc)
make modules_install
make install

# QEMU での仮想マシンテスト
qemu-kvm -kernel arch/x86/boot/bzImage -initrd initramfs.cpio.gz
```

### 7. よくあるデバッグポイント

#### パケット紛失の調査
**関連ソースコード場所:**
```c
net/core/dev.c:enqueue_to_backlog()  // 受信バッファオーバーフロー
net/ipv4/tcp_input.c:tcp_validate_incoming()  // 不正パケット検出
net/ipv4/ip_forward.c:ip_forward()   // ルーティング問題
```

#### 性能問題の調査
**監視ポイント:**
```c
net/ipv4/tcp_output.c:tcp_write_xmit()  // 送信ボトルネック
net/core/dev.c:net_rx_action()          // 受信処理性能
mm/page_alloc.c:__alloc_pages_nodemask()  // メモリ割り当て問題
```

### 8. 実践的なデバッグワークフロー
1. **問題の特定**: ログ、統計、パケットキャプチャで現象を確認
2. **仮説立案**: 問題の原因を推測
3. **コード調査**: 関連するソースコードを読み解く
4. **トレーシング**: ftrace、perf、eBPF で動作を追跡
5. **修正実装**: パッチの作成とテスト
6. **回帰テスト**: 修正が他の機能に影響しないか確認

このような手法を組み合わせることで、Linux カーネルの TCP/IP スタックの動作を詳細に理解し、効果的にデバッグすることができます。