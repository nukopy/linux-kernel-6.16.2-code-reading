# Linux Kernel 6.16.2 ブートフロー詳細 (x86_64)

このドキュメントでは、Linux カーネル 6.16.2 の x86_64 アーキテクチャにおけるブートプロセスを、実際のソースコードと対応付けながら詳細に説明します。

## 概要：ブートフローの全体像

```
[電源ON] → [BIOS/UEFI] → [ブートローダー(GRUB等)] → [Linux カーネル]
                                                          ↓
                                    [リアルモード] → [プロテクトモード] → [ロングモード]
                                          ↓               ↓                  ↓
                                    [16-bit code]   [32-bit code]      [64-bit code]
```

## 1. BIOS/UEFI からブートローダーへ

### 1.1 電源投入とファームウェア初期化

- **動作**: 電源投入後、CPU は リセットベクタ (0xFFFFFFF0) から実行開始
- **処理内容**:
  - POST (Power-On Self Test) の実行
  - ハードウェア初期化
  - ブートデバイスの検索
  - MBR (Master Boot Record) または EFI システムパーティションからブートローダーをロード

### 1.2 ブートローダー (GRUB) の実行

- **動作**: ブートローダーがカーネルイメージをメモリにロード
- **処理内容**:
  - カーネルイメージ (`vmlinuz`) を指定アドレスにロード
  - initrd/initramfs をメモリにロード
  - カーネルコマンドラインパラメータの設定
  - カーネルのエントリーポイントへジャンプ

## 2. Linux カーネルの起動：リアルモードステージ

### 2.1 リアルモードエントリー

**ファイル**: `arch/x86/boot/header.S`
**エントリーポイント**: `_start` (オフセット 0x200)

```assembly
# arch/x86/boot/header.S
_start:
    # Boot sector code (512 bytes)
    .byte   0xeb        # Jump instruction
    .byte   start_of_setup-1f
    # ...
start_of_setup:
    # Real mode setup code begins here
```

**処理内容**:

- ブートセクタのマジックナンバー確認 (0xAA55)
- ブートプロトコルバージョンの設定
- カーネルの各種パラメータ（ロードアドレス、カーネルサイズ等）の定義
- `main()` 関数へのジャンプ

**重要なデータ構造**:

- `struct boot_params` - ブートパラメータ構造体
- `hdr` - ブートヘッダー（カーネルバージョン、ロードフラグ等）

### 2.2 リアルモード C コード実行

**ファイル**: `arch/x86/boot/main.c`
**エントリーポイント**: `main()`

```c
// arch/x86/boot/main.c
void main(void)
{
    copy_boot_params();      // ブートパラメータのコピー
    console_init();          // コンソール初期化
    init_heap();            // ヒープ初期化
    validate_cpu();         // CPU機能の検証
    detect_memory();        // メモリマップの検出
    keyboard_init();        // キーボード初期化
    query_ist();           // Intel SpeedStep情報
    query_apm_bios();      // APM BIOS情報
    query_edd();           // Enhanced Disk Drive情報
    set_video();           // ビデオモード設定
    go_to_protected_mode(); // プロテクトモードへ移行
}
```

**処理内容**:

- BIOS 呼び出しによるハードウェア情報収集
- E820 メモリマップの取得
- ビデオモードの設定
- A20 ゲートの有効化準備

## 3. プロテクトモードへの移行

### 3.1 プロテクトモード移行準備

**ファイル**: `arch/x86/boot/pm.c`
**エントリーポイント**: `go_to_protected_mode()`

```c
// arch/x86/boot/pm.c
void go_to_protected_mode(void)
{
    realmode_switch_hook();     // リアルモードフック実行
    enable_a20();              // A20ゲート有効化
    reset_coprocessor();       // 数値演算コプロセッサリセット
    mask_all_interrupts();     // 全割り込みマスク
    setup_idt();              // IDT設定（null IDT）
    setup_gdt();              // GDT設定
    protected_mode_jump(boot_params.hdr.code32_start,
                       (u32)&boot_params);
}
```

**処理内容**:

- A20 ゲート有効化（1MB 以上のメモリアクセス可能に）
- GDT (Global Descriptor Table) の設定
- IDT (Interrupt Descriptor Table) の初期化
- CR0 レジスタの PE ビット設定

### 3.2 プロテクトモードへのジャンプ

**ファイル**: `arch/x86/boot/pmjump.S`
**エントリーポイント**: `protected_mode_jump`

