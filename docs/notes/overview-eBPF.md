# eBPF の概要

## eBPF とはなにか？

eBPF (Extended Berkeley Packet Filter) は、Linux カーネル内で安全にユーザー定義コードを実行できる仮想マシン技術です。元々はパケットフィルタリング用の BPF (Berkeley Packet Filter) から発展した技術で、現在はネットワーク、トレーシング、セキュリティなど幅広い用途で使用されています。

### eBPF の主な特徴

1. **安全性**: カーネル内で実行されるが、検証機構により安全性が保証される
2. **高性能**: カーネル空間で直接実行されるため、ユーザー空間とのコンテキストスイッチが不要
3. **プログラマビリティ**: C言語ライクな言語でカスタムロジックを記述可能
4. **動的性**: システム再起動なしに動的にプログラムをロード・アンロード可能

**関連ソースコード:**
- `kernel/bpf/` - eBPF コア実装
- `include/linux/bpf.h` - eBPF 関連定義
- `include/uapi/linux/bpf.h` - eBPF ユーザー空間 API

## eBPF の OS 内での位置づけ

eBPF は Linux カーネル内に組み込まれた仮想マシンとして動作し、カーネルの各サブシステムにフックポイントを提供します。

```
ユーザー空間
├── eBPF プログラム (.c)
├── eBPF ローダー (libbpf)
└── eBPF ツール (bpftrace, bcc)
        ↓ (bpf() システムコール)
カーネル空間
├── eBPF 検証器 (Verifier)
├── eBPF JIT コンパイラ
├── eBPF Maps (データ共有)
└── eBPF プログラム実行
        ↓ (フック)
各カーネルサブシステム
├── ネットワークスタック
├── スケジューラ
├── ファイルシステム
└── システムコール
```

**関連ソースコード:**
```c
// eBPF システムコール実装
kernel/bpf/syscall.c
  ├── SYSCALL_DEFINE3(bpf)      // bpf() システムコール
  ├── bpf_prog_load()           // プログラムロード
  └── bpf_map_create()          // Map 作成

// eBPF 検証器
kernel/bpf/verifier.c
  ├── bpf_check()               // プログラム検証メイン
  ├── check_cfg()               // 制御フロー検証
  └── check_mem_access()        // メモリアクセス検証
```

#### eBPF システムコールの実装解析

`bpf()` システムコールは eBPF の全ての操作を担当する統一エントリーポイントです：

```c
// kernel/bpf/syscall.c
SYSCALL_DEFINE3(bpf, int, cmd, union bpf_attr __user *, uattr, unsigned int, size)
{
    union bpf_attr attr = {};
    int err;

    if (sysctl_unprivileged_bpf_disabled && !bpf_capable())
        return -EPERM;

    err = bpf_check_uarg_tail_zero(uattr, sizeof(attr), size);
    if (err)
        return err;
    size = min_t(u32, size, sizeof(attr));

    /* ユーザー空間から属性をコピー */
    if (copy_from_user(&attr, uattr, size) != 0)
        return -EFAULT;

    err = security_bpf(cmd, &attr, size);
    if (err < 0)
        return err;

    switch (cmd) {
    case BPF_MAP_CREATE:
        err = map_create(&attr);
        break;
    case BPF_MAP_LOOKUP_ELEM:
        err = map_lookup_elem(&attr);
        break;
    case BPF_MAP_UPDATE_ELEM:
        err = map_update_elem(&attr);
        break;
    case BPF_MAP_DELETE_ELEM:
        err = map_delete_elem(&attr);
        break;
    case BPF_MAP_GET_NEXT_KEY:
        err = map_get_next_key(&attr);
        break;
    case BPF_PROG_LOAD:
        err = bpf_prog_load(&attr, uattr);
        break;
    case BPF_OBJ_PIN:
        err = bpf_obj_pin(&attr);
        break;
    case BPF_OBJ_GET:
        err = bpf_obj_get(&attr);
        break;
    case BPF_PROG_ATTACH:
        err = bpf_prog_attach(&attr);
        break;
    case BPF_PROG_DETACH:
        err = bpf_prog_detach(&attr);
        break;
    case BPF_PROG_QUERY:
        err = bpf_prog_query(&attr, uattr);
        break;
    case BPF_PROG_TEST_RUN:
        err = bpf_prog_test_run(&attr, uattr);
        break;
    case BPF_PROG_GET_NEXT_ID:
        err = bpf_prog_get_next_id(&attr);
        break;
    case BPF_MAP_GET_NEXT_ID:
        err = bpf_map_get_next_id(&attr);
        break;
    case BPF_PROG_GET_FD_BY_ID:
        err = bpf_prog_get_fd_by_id(&attr);
        break;
    case BPF_MAP_GET_FD_BY_ID:
        err = bpf_map_get_fd_by_id(&attr);
        break;
    case BPF_OBJ_GET_INFO_BY_FD:
        err = bpf_obj_get_info_by_fd(&attr, uattr);
        break;
    case BPF_RAW_TRACEPOINT_OPEN:
        err = bpf_raw_tracepoint_open(&attr);
        break;
    case BPF_BTF_LOAD:
        err = bpf_btf_load(&attr);
        break;
    case BPF_BTF_GET_FD_BY_ID:
        err = bpf_btf_get_fd_by_id(&attr);
        break;
    case BPF_TASK_FD_QUERY:
        err = bpf_task_fd_query(&attr, uattr);
        break;
    case BPF_MAP_LOOKUP_AND_DELETE_ELEM:
        err = map_lookup_and_delete_elem(&attr);
        break;
    default:
        err = -EINVAL;
        break;
    }

    return err;
}
```

