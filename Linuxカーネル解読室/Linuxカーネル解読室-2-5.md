# 2.5 マスク不可割り込みNMI

  * 外部ハードウェアが NMIピン をアサート
  * APICがNMIメッセージを流す

> NMIは特殊な用途に利用しています。通常の割り込みは禁止（マスク）することが可能ですが、このNMIは禁止できません。CPUの割り込み禁止（local_irq_disable関数）を行っていても、NMIが発生すると、指定した割り込みハンドラへ実行を移すことができます。

 * CPUの割り込み禁止をシカト
   * 割り込みとはアサートされるピン/メッセージが別にしているので?

>　NMIは特殊な目的で利用されます。ハードウェアに依存しますが、メモリのパリティエラー発生の捕捉、ウォッチドッグ、デバッガの強制起動などに利用されます。

 * NMI割り込みハンドラはどこで定義? => ENTRY(nmi)
   * IDT(割り込みベクタ)の2番
 * パリティエラー
   * mem_parity_error
 * I/Oチェックエラー
   * io_check_error
   * どんなデバイスが飛ばすんだろ?
     * `	printk(KERN_EMERG "NMI: IOCK error (debug interrupt?)\n");`
     * `		panic("NMI IOCK error: Not continuing");
 * watchdog_*** てなドライバ実装がある
   * watchdog_register_device(struct watchdog_device *wdd)

割り込みよりコールスタックが浅くて理解しやすい気も

```
[vagrant@vagrant-centos65 ~]$ find /sys/ | grep watchdog
/sys/class/watchdog
/sys/module/ehci_hcd/parameters/io_watchdog_force
```

>　Intel x86用Linuxでは、LDTに直接NMI用の割り込みハンドラ（nmi関数）を登録しています。NMIが発生するとその時点におけるCPU状態によらず、割り込みハンドラが呼び出されます。

 * LDT ... Local Descriptor Table
   * ????
 * ウォッチドッグ
   * http://ja.wikipedia.org/wiki/ウォッチドッグタイマー

## nmi

```asm
	/* runs on exception stack */
ENTRY(nmi)
	INTR_FRAME
	PARAVIRT_ADJUST_EXCEPTION_FRAME
	pushq_cfi $-1
	subq $15*8, %rsp
	CFI_ADJUST_CFA_OFFSET 15*8
	call save_paranoid
	DEFAULT_FRAME 0
	/* paranoidentry do_nmi, 0; without TRACE_IRQS_OFF */
	movq %rsp,%rdi
	movq $-1,%rsi
    // ここ
	call do_nmi
# #ifdef CONFIG_TRACE_IRQFLAGS
...
# #else
	jmp paranoid_exit
	CFI_ENDPROC
#endif
END(nmi)
```

```
dotraplinkage notrace __kprobes void
do_nmi(struct pt_regs *regs, long error_code)
{
    // irq_enter() 的な
	nmi_enter();

    // 統計インクリメント
	inc_irq_stat(__nmi_count);

	if (!ignore_nmis)
		default_do_nmi(regs);

   // irq_exit() 的な
	nmi_exit();
}
```

acpi_nmi_disable, acpi_nmi_enable で NMI の on/off を切り替える

```c
void stop_nmi(void)
{
	acpi_nmi_disable();
	ignore_nmis++;
}

void restart_nmi(void)
{
	ignore_nmis--;
	acpi_nmi_enable();
```

acpi_nmi_enable で NMI on/off は CPU ごとにセットされる

```
/*
 * Enable timer based NMIs on all CPUs:
 */
void acpi_nmi_enable(void)
{
	if (atomic_read(&nmi_active) && nmi_watchdog == NMI_IO_APIC)
		on_each_cpu(__acpi_nmi_enable, NULL, 1);
}
```

__acpi_nmi_enable は APIC になんか書いている

```c
static void __acpi_nmi_enable(void *__unused)
{
	apic_write(APIC_LVT0, APIC_DM_NMI);
}
```

## default_do_nmi

```c
static notrace __kprobes void default_do_nmi(struct pt_regs *regs)
{
	unsigned char reason = 0;
	int cpu;

	cpu = smp_processor_id();

	/* Only the BSP gets external NMIs from the system. */
	if (!cpu)
       	// return inb(0x61); で読みこんだ値
		reason = get_nmi_reason();

	if (!(reason & 0xc0)) {
		if (notify_die(DIE_NMI_IPI, "nmi_ipi", regs, reason, 2, SIGINT)
								== NOTIFY_STOP)
			return;

#ifdef CONFIG_X86_LOCAL_APIC
		if (notify_die(DIE_NMI, "nmi", regs, reason, 2, SIGINT)
							== NOTIFY_STOP)
			return;

#ifndef CONFIG_LOCKUP_DETECTOR
		/*
		 * Ok, so this is none of the documented NMI sources,
		 * so it must be the NMI watchdog.
		 */
		if (nmi_watchdog_tick(regs, reason))
			return;
		if (!do_nmi_callback(regs, cpu))
#endif /* !CONFIG_LOCKUP_DETECTOR */
			unknown_nmi_error(reason, regs);
#else
		unknown_nmi_error(reason, regs);
#endif

		return;
	}
	if (notify_die(DIE_NMI, "nmi", regs, reason, 2, SIGINT) == NOTIFY_STOP)
		return;

	/* AK: following checks seem to be broken on modern chipsets. FIXME */
	if (reason & 0x80)
        // パリティエラー
		mem_parity_error(reason, regs);
	if (reason & 0x40)
        // I/O 
		io_check_error(reason, regs);
#ifdef CONFIG_X86_32
	/*
	 * Reassert NMI in case it became active meanwhile
	 * as it's edge-triggered:
	 */
	reassert_nmi();
#endif
}
```