```assembly
# arch/x86/boot/pmjump.S
SYM_FUNC_START_NOALIGN(protected_mode_jump)
    movl    %cr0, %edx
    orb     $X86_CR0_PE, %dl    # Protected Mode Enable
    movl    %edx, %cr0

    # Far jump to 32-bit code
    ljmpl   $__BOOT_CS, $pa_2

SYM_FUNC_START_LOCAL_NOALIGN(pa_2)
    # Now in 32-bit protected mode
    movl    $__BOOT_DS, %ecx
    movl    %ecx, %ds
    movl    %ecx, %es
    movl    %ecx, %fs
    movl    %ecx, %gs
    movl    %ecx, %ss

    # Jump to compressed kernel
    jmpl    *%eax
```

## 4. 圧縮カーネルステージ（32 ビット）

### 4.1 圧縮カーネル 32 ビットエントリー

**ファイル**: `arch/x86/boot/compressed/head_64.S`
**エントリーポイント**: `startup_32`

```assembly
# arch/x86/boot/compressed/head_64.S
SYM_FUNC_START(startup_32)
    cld
    cli

    # Calculate relocation offset
    leal    (BP_scratch+4)(%esi), %esp
    call    1f
1:  popl    %ebp
    subl    $(1b - startup_32), %ebp

    # Load new GDT with 64-bit segments
    leal    gdt(%ebp), %eax
    movl    %eax, 2(%eax)
    lgdt    (%eax)

    # Enable PAE and PGE
    movl    %cr4, %eax
    orl     $(X86_CR4_PAE | X86_CR4_PGE), %eax
    movl    %eax, %cr4

    # Build early page tables
    # ... (ページテーブル構築コード)

    # Enable long mode
    movl    $MSR_EFER, %ecx
    rdmsr
    btsl    $_EFER_LME, %eax
    wrmsr

    # Enable paging and long mode
    movl    %cr0, %eax
    orl     $(X86_CR0_PG | X86_CR0_WP), %eax
    movl    %eax, %cr0

    # Jump to 64-bit code
    ljmp    $__KERNEL_CS, $startup_64
```

**処理内容**:

- 再配置オフセットの計算
- 64 ビット対応 GDT のロード
- PAE (Physical Address Extension) 有効化
- 4 レベルページテーブルの構築（PML4 → PDPT → PD → PT）
- EFER.LME ビット設定（Long Mode Enable）
- ページング有効化
- 64 ビットコードへのファージャンプ

## 5. 圧縮カーネルステージ（64 ビット）

### 5.1 圧縮カーネル 64 ビットエントリー

**ファイル**: `arch/x86/boot/compressed/head_64.S`
**エントリーポイント**: `startup_64` (オフセット 0x200)

```assembly
# arch/x86/boot/compressed/head_64.S
SYM_FUNC_START(startup_64)
    # Clear registers and setup stack
    xorl    %eax, %eax
    movl    %eax, %ds
    movl    %eax, %es
    movl    %eax, %ss
    movl    %eax, %fs
    movl    %eax, %gs

    # Setup stack
    leaq    boot_stack_end(%rbx), %rsp

    # Copy compressed kernel to safe location
    pushq   %rsi                # boot_params
    movq    %rsi, %rdi          # destination
    leaq    (_bss-8)(%rip), %rsi # source
    movl    $(_bss - startup_32), %ecx
    std
    rep     movsq
    cld
    popq    %rsi

    # Jump to relocated code
    leaq    .Lrelocated(%rbx), %rax
    jmp     *%rax

.Lrelocated:
    # Clear BSS
    xorl    %eax, %eax
    leaq    _bss(%rip), %rdi
    leaq    _ebss(%rip), %rcx
    subq    %rdi, %rcx
    shrq    $3, %rcx
    rep     stosq

    # Call kernel extraction
    pushq   %rsi                # boot_params
    movq    %rsi, %rdi          # boot_params argument
    leaq    boot_heap(%rip), %rsi # heap area
    call    extract_kernel
    popq    %rsi

    # Jump to decompressed kernel
    jmp     *%rax
```

### 5.2 カーネル展開処理

**ファイル**: `arch/x86/boot/compressed/misc.c`
**エントリーポイント**: `extract_kernel()`

```c
// arch/x86/boot/compressed/misc.c
asmlinkage __visible void *extract_kernel(void *rmode,
                                          unsigned char *input_data,
                                          unsigned long input_len,
                                          unsigned char *output,
                                          unsigned long output_len)
{
    const unsigned long kernel_total_size = VO__end - VO__text;
    unsigned long virt_addr = LOAD_PHYSICAL_ADDR;

    /* Initialize boot console */
    console_init();

    /* Kernel Address Space Layout Randomization */
    if (IS_ENABLED(CONFIG_RANDOMIZE_BASE))
        virt_addr = choose_random_location(input_data, input_len,
                                          output, output_len);

    /* Decompress kernel */
    __decompress(input_data, input_len, NULL, NULL, output,
                output_len, NULL, error);

    /* Parse ELF and apply relocations */
    parse_elf(output);
    handle_relocations(output, output_len, virt_addr);

    return output;
}
```