**解析ポイント:**
1. **権限チェック**: `bpf_capable()` で非特権ユーザーからの実行を制御
2. **セキュリティフック**: `security_bpf()` でLSMによる追加セキュリティチェック
3. **コマンド分岐**: switch文で20以上のサブコマンドに対応
4. **ユーザー空間連携**: `copy_from_user()` で安全なデータ転送
5. **統一インターフェース**: 全てのeBPF操作を1つのシステムコールで処理

#### eBPF プログラム検証器の実装解析

`bpf_check()` 関数は eBPF プログラムの安全性を保証する検証器の核：

```c
// kernel/bpf/verifier.c
int bpf_check(struct bpf_prog **prog, union bpf_attr *attr,
              union bpf_attr __user *uattr)
{
    struct bpf_verifier_env *env;
    struct bpf_verifier_log *log;
    int i, len, ret = -EINVAL;
    bool is_priv;

    /* 検証環境の初期化 */
    env = kzalloc(sizeof(struct bpf_verifier_env), GFP_KERNEL);
    if (!env)
        return -ENOMEM;

    log = &env->log;

    env->insn_aux_data =
        vzalloc(array_size(sizeof(struct bpf_insn_aux_data),
                          (*prog)->len));
    ret = -ENOMEM;
    if (!env->insn_aux_data)
        goto err_free_env;

    for (i = 0; i < (*prog)->len; i++)
        env->insn_aux_data[i].orig_idx = i;
    env->prog = *prog;
    env->ops = bpf_verifier_ops[env->prog->type];
    is_priv = bpf_capable();

    /* ログレベル設定 */
    if (attr->log_level || attr->log_buf || attr->log_size) {
        /* ユーザーがログを要求した場合 */
        ret = -EINVAL;
        if (attr->log_buf && !attr->log_size)
            goto err_free_aux_data;
        if (attr->log_size < 128 || attr->log_size > UINT_MAX >> 2 ||
            attr->log_size & (sizeof(long) - 1))
            goto err_free_aux_data;

        log->level = attr->log_level;
        log->ubuf = (char __user *) (unsigned long) attr->log_buf;
        log->len_total = attr->log_size;

        ret = -ENOMEM;
        log->kbuf = kvmalloc(log->len_total, GFP_KERNEL);
        if (!log->kbuf)
            goto err_free_aux_data;
    }

    env->strict_alignment = !!(attr->prog_flags & BPF_F_STRICT_ALIGNMENT);
    if (!IS_ENABLED(CONFIG_HAVE_EFFICIENT_UNALIGNED_ACCESS))
        env->strict_alignment = true;
    if (attr->prog_flags & BPF_F_ANY_ALIGNMENT)
        env->strict_alignment = false;

    env->allow_ptr_leaks = bpf_allow_ptr_leaks();

    ret = replace_map_fd_with_map_ptr(env);
    if (ret < 0)
        goto skip_full_check;

    if (bpf_prog_is_dev_bound(env->prog->aux)) {
        ret = bpf_prog_offload_verifier_prep(env->prog);
        if (ret)
            goto skip_full_check;
    }

    /* 制御フローグラフ（CFG）の構築と検証 */
    ret = check_cfg(env);
    if (ret < 0)
        goto skip_full_check;

    /* メイン検証ループ */
    env->explored_states = kvcalloc(state_htab_size(env),
                                   sizeof(struct bpf_verifier_state_list *),
                                   GFP_USER);
    ret = -ENOMEM;
    if (!env->explored_states)
        goto skip_full_check;

    ret = check_subprogs(env);
    if (ret < 0)
        goto skip_full_check;

    ret = check_btf_info(env, attr, uattr);
    if (ret < 0)
        goto skip_full_check;

    ret = check_attach_btf_id(env);
    if (ret)
        goto skip_full_check;

    ret = resolve_pseudo_ldimm64(env);
    if (ret < 0)
        goto skip_full_check;

    /* 深度優先探索による命令検証 */
    ret = do_check(env);
    if (ret < 0)
        goto skip_full_check;

    ret = check_max_stack_depth(env);
    if (ret < 0)
        goto skip_full_check;

    /* JITコンパイル用の最適化 */
    ret = optimize_bpf_loop(env);
    if (ret < 0)
        goto skip_full_check;

    /* プログラム変換とパッチ適用 */
    ret = fixup_bpf_calls(env);
    if (ret < 0)
        goto skip_full_check;

    ret = fixup_kfunc_calls(env);
    if (ret < 0)
        goto skip_full_check;

    if (log->level && bpf_verifier_log_full(log))
        ret = -ENOSPC;

    if (log->level && !log->ubuf) {
        ret = -EFAULT;
        goto err_release_maps;
    }

skip_full_check:
    if (ret)
        goto err_release_maps;

    if (log->level && log->ubuf) {
        if (copy_to_user(log->ubuf, log->kbuf, log->len_used) != 0) {
            ret = -EFAULT;
            goto err_release_maps;
        }
    }

    /* プログラム統計情報の更新 */
    if (env->prog->aux->offload) {
        ret = bpf_prog_offload_finalize(env);
        if (ret)
            goto err_release_maps;
    }

    bpf_prog_select_runtime(env->prog, &ret);

    kvfree(env->explored_states);

    if (log->kbuf) {
        kvfree(log->kbuf);
        log->kbuf = NULL;
    }

err_release_maps:
    if (!env->prog->aux->used_maps)
        /* if we didn't copy map pointers into bpf_prog_info, release them */
        release_maps(env);

err_free_aux_data:
    kvfree(env->insn_aux_data);
err_free_env:
    kfree(env);
    return ret;
}
```

