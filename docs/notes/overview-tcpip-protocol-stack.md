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

#### IP パケット受信処理の実装解析

`ip_rcv()` 関数は IPv4 パケット受信処理の中核です：

```c
// net/ipv4/ip_input.c
int ip_rcv(struct sk_buff *skb, struct net_device *dev, 
           struct packet_type *pt, struct net_device *orig_dev)
{
    const struct iphdr *iph;
    u32 len;

    /* パケットがIPパケットかチェック */
    if (skb->pkt_type == PACKET_OTHERHOST)
        goto drop;

    IP_UPD_PO_STATS_BH(dev_net(dev), IPSTATS_MIB_IN, skb->len);

    skb = skb_share_check(skb, GFP_ATOMIC);
    if (!skb) {
        IP_INC_STATS_BH(dev_net(dev), IPSTATS_MIB_INDISCARDS);
        goto out;
    }

    if (!pskb_may_pull(skb, sizeof(struct iphdr)))
        goto inhdr_error;

    iph = ip_hdr(skb);

    /* IPヘッダーの基本チェック */
    if (iph->ihl < 5 || iph->version != 4)
        goto inhdr_error;

    BUILD_BUG_ON(IPSTATS_MIB_ECT1PKTS != IPSTATS_MIB_NOECTPKTS + INET_ECN_ECT_1);
    BUILD_BUG_ON(IPSTATS_MIB_ECT0PKTS != IPSTATS_MIB_NOECTPKTS + INET_ECN_ECT_0);
    BUILD_BUG_ON(IPSTATS_MIB_CEPKTS != IPSTATS_MIB_NOECTPKTS + INET_ECN_CE);
    IP_ADD_STATS_BH(dev_net(dev),
                    IPSTATS_MIB_NOECTPKTS + (iph->tos & INET_ECN_MASK),
                    max_t(unsigned short, 1, skb_shinfo(skb)->gso_segs));

    /* チェックサム検証とパケット長の確認 */
    if (!pskb_may_pull(skb, iph->ihl*4))
        goto inhdr_error;

    iph = ip_hdr(skb);

    if (unlikely(ip_fast_csum((u8*)iph, iph->ihl)))
        goto csum_error;

    len = ntohs(iph->tot_len);
    if (skb->len < len) {
        IP_INC_STATS_BH(dev_net(dev), IPSTATS_MIB_INTRUNCATEDPKTS);
        goto drop;
    } else if (len < (iph->ihl*4))
        goto inhdr_error;

    /* パケットのトリミング */
    if (pskb_trim_rcsum(skb, len)) {
        IP_INC_STATS_BH(dev_net(dev), IPSTATS_MIB_INDISCARDS);
        goto drop;
    }

    skb->transport_header = skb->network_header + iph->ihl*4;

    /* NetFilter フックポイント */
    return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, skb, dev, NULL,
                   ip_rcv_finish);

csum_error:
    IP_INC_STATS_BH(dev_net(dev), IPSTATS_MIB_CSUMERRORS);
inhdr_error:
    IP_INC_STATS_BH(dev_net(dev), IPSTATS_MIB_INHDRERRORS);
drop:
    kfree_skb(skb);
out:
    return NET_RX_DROP;
}
```

**解析ポイント:**
1. **パケット種別の確認**: `skb->pkt_type` で自分宛のパケットかチェック
2. **IPヘッダー検証**: バージョン（4）とヘッダー長の最小サイズをチェック
3. **チェックサム検証**: `ip_fast_csum()` でヘッダーの整合性を確認
4. **パケット長チェック**: `tot_len` フィールドと実際のデータ長を比較
5. **NetFilter統合**: `NF_HOOK` でファイアウォール処理に連携

### TCP (Transmission Control Protocol)
- 信頼性のあるコネクション指向通信
- フロー制御、輻輳制御、エラー回復

**関連ソースコード:**
- `net/ipv4/tcp.c` - TCP の主要実装
- `net/ipv4/tcp_input.c` - TCP パケット入力処理
- `net/ipv4/tcp_output.c` - TCP パケット出力処理
- `net/ipv4/tcp_timer.c` - TCP タイマー管理

#### TCP ソケット構造体の解析

TCP の核となる `tcp_sock` 構造体：