**処理内容**:

- ヒープ初期化
- KASLR (Kernel Address Space Layout Randomization) 実行
- カーネル解凍（gzip/bzip2/xz/lz4 等）
- ELF ヘッダー解析
- 再配置処理
- BSS セクションのクリア

## 6. メインカーネルエントリー

### 6.1 カーネル 64 ビットエントリー

**ファイル**: `arch/x86/kernel/head_64.S`
**エントリーポイント**: `startup_64`

```assembly
# arch/x86/kernel/head_64.S
SYM_CODE_START_NOALIGN(startup_64)
    # Verify CPU type
    call    verify_cpu
    testl   %eax, %eax
    jnz     .Lno_longmode

    # Setup early page tables
    leaq    _text(%rip), %rdi
    movq    %rdi, %rax
    shrq    $PGDIR_SHIFT, %rax

    leaq    early_top_pgt(%rip), %rdx
    addq    $(PAGE_SIZE + _PAGE_TABLE_NOENC), %rdx
    movq    %rdx, (%rbx)

    # Load new page table
    movq    %rax, %cr3

    # Jump to common startup code
    jmp     common_startup_64

SYM_CODE_START_LOCAL(common_startup_64)
    # Setup GS for per-CPU data
    movl    $MSR_GS_BASE, %ecx
    movl    initial_gs(%rip), %eax
    movl    initial_gs+4(%rip), %edx
    wrmsr

    # Setup IDT
    lidt    idt_descr(%rip)

    # Jump to C code
    movq    initial_code(%rip), %rax
    pushq   $__KERNEL_CS
    pushq   %rax
    lretq
```

### 6.2 カーネル C エントリー

**ファイル**: `arch/x86/kernel/head64.c`
**エントリーポイント**: `x86_64_start_kernel()`

```c
// arch/x86/kernel/head64.c
asmlinkage __visible void __init x86_64_start_kernel(char *real_mode_data)
{
    /* Clear BSS */
    clear_bss();

    /* Reset early page tables */
    reset_early_page_tables();

    /* Clear initial page tables */
    clear_page(init_top_pgt);

    /* SME (Secure Memory Encryption) initialization */
    sme_early_init();

    /* KASAN initialization */
    kasan_early_init();

    /* Setup early exception handlers */
    idt_setup_early_handler();

    /* Copy boot params */
    copy_bootdata(__va(real_mode_data));

    /* Load microcode */
    load_ucode_bsp();

    /* Initialize boot CPU data */
    init_cpu_data_detached(0);

    /* Jump to generic kernel initialization */
    x86_64_start_reservations(real_mode_data);
}

void __init x86_64_start_reservations(char *real_mode_data)
{
    /* Copy initial ramdisk */
    if (boot_params.hdr.type_of_loader && boot_params.hdr.ramdisk_image) {
        memblock_reserve(ramdisk_image, ramdisk_end - ramdisk_image);
    }

    /* Reserve memory regions */
    reserve_early_setup_data();

    /* Call main kernel initialization */
    start_kernel();
}
```

## 7. カーネル初期化メインフロー

### 7.1 start_kernel() - カーネル初期化の中心

**ファイル**: `init/main.c:898`
**エントリーポイント**: `start_kernel()`

```c
// init/main.c
asmlinkage __visible __init __no_sanitize_address __noreturn __no_stack_protector
void start_kernel(void)
{
    char *command_line;

    /* 初期タスク設定 */
    set_task_stack_end_magic(&init_task);
    smp_setup_processor_id();

    /* 初期デバッグ設定 */
    debug_objects_early_init();
    init_vmlinux_build_id();

    /* cgroup早期初期化 */
    cgroup_init_early();

    /* 割り込み無効化 */
    local_irq_disable();
    early_boot_irqs_disabled = true;

    /* ブートCPU初期化 */
    boot_cpu_init();
    page_address_init();

    /* アーキテクチャ固有の設定 */
    pr_notice("%s", linux_banner);
    setup_arch(&command_line);

    /* 静的キーと静的呼び出しの初期化 */
    jump_label_init();
    static_call_init();

    /* セキュリティ初期化 */
    early_security_init();

    /* コマンドライン処理 */
    setup_boot_config();
    setup_command_line(command_line);

    /* CPU設定 */
    setup_nr_cpu_ids();
    setup_per_cpu_areas();
    smp_prepare_boot_cpu();

    /* メモリ管理初期化 */
    mm_core_init();

    /* スケジューラ初期化 */
    sched_init();

    /* 割り込み関連初期化 */
    radix_tree_init();
    maple_tree_init();
    housekeeping_init();
    workqueue_init_early();
    rcu_init();

    /* トレース機能初期化 */
    trace_init();

    /* 残りのサブシステム初期化... */

    /* 最終的にrest_init()を呼び出し */
    arch_call_rest_init();
}

void __init __weak arch_call_rest_init(void)
{
    rest_init();
}
```