**解析ポイント:**
1. **環境初期化**: `bpf_verifier_env` で検証状態を管理
2. **CFG検証**: `check_cfg()` で制御フローの正当性をチェック
3. **深度優先検証**: `do_check()` で全ての実行パスを探索
4. **スタック検証**: `check_max_stack_depth()` でスタックオーバーフロー防止
5. **最適化**: `optimize_bpf_loop()` でループ最適化を適用
6. **パッチ適用**: `fixup_bpf_calls()` でヘルパー関数呼び出しを解決

## eBPF によるパケットフィルタリングの基礎

### 基本的なパケットフィルタリングの仕組み

eBPF プログラムはネットワークインターフェースやソケットにアタッチされ、パケットが通過する際に呼び出されます。プログラムはパケットの内容を解析し、以下の判定を返します：

- `XDP_PASS`: パケットを通常の処理に渡す
- `XDP_DROP`: パケットを破棄
- `XDP_REDIRECT`: パケットを他のインターフェースにリダイレクト
- `XDP_TX`: パケットを送信元に送り返す

### XDP (eXpress Data Path) による高速パケット処理

XDP は eBPF プログラムをネットワークドライバレベルで実行する仕組みです。これにより、従来のネットワークスタックを通らずに高速なパケット処理が可能になります。