```c
// include/linux/tcp.h
struct tcp_sock {
    /* inet_connection_sock 構造体を継承 */
    struct inet_connection_sock inet_conn;

    /* TCP 固有の制御ブロック */
    u16 tcp_header_len;     /* TCP ヘッダー長 (オプション含む) */
    u16 gso_segs;          /* GSO セグメント数 */
    
    /* 送信制御 */
    u32 snd_nxt;           /* 次に送信するシーケンス番号 */
    u32 snd_una;           /* 未確認の最小シーケンス番号 */
    u32 snd_sml;           /* 最後に送信した小さなパケットのシーケンス番号 */
    u32 rcv_tstamp;        /* 最後にデータを受信した時刻 */
    u32 lsndtime;          /* 最後にデータを送信した時刻 */

    /* 受信制御 */
    u32 rcv_nxt;           /* 次に受信すべきシーケンス番号 */
    u32 copied_seq;        /* ユーザー空間にコピー済みのシーケンス番号 */
    u32 rcv_wup;           /* ウィンドウ更新送信時のrcv_nxt */
    u32 snd_wnd;           /* 送信ウィンドウサイズ */
    u32 max_window;        /* 最大ウィンドウサイズ */
    u32 mss_cache;         /* キャッシュされたMSS */

    /* フロー制御 */
    u32 window_clamp;      /* ウィンドウサイズの上限 */
    u32 rcv_ssthresh;      /* 受信スローリスト閾値 */

    /* 輻輳制御 */
    u32 snd_cwnd;          /* 輻輳ウィンドウサイズ */
    u32 snd_ssthresh;      /* スローリスト閾値 */
    u32 prior_cwnd;        /* 前回の輻輳ウィンドウ */
    u32 prr_delivered;     /* PRR: 配信済みセグメント数 */
    u32 prr_out;           /* PRR: 送信済みセグメント数 */

    /* RTT 測定 */
    u32 srtt_us;           /* 平滑化RTT (μ秒) */
    u32 mdev_us;           /* RTT偏差 (μ秒) */
    u32 mdev_max_us;       /* 最大RTT偏差 */
    u32 rttvar_us;         /* RTT分散 */
    u32 rtt_seq;           /* RTT測定対象のシーケンス番号 */

    /* 再送制御 */
    struct sk_buff_head out_of_order_queue; /* 順序外パケット待ち */
    struct tcp_sack_block duplicate_sack[1]; /* D-SACK情報 */
    struct tcp_sack_block selective_acks[4]; /* SACK情報 */
    
    /* 高速再送/高速復旧 */
    u8 reordering;         /* パケット並び替え耐性 */
    u8 keepalive_probes;   /* キープアライブ試行回数 */
    
    /* タイマー */
    struct timer_list retransmit_timer; /* 再送タイマー */
    struct timer_list delack_timer;     /* 遅延ACKタイマー */
    
    /* その他のTCP拡張機能 */
    u8 ecn_flags;          /* ECN フラグ */
    u8 syn_retries;        /* SYN 再送回数 */
};
```

**解析ポイント:**
1. **シーケンス番号管理**: `snd_nxt`, `rcv_nxt` でデータの順序制御
2. **ウィンドウ制御**: `snd_wnd`, `rcv_wnd` でフロー制御実現
3. **輻輳制御**: `snd_cwnd`, `snd_ssthresh` で帯域制御
4. **RTT測定**: `srtt_us`, `mdev_us` で応答時間監視と再送間隔決定
5. **SACK**: `selective_acks` で選択的確認応答による効率的な再送

#### TCP 送信処理の実装解析

`tcp_sendmsg()` 関数はアプリケーションからのデータ送信要求を処理：