### 7.2 rest_init() - ユーザー空間への移行準備

**ファイル**: `init/main.c`
**エントリーポイント**: `rest_init()`

```c
// init/main.c
noinline void __ref __noreturn rest_init(void)
{
    struct task_struct *tsk;
    int pid;

    /* RCU初期化完了 */
    rcu_scheduler_starting();

    /* kernel_init スレッド作成（PID 1） */
    pid = user_mode_thread(kernel_init, NULL, CLONE_FS);

    /* kthreadd スレッド作成（PID 2） */
    pid = kernel_thread(kthreadd, NULL, NULL, CLONE_FS | CLONE_FILES);

    /* Boot CPU をアイドル状態へ */
    cpu_startup_entry(CPUHP_ONLINE);
}
```

## 8. init プロセスの起動

### 8.1 kernel_init() - 最初のユーザープロセス

**ファイル**: `init/main.c`
**エントリーポイント**: `kernel_init()`

```c
// init/main.c
static int __ref kernel_init(void *unused)
{
    /* 全てのサブシステム初期化を完了 */
    kernel_init_freeable();

    /* initメモリ解放 */
    free_initmem();
    mark_readonly();

    /* ユーザーモードへの移行準備 */
    numa_default_policy();

    /* /sbin/init, /etc/init, /bin/init, /bin/sh の順で実行試行 */
    if (ramdisk_execute_command) {
        ret = run_init_process(ramdisk_execute_command);
    }

    if (execute_command) {
        ret = run_init_process(execute_command);
    }

    ret = run_init_process("/sbin/init");
    ret = run_init_process("/etc/init");
    ret = run_init_process("/bin/init");
    ret = run_init_process("/bin/sh");

    panic("No working init found.");
}
```

## まとめ：ブートフローの要点

### フェーズ遷移のまとめ

| フェーズ                     | モード         | ビット幅 | 主要ファイル                            | 主な処理                             |
| ---------------------------- | -------------- | -------- | --------------------------------------- | ------------------------------------ |
| 1. リアルモードセットアップ  | Real Mode      | 16-bit   | `arch/x86/boot/header.S`, `main.c`      | BIOS 呼び出し、メモリ検出            |
| 2. プロテクトモード移行      | Protected Mode | 32-bit   | `arch/x86/boot/pm.c`, `pmjump.S`        | A20 有効化、GDT 設定                 |
| 3. 圧縮カーネル（32 ビット） | Protected Mode | 32-bit   | `arch/x86/boot/compressed/head_64.S`    | ページテーブル構築、Long Mode 有効化 |
| 4. 圧縮カーネル（64 ビット） | Long Mode      | 64-bit   | `arch/x86/boot/compressed/head_64.S`    | カーネル再配置、展開準備             |
| 5. カーネル展開              | Long Mode      | 64-bit   | `arch/x86/boot/compressed/misc.c`       | 解凍、ELF 解析、KASLR                |
| 6. メインカーネル開始        | Long Mode      | 64-bit   | `arch/x86/kernel/head_64.S`, `head64.c` | 最終ページテーブル、CPU 初期化       |
| 7. カーネル初期化            | Long Mode      | 64-bit   | `init/main.c:start_kernel()`            | 全サブシステム初期化                 |
| 8. init 起動                 | Long Mode      | 64-bit   | `init/main.c:kernel_init()`             | PID 1 プロセス、ユーザー空間へ       |

### 重要なデータ構造と遷移

1. **boot_params 構造体**: リアルモードからカーネル全体へブート情報を伝達
2. **ページテーブル**: 初期簡易版 → 圧縮カーネル用 → 最終カーネル用と段階的に構築
3. **GDT/IDT**: リアルモード → プロテクトモード用 → ロングモード用と更新
4. **スタック**: 各段階で適切なスタックポインタを設定

### メモリレイアウトの変遷

```
リアルモード:     0x00000 - 0x100000 (1MB以下)
プロテクトモード: 0x100000以上も使用可能（A20有効化後）
ロングモード:     仮想アドレス空間（カーネル空間: 0xFFFF8000_00000000以降）
```

このブートプロセスにより、16 ビットの BIOS 環境から 64 ビットの Linux カーネルへと段階的に移行し、最終的に完全な機能を持つオペレーティングシステムが起動します。