**関連ソースコード:**
```c
// XDP 実装
net/core/xdp.c
  ├── xdp_do_redirect()         // XDP リダイレクト処理
  └── __xdp_map_lookup_elem()   // XDP Map 検索

// XDP フック実装例
drivers/net/ethernet/mellanox/mlx5/core/en_rx.c
  └── mlx5e_xdp_handle()        // XDP プログラム実行

// XDP プログラム例の構造
include/linux/filter.h
  └── struct xdp_md             // XDP メタデータ構造体
```

#### XDP プログラム実行の実装解析

ネットワークドライバ内での XDP プログラム実行の典型例：

```c
// drivers/net/ethernet/mellanox/mlx5/core/en_rx.c  
static inline int mlx5e_xdp_handle(struct mlx5e_rq *rq,
                                  struct mlx5e_dma_info *di,
                                  void *va, u16 *rx_headroom, u32 *len)
{
    struct bpf_prog *prog = rcu_dereference(rq->xdp_prog);
    struct xdp_buff xdp;
    u32 act;
    int err;

    if (!prog)
        return MLX5E_XDP_PASS;

    /* XDP バッファの初期化 */
    xdp.data = va + *rx_headroom;
    xdp_set_data_meta_invalid(&xdp);
    xdp.data_end = xdp.data + *len;
    xdp.data_hard_start = va;
    xdp.rxq = &rq->xdp_rxq;

    /* XDP プログラムの実行 */
    act = bpf_prog_run_xdp(prog, &xdp);

    /* 戻り値に応じた処理分岐 */
    switch (act) {
    case XDP_PASS:
        *rx_headroom = xdp.data - xdp.data_hard_start;
        *len = xdp.data_end - xdp.data;
        return MLX5E_XDP_PASS;
    case XDP_TX:
        err = mlx5e_xdp_xmit(rq->netdev, 1, &xdp, 0);
        if (unlikely(err))
            goto xdp_abort;
        __set_bit(MLX5E_RQ_FLAG_XDP_XMIT, rq->flags);
        return err ? MLX5E_XDP_CONSUMED : MLX5E_XDP_TX;
    case XDP_REDIRECT:
        /* リダイレクト処理 */
        err = xdp_do_redirect(rq->netdev, &xdp, prog);
        if (unlikely(err))
            goto xdp_abort;
        __set_bit(MLX5E_RQ_FLAG_XDP_REDIRECT, rq->flags);
        return MLX5E_XDP_REDIRECT;
    default:
        bpf_warn_invalid_xdp_action(act);
        fallthrough;
    case XDP_ABORTED:
xdp_abort:
        trace_xdp_exception(rq->netdev, prog, act);
        fallthrough;
    case XDP_DROP:
        return MLX5E_XDP_CONSUMED;
    }
}
```

**解析ポイント:**
1. **プログラム取得**: `rcu_dereference()` でRCU保護されたプログラムポインターを取得
2. **バッファ設定**: `xdp_buff` 構造体でパケットメタデータを準備
3. **プログラム実行**: `bpf_prog_run_xdp()` でXDPプログラムを実行
4. **結果処理**: XDP アクションに応じた分岐処理
5. **統計更新**: `trace_xdp_exception()` で例外ケースをトレーシング

#### eBPF Map の実装解析

eBPF Map は eBPF プログラムとユーザー空間の間でデータを共有する仕組み：