```c
// net/ipv4/tcp.c
int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *skb;
    struct sockcm_cookie sockc;
    int flags, err, copied = 0;
    int mss_now = 0, size_goal;
    bool process_backlog = false;
    bool sg;
    long timeo;

    lock_sock(sk);

    flags = msg->msg_flags;
    if (flags & MSG_FASTOPEN) {
        err = tcp_sendmsg_fastopen(sk, msg, &copied, size);
        if (err == -EINPROGRESS && copied > 0)
            goto out;
        if (err)
            goto out_err;
    }

    timeo = sock_sndtimeo(sk, flags & MSG_DONTWAIT);

    tcp_rate_check_app_limited(sk);  /* 帯域制限チェック */

    /* ソケット状態の確認 */
    if (((1 << sk->sk_state) & ~(TCPF_ESTABLISHED | TCPF_CLOSE_WAIT)) &&
        !tcp_passive_fastopen(sk)) {
        err = sk_stream_wait_connect(sk, &timeo);
        if (err != 0)
            goto do_error;
    }

    if (unlikely(tp->repair)) {
        if (tp->repair_queue == TCP_RECV_QUEUE) {
            copied = tcp_send_rcvq(sk, msg, size);
            goto out_nopush;
        }

        err = -EINVAL;
        if (tp->repair_queue == TCP_NO_QUEUE)
            goto out_err;
    }

    mss_now = tcp_send_mss(sk, &size_goal, flags);

    /* メインの送信ループ */
    while (msg_data_left(msg)) {
        int copy = 0;
        int max = size_goal;

        skb = tcp_write_queue_tail(sk);
        if (tcp_send_head(sk)) {
            if (skb->ip_summed == CHECKSUM_NONE)
                max = mss_now;
            copy = max - skb->len;
        }

        if (copy <= 0 || !tcp_skb_can_collapse_to(skb)) {
new_segment:
            /* 新しいセグメント作成 */
            if (!sk_stream_memory_free(sk))
                goto wait_for_sndbuf;

            skb = sk_stream_alloc_skb(sk, 
                                    select_size(sk, sg),
                                    sk->sk_allocation,
                                    skb_queue_empty(&sk->sk_write_queue));
            if (!skb)
                goto wait_for_memory;

            process_backlog = true;
            skb_entail(sk, skb);
            copy = size_goal;
            max = size_goal;
        }

        /* データのコピー */
        copy = min_t(int, copy, msg_data_left(msg));

        /* ユーザー空間からカーネル空間へデータコピー */
        err = skb_add_data_nocache(sk, skb, &msg->msg_iter, copy);
        if (err)
            goto do_fault;

        if (!copied)
            TCP_SKB_CB(skb)->tcp_flags &= ~TCPHDR_PSH;

        tp->write_seq += copy;
        TCP_SKB_CB(skb)->end_seq += copy;
        tcp_skb_pcount_set(skb, 0);

        copied += copy;
        if (!msg_data_left(msg)) {
            if (unlikely(flags & MSG_EOR))
                TCP_SKB_CB(skb)->eor = 1;
            goto out;
        }

        if (skb->len < max || (flags & MSG_OOB) || unlikely(tp->repair))
            continue;

        if (forced_push(tp)) {
            tcp_mark_push(tp, skb);
            __tcp_push_pending_frames(sk, mss_now, TCP_NAGLE_PUSH);
        } else if (skb == tcp_send_head(sk))
            tcp_push_one(sk, mss_now);
        continue;

wait_for_sndbuf:
        set_bit(SOCK_NOSPACE, &sk->sk_socket->flags);
wait_for_memory:
        if (copied)
            tcp_push(sk, flags & ~MSG_MORE, mss_now,
                     TCP_NAGLE_PUSH, size_goal);

        err = sk_stream_wait_memory(sk, &timeo);
        if (err != 0)
            goto do_error;

        mss_now = tcp_send_mss(sk, &size_goal, flags);
    }

out:
    if (copied && !(flags & MSG_SENDPAGE_NOTLAST))
        tcp_push(sk, flags, mss_now, tp->nonagle, size_goal);

out_nopush:
    release_sock(sk);
    return copied + copied_syn;

do_fault:
    if (!skb->len) {
        tcp_unlink_write_queue(skb, sk);
        sk_wmem_free_skb(sk, skb);
    }

do_error:
    if (copied + copied_syn)
        goto out;
out_err:
    err = sk_stream_error(sk, flags, err);
    release_sock(sk);
    return err;
}
```

**解析ポイント:**
1. **接続状態確認**: `sk_state` でTCP接続の状態をチェック
2. **MSS計算**: `tcp_send_mss()` で最適なセグメントサイズを決定
3. **セグメント作成**: `sk_stream_alloc_skb()` で新しいskb構造体を割り当て
4. **データコピー**: `skb_add_data_nocache()` でユーザーデータをコピー
5. **送信処理**: `tcp_push()` で実際のパケット送信を実行
6. **フロー制御**: `sk_stream_wait_memory()` で送信バッファが満杯時に待機

### UDP (User Datagram Protocol)
- シンプルなコネクションレス通信

**関連ソースコード:**
- `net/ipv4/udp.c` - UDP 実装

#### UDP 送信処理の実装解析

TCP と比較して非常にシンプルな UDP 送信処理：