```c
// include/linux/bpf.h
struct bpf_map {
    /* Hot fields */
    const struct bpf_map_ops *ops ____cacheline_aligned;
    struct bpf_map *inner_map_meta;
    void *security;
    enum bpf_map_type map_type;
    u32 key_size;
    u32 value_size;
    u32 max_entries;
    u32 map_flags;
    int spin_lock_off; /* >=0 valid offset, <0 error */
    u32 id;
    int numa_node;
    u32 btf_key_type_id;
    u32 btf_value_type_id;
    struct btf *btf;
    struct mem_cgroup *memcg;
    char name[BPF_OBJ_NAME_LEN];
    u32 btf_vmlinux_value_type_id;
    bool bypass_spec_v1;
    bool frozen; /* write-once; write-protected by freeze_mutex */
    /* 22 bytes hole */

    /* The 3rd and 4th cacheline with misc members to avoid false sharing
     * particularly with refcounting.
     */
    atomic64_t refcnt ____cacheline_aligned;
    atomic64_t usercnt;
    struct work_struct work;
    struct mutex freeze_mutex;
    u64 writecnt; /* writable mmap cnt; protected by freeze_mutex */
};
```

**Map 操作の実装例（ハッシュマップ）:**

```c
// kernel/bpf/hashtab.c
static void *htab_map_lookup_elem(struct bpf_map *map, void *key)
{
    struct bpf_htab *htab = container_of(map, struct bpf_htab, map);
    struct hlist_nulls_head *head;
    struct htab_elem *l;
    u32 hash, key_size;

    WARN_ON_ONCE(!rcu_read_lock_held() && !rcu_read_lock_trace_held());

    key_size = map->key_size;

    /* ハッシュ値の計算 */
    hash = htab_map_hash(key, key_size, htab->hashrnd);

    /* バケット選択 */
    head = select_bucket(htab, hash);

    /* RCU保護下での線形探索 */
    l = lookup_nulls_elem_raw(head, hash, key, key_size, htab->n_buckets);

    return l ? l->key + round_up(key_size, 8) : NULL;
}

static int htab_map_update_elem(struct bpf_map *map, void *key, void *value,
                               u64 map_flags)
{
    struct bpf_htab *htab = container_of(map, struct bpf_htab, map);
    struct htab_elem *l_new = NULL, *l_old;
    struct hlist_nulls_head *head;
    unsigned long flags;
    struct bucket *b;
    u32 key_size, hash;
    int ret;

    if (unlikely((map_flags & ~BPF_F_LOCK) > BPF_EXIST))
        return -EINVAL;

    WARN_ON_ONCE(!rcu_read_lock_held() && !rcu_read_lock_trace_held());

    key_size = map->key_size;

    if (unlikely(map_flags & BPF_F_LOCK)) {
        if (unlikely(!map_value_has_spin_lock(map)))
            return -EINVAL;
        if (unlikely(map_flags & BPF_NOEXIST))
            return -EINVAL;
    }

    hash = htab_map_hash(key, key_size, htab->hashrnd);

    b = __select_bucket(htab, hash);
    head = &b->head;

    /* 既存エントリの検索 */
    if (map_flags & BPF_F_LOCK) {
        l_old = lookup_elem_raw(head, hash, key, key_size);
        if (!l_old)
            return -ENOENT;
        if (map_flags & BPF_F_LOCK)
            copy_map_value_locked(map, l_old->key + round_up(key_size, 8),
                                 value, false);
        else
            copy_map_value(map, l_old->key + round_up(key_size, 8), value);
        return 0;
    }

    /* 新しいエントリの割り当て */
    l_new = alloc_htab_elem(htab, key, value, key_size, hash, false, false,
                           l_old);
    if (IS_ERR(l_new))
        return PTR_ERR(l_new);

    /* スピンロック取得 */
    ret = htab_lock_bucket(htab, b, hash, &flags);
    if (ret)
        goto err;

    l_old = lookup_elem_raw(head, hash, key, key_size);

    ret = check_flags(htab, l_old, map_flags);
    if (ret)
        goto err;

    /* 既存エントリの置き換えまたは新規追加 */
    if (l_old) {
        /* 既存エントリを置き換え */
        hlist_nulls_replace_rcu(&l_old->hash_node, &l_new->hash_node);
    } else {
        /* 新規エントリを追加 */
        hlist_nulls_add_head_rcu(&l_new->hash_node, head);
        if (htab_is_lru(htab))
            bpf_lru_add(&htab->lru, &l_new->lru_node);
        else
            htab_inc_elem_count(htab);
    }

    htab_unlock_bucket(htab, b, hash, flags);

    /* 古いエントリをRCU後に削除 */
    if (l_old) {
        if (htab_is_lru(htab))
            bpf_lru_del(&htab->lru, &l_old->lru_node);
        else
            htab_dec_elem_count(htab);
        call_rcu(&l_old->rcu, htab_elem_free_rcu);
    }
    return 0;

err:
    if (l_new)
        bpf_mem_cache_free(&htab->ma, l_new);
    htab_unlock_bucket(htab, b, hash, flags);
    return ret;
}
```

**解析ポイント:**
1. **ハッシュ計算**: `htab_map_hash()` でキーから高速ハッシュを計算
2. **RCU同期**: `rcu_read_lock()` で読み込み時のメモリ安全性を保証
3. **バケット処理**: `select_bucket()` でハッシュテーブルのバケットを選択
4. **スピンロック**: `htab_lock_bucket()` で書き込み時の排他制御
5. **メモリ管理**: `bpf_mem_cache_free()` で効率的なメモリ割り当て/解放
6. **LRU管理**: `bpf_lru_add()` で使用頻度に基づく自動削除

### Socket Filter による詳細パケット解析

ソケットレベルでのパケットフィルタリングでは、より詳細なプロトコル解析が可能です。

**関連ソースコード:**
```c
// ソケットフィルタ実装
net/core/filter.c
  ├── sk_run_filter()           // ソケットフィルタ実行
  ├── bpf_skb_load_bytes()      // パケットデータ読み込み
  └── bpf_clone_redirect()      // パケットクローンとリダイレクト

// ソケットフィルタアタッチ
net/core/sock.c
  └── sock_setsockopt()         // SO_ATTACH_BPF オプション処理
```

## eBPF のユースケース

### 1. ネットワーク: パケットフィルタリング、ロードバランシング、ファイアウォール

#### パケットフィルタリング
**実装例:**
```c
// DDoS 攻撃防御 eBPF プログラム
SEC("xdp")
int ddos_protection(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    
    struct ethhdr *eth = data;
    if (eth + 1 > data_end)
        return XDP_PASS;
    
    if (eth->h_proto != htons(ETH_P_IP))
        return XDP_PASS;
    
    struct iphdr *ip = (struct iphdr *)(eth + 1);
    if (ip + 1 > data_end)
        return XDP_PASS;
    
    // 特定の送信元IPからのトラフィックをブロック
    if (ip->saddr == htonl(0xc0a80100)) // 192.168.1.0
        return XDP_DROP;
    
    return XDP_PASS;
}
```

#### ロードバランシング
Cilium などのプロジェクトでは eBPF を使用して高性能なロードバランシングを実現：

**関連ソースコード:**
```c
// TC (Traffic Control) による eBPF
net/sched/cls_bpf.c
  └── cls_bpf_classify()        // BPF 分類器実行

// eBPF helper 関数
kernel/bpf/helpers.c
  ├── bpf_redirect()            // パケットリダイレクト
  └── bpf_skb_change_head()     // パケットヘッダー変更
```

### 2. トレーシング / オブザーバビリティ

#### システムコール監視
**実装例:**
```c
// openat システムコール監視
SEC("tracepoint/syscalls/sys_enter_openat")
int trace_openat(struct trace_event_raw_sys_enter *ctx) {
    pid_t pid = bpf_get_current_pid_tgid() >> 32;
    char filename[256];
    
    // ファイル名を取得
    bpf_probe_read_user_str(filename, sizeof(filename), 
                           (void *)ctx->args[1]);
    
    bpf_printk("PID %d opened file: %s\n", pid, filename);
    return 0;
}
```

**関連ソースコード:**
```c
// トレースポイント実装
include/trace/events/ 
  ├── syscalls.h                // システムコールトレースポイント
  └── kmem.h                   // メモリアロケーション監視

// kprobe/uprobe サポート
kernel/trace/trace_kprobe.c
  ├── __kprobe_trace_func()     // kprobe eBPF 実行
  └── kretprobe_trace_func()    // kretprobe eBPF 実行
```