```c
// net/ipv4/udp.c  
int udp_sendmsg(struct sock *sk, struct msghdr *msg, size_t len)
{
    struct inet_sock *inet = inet_sk(sk);
    struct udp_sock *up = udp_sk(sk);
    DECLARE_SOCKADDR(struct sockaddr_in *, usin, msg->msg_name);
    struct flowi4 fl4_stack;
    struct flowi4 *fl4;
    int ulen = len;
    struct ipcm_cookie ipc;
    struct rtable *rt = NULL;
    int free = 0;
    int connected = 0;
    __be32 daddr, faddr, saddr;
    __be16 dport;
    u8  tos;
    int err, is_udplite = IS_UDPLITE(sk);
    int corkreq = up->corkflag || msg->msg_flags&MSG_MORE;
    int (*getfrag)(void *, char *, int, int, int, struct sk_buff *);
    struct sk_buff *skb;
    struct ip_options_data opt_copy;

    if (len > 0xFFFF)
        return -EMSGSIZE;

    /* 宛先アドレス決定 */
    if (msg->msg_name) {
        struct sockaddr_in *usin = (struct sockaddr_in *)msg->msg_name;
        if (msg->msg_namelen < sizeof(*usin))
            return -EINVAL;
        if (usin->sin_family != AF_INET) {
            if (usin->sin_family != AF_UNSPEC)
                return -EAFNOSUPPORT;
        }

        daddr = usin->sin_addr.s_addr;
        dport = usin->sin_port;
        if (dport == 0)
            return -EINVAL;
    } else {
        if (sk->sk_state != TCP_ESTABLISHED)
            return -EDESTADDRREQ;
        daddr = inet->inet_daddr;
        dport = inet->inet_dport;
        connected = 1;
    }

    /* 送信元アドレス決定 */  
    ipc.addr = inet->inet_saddr;
    ipc.oif = sk->sk_bound_dev_if;

    if (msg->msg_controllen) {
        err = ip_cmsg_send(sock_net(sk), msg, &ipc,
                          sk->sk_family == AF_INET6);
        if (unlikely(err)) {
            kfree(opt_copy.opt.optlen ? &opt_copy : NULL);
            return err;
        }
        if (ipc.opt)
            free = 1;
        connected = 0;
    }

    saddr = ipc.addr;
    ipc.addr = faddr = daddr;

    /* ルーティング */
    if (connected && !corkreq) {
        rt = (struct rtable *)sk_dst_check(sk, 0);
        if (rt) {
            if (rt->rt_route_iif != LOOPBACK_IFINDEX &&
                rt->rt_route_iif != dev_net(rt->dst.dev)->loopback_dev->ifindex) {
                dst_release(&rt->dst);
                rt = NULL;
            }
        }
    }

    if (!rt) {
        struct net *net = sock_net(sk);

        fl4 = &fl4_stack;
        flowi4_init_output(fl4, ipc.oif, sk->sk_mark, tos,
                          RT_SCOPE_UNIVERSE, sk->sk_protocol,
                          inet_sk_flowi_flags(sk),
                          faddr, saddr, dport, inet->inet_sport);

        security_sk_classify_flow(sk, flowi4_to_flowi(fl4));
        rt = ip_route_output_flow(net, fl4, sk);
        if (IS_ERR(rt)) {
            err = PTR_ERR(rt);
            rt = NULL;
            if (err == -ENETUNREACH)
                IP_INC_STATS(net, IPSTATS_MIB_OUTNOROUTES);
            goto out;
        }

        err = -EACCES;
        if ((rt->rt_flags & RTCF_BROADCAST) &&
            !sock_flag(sk, SOCK_BROADCAST))
            goto out;
        if (connected)
            sk_dst_set(sk, dst_clone(&rt->dst));
    }

    if (msg->msg_flags&MSG_CONFIRM)
        goto do_confirm;
back_from_confirm:

    saddr = fl4->saddr;
    if (!ipc.addr)
        daddr = ipc.addr = fl4->daddr;

    /* フラグメント処理 */
    lock_sock(sk);
    if (up->pending) {
        /*
         * There are pending frames.
         * The socket lock must be held while it's corked.
         */
        fl4 = &inet->cork.fl.u.ip4;
        goto do_append_data;
    }
    up->len += ulen;
    err = ip_append_data(sk, fl4, getfrag, msg,
                        ulen, sizeof(struct udphdr), &ipc, &rt,
                        corkreq ? msg->msg_flags|MSG_MORE : msg->msg_flags);
    if (err)
        udp_flush_pending_frames(sk);
    else if (!corkreq)
        err = udp_push_pending_frames(sk);
    else if (unlikely(skb_queue_empty(&sk->sk_write_queue)))
        up->pending = 0;
    release_sock(sk);

out:
    ip_rt_put(rt);
    if (free)
        kfree(opt_copy.opt.optlen ? &opt_copy : NULL);
    if (!err)
        return len;
    return err;

do_confirm:
    dst_confirm(&rt->dst);
    if (!(msg->msg_flags&MSG_PROBE) || len)
        goto back_from_confirm;
    err = 0;
    goto out;
}
```

**解析ポイント:**
1. **宛先決定**: `msg->msg_name` から宛先アドレスとポートを取得
2. **ルーティング**: `ip_route_output_flow()` で送信経路を決定
3. **ブロードキャスト制御**: `RTCF_BROADCAST` フラグでブロードキャスト送信の権限チェック
4. **フラグメント処理**: `ip_append_data()` で大きなデータグラムの分割
5. **送信実行**: `udp_push_pending_frames()` で実際のパケット送信
6. **接続型UDP**: 一度connect()されたソケットは宛先情報を再利用

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