#### パフォーマンス解析
**CPU 使用率監視:**
```c
// CPU サンプリング
SEC("perf_event")
int cpu_profile(struct bpf_perf_event_data *ctx) {
    u64 pid_tgid = bpf_get_current_pid_tgid();
    u32 pid = pid_tgid >> 32;
    
    // スタックトレース収集
    int key = 0;
    bpf_get_stack(ctx, stack_traces, sizeof(stack_traces), 0);
    
    return 0;
}
```

### 3. セキュリティ: サンドボックス、システムコール監視

#### LSM (Linux Security Modules) との統合
eBPF プログラムを LSM フックにアタッチしてセキュリティポリシーを実装：

**関連ソースコード:**
```c
// eBPF LSM 実装
security/bpf/hooks.c
  ├── bpf_lsm_file_open()       // ファイルオープン制御
  ├── bpf_lsm_task_alloc()      // プロセス作成制御
  └── bpf_lsm_socket_create()   // ソケット作成制御

// LSM フック定義
include/linux/lsm_hooks.h
  └── struct security_hook_heads // セキュリティフック一覧
```

#### seccomp との統合
```c
// seccomp eBPF フィルタ
kernel/seccomp.c
  ├── __seccomp_filter()        // seccomp フィルタ実行
  └── seccomp_run_filters()     // eBPF プログラム実行
```

## eBPF の企業での活用事例

### 1. Netflix - 高性能ロードバランサ
Netflix は eBPF を使用してカーネルバイパス型ロードバランサを構築：
- **Katran プロジェクト**: XDP ベースの L4 ロードバランサ
- **パフォーマンス**: 従来比 10-20倍の性能向上

### 2. Cloudflare - DDoS 攻撃防御
Cloudflare は XDP を使用して大規模 DDoS 攻撃を防御：
- **L4Drop**: eBPF ベースのパケットドロップシステム
- **処理能力**: 1台のサーバで数十Mpps の攻撃を防御

### 3. Facebook (Meta) - ネットワーク可観測性
Facebook は eBPF を大規模データセンターの監視に活用：
- **katran**: L4 ロードバランサ
- **bpftrace**: システム動作の詳細監視

### 4. Google - コンテナセキュリティ
Google は gVisor で eBPF を活用：
- **Sentry**: システムコール監視とサンドボッシング
- **性能**: ネイティブ実行に近い性能でセキュリティを確保

**関連するオープンソースプロジェクト:**
```bash
# 主要な eBPF プロジェクトとソースコード
├── Cilium          # ネットワークセキュリティ・可観測性
├── Falco           # ランタイムセキュリティ監視  
├── Katran          # L4 ロードバランサ
├── bcc             # eBPF 開発ツール
└── libbpf          # eBPF ライブラリ
```

## eBPF を実際に試すにはどうすればよいか？

### 1. 開発環境の準備

#### カーネル要件
- Linux カーネル 4.1 以上（基本機能）
- Linux カーネル 4.8 以上（XDP サポート）  
- Linux カーネル 5.8 以上（LSM サポート）

#### 必要な設定
```bash
# カーネル設定確認
grep CONFIG_BPF /boot/config-$(uname -r)
grep CONFIG_BPF_SYSCALL /boot/config-$(uname -r)
grep CONFIG_XDP_SOCKETS /boot/config-$(uname -r)

# 必要なパッケージインストール (Ubuntu/Debian)
sudo apt install clang llvm libelf-dev linux-headers-$(uname -r)
sudo apt install bpfcc-tools bpftrace

# libbpf インストール
git clone https://github.com/libbpf/libbpf.git
cd libbpf/src
make
sudo make install
```

### 2. 簡単な eBPF プログラムの作成

#### Hello World 例
**hello.c:**
```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("tracepoint/syscalls/sys_enter_openat")
int trace_openat(void *ctx) {
    char msg[] = "Hello, eBPF World!\n";
    bpf_trace_printk(msg, sizeof(msg));
    return 0;
}

char _license[] SEC("license") = "GPL";
```

**コンパイルとロード:**
```bash
# eBPF プログラムコンパイル
clang -O2 -target bpf -c hello.c -o hello.o

# ロードと実行
sudo bpftool prog load hello.o /sys/fs/bpf/hello
sudo bpftool prog attach pinned /sys/fs/bpf/hello tracepoint/syscalls/sys_enter_openat

# ログ確認
sudo cat /sys/kernel/debug/tracing/trace_pipe
```

### 3. パケットフィルタリング例

#### 基本的な XDP プログラム
**xdp_drop.c:**
```c
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_drop_tcp(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    
    struct ethhdr *eth = data;
    if (eth + 1 > data_end)
        return XDP_PASS;
    
    if (eth->h_proto != __constant_htons(ETH_P_IP))
        return XDP_PASS;
    
    struct iphdr *ip = (struct iphdr *)(eth + 1);
    if (ip + 1 > data_end)
        return XDP_PASS;
    
    // TCP トラフィックをドロップ
    if (ip->protocol == IPPROTO_TCP)
        return XDP_DROP;
    
    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

**アタッチとテスト:**
```bash
# コンパイル
clang -O2 -target bpf -c xdp_drop.c -o xdp_drop.o

# ネットワークインターフェースにアタッチ
sudo ip link set dev eth0 xdp obj xdp_drop.o sec xdp

# プログラム確認
sudo bpftool prog list
sudo bpftool net list

# デタッチ
sudo ip link set dev eth0 xdp off
```

### 4. 高レベル開発ツールの使用

#### bcc (BPF Compiler Collection) の使用
```python
#!/usr/bin/env python3
# tcp_connect.py - TCP 接続監視

from bcc import BPF

# eBPF プログラム (C)
program = """
#include <uapi/linux/ptrace.h>
#include <net/sock.h>
#include <bcc/proto.h>

int trace_connect(struct pt_regs *ctx, struct sock *sk) {
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u16 dport = sk->__sk_common.skc_dport;
    u32 daddr = sk->__sk_common.skc_daddr;
    
    bpf_trace_printk("PID %d connected to %d.%d.%d.%d:%d\\n",
        pid, daddr & 0xff, (daddr >> 8) & 0xff,
        (daddr >> 16) & 0xff, (daddr >> 24) & 0xff,
        ntohs(dport));
    
    return 0;
}
"""

# eBPF プログラムロード
b = BPF(text=program)
b.attach_kprobe(event="tcp_v4_connect", fn_name="trace_connect")

# イベント出力
print("Tracing TCP connections... Ctrl-C to end")
try:
    b.trace_print()
except KeyboardInterrupt:
    print("Detaching...")
```

#### bpftrace の使用例
```bash
# TCP 接続をワンライナーで監視
sudo bpftrace -e 'kprobe:tcp_v4_connect { printf("TCP connect by PID %d\n", pid); }'

# システムコール頻度をカウント
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[probe] = count(); }'

# ファイルアクセス監視
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s opened %s\n", comm, str(args->filename)); }'
```

### 5. デバッグとトラブルシューティング

#### よくある問題と解決法
```bash
# eBPF プログラムの検証エラー
sudo bpftool prog load program.o /sys/fs/bpf/test 2>&1 | head -20

# Map の状態確認
sudo bpftool map list
sudo bpftool map dump id <map_id>

# プログラムの実行統計
sudo bpftool prog show id <prog_id>

# カーネルログ確認
dmesg | grep bpf
```

### 6. 学習リソースとドキュメント

**公式ドキュメント:**
- `Documentation/bpf/` - カーネル内 eBPF ドキュメント
- `samples/bpf/` - サンプルプログラム集
- `tools/testing/selftests/bpf/` - テストプログラム

**オンライン学習リソース:**
- eBPF.io - 公式学習サイト
- Cilium eBPF ガイド
- bcc チュートリアル

### 7. 実践的な学習ステップ

1. **基礎学習**: トレースポイントを使った簡単な監視プログラム
2. **ネットワーク**: XDP を使ったパケットフィルタリング
3. **可観測性**: 詳細なシステム監視ツールの作成
4. **パフォーマンス**: Maps を使った統計情報の収集
5. **セキュリティ**: LSM フックを使ったアクセス制御

これらの手順を段階的に実践することで、eBPF の強力な機能を効果的に活用できるようになります。