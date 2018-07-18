title: 中断虚拟化二
date: 2015-11-20 23:36:00
tags: Virtualization
categories: KVM
---
上篇讲到虚拟PIT注册了一个高精度定时器，1ms周期，注册的定时器处理函数为：kvm_timer_fn。综合以上机制介绍，来想像一个场景，就可以理解物理pit timer interrupt是如何经为虚拟pit提供时钟，进而产生虚拟timer interrupt提交给guest OS。假如：guest vcpu正在执行，来了一个时钟物理中断，而这个物理时钟中断也要传递给guest OS。

<!--more-->
## 中断流程
刚才讲到虚拟PIT注册了一个高精度定时器，1ms周期，注册的定时器处理函数为：
```c
kvm_timer_fn()------------- > __kvm_timer_fn()
	if (ktimer->reinject || !atomic_read(&ktimer->pending)) {
		atomic_inc(&ktimer->pending);
		set_bit(KVM_REQ_PENDING_TIMER, &vcpu->requests); 
		//设置KVM_REQ_PENDING_TIMER标志，表示有时间中断pendding
	}
```

综合以上机制介绍，来想像一个场景，就可以理解物理pit timer interrupt是如何经为虚拟pit提供时钟，进而产生虚拟timer interrupt提交给guest OS。假如：guest vcpu正在执行，来了一个时钟物理中断，而这个物理时钟中断也要传递给guest OS.

1. 物理中断导致VMExit，guest vcpu线程返回kvm_x86_ops->run，接着vcpu_enter_guest开中断，host timer interrupt handler处理这个物理中断。

2. timer interrupt handler发现内核PIT使用的hrtimer到时，调用kvm_pit_fun回调函数，将pit_state.pit_timer.pending置位

3. timer interrupt handler返回，vcpu_enter_guest调用kvm_x86_ops->handle_exit处理VMExit，发现exit_reason是外部中断，返回1，vcpu_enter_guest传递返回值回到__vcpu_run。


* kvm_x86_ops->run(vcpu); 即Vmx_vcpu_run() 发生外部中断口
* Vmx_vcpu_run()  开始退出时调用
* vmx_complete_interrupts()：
* vmx->exit_reason = vmcs_read32(VM_EXIT_REASON);  读取vm_exit_reason
* vmx_vcpu_run()  退出；
* 调用kvm_x86_ops->handle_exit(vcpu); 处理VMExit
* .handle_exit = vmx_handle_exit,
* vmx_handle_exit;
* return kvm_vmx_exit_handlers[exit_reason]  (vcpu);  调用注册的退出原因处理回调函数；
* 外部中断原因为1号，

```c
#define EXIT_REASON_EXTERNAL_INTERRUPT  1
static int (*kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = {
	[EXIT_REASON_EXCEPTION_NMI]           = handle_exception,
```

即调用handle_exception();

```c
u32 exit_reason = vmx->exit_reason;
return kvm_vmx_exit_handlers[exit_reason](vcpu);

static int (*kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = {
		[EXIT_REASON_EXTERNAL_INTERRUPT]      = handle_external_interrupt,
static int handle_external_interrupt(struct kvm_vcpu *vcpu)
{
	++vcpu->stat.irq_exits;
	return 1;
}
```

即vcpu_enter_guest（）返回了1；

退回到__vcpu_run() 继续往下走：

```c
	clear_bit(KVM_REQ_PENDING_TIMER, &vcpu->requests);
		if (kvm_cpu_has_pending_timer(vcpu))
			kvm_inject_pending_timer_irqs(vcpu);	
```

* kvm_cpu_has_pending_timer(vcpu))：判断是否有pending的timer
* atomic_read(&pit->pit_state.pit_timer.pending);   即时kvm_timer_fn()中置的pending位；

调用kvm_inject_pending_timer_irqs(vcpu); 进行注入；
```bash
->kvm_inject_pit_timer_irqs(vcpu);
-> __inject_pit_timer_intr(kvm);
-> kvm_set_irq(kvm, kvm->arch.vpit->irq_source_id, 0, 1);
-> irq_rt = rcu_dereference(kvm->irq_routing); 到中断路由表中找到irq0号对应的set函数
->r = irq_set[i].set(&irq_set[i], kvm, irq_source_id, level);
```
即在setup_routing_entry（）中注册的kvm_set_pic_irq（）
```bash
->kvm_set_pic_irq()
->kvm_pic_set_irq(pic, e->irqchip.pin, level);
->pic_update_irq() 
->s->irq_request(s->irq_request_opaque, 1);
->kvm_vcpu_kick
```
完成将虚拟时钟中断注入到内核模拟的虚拟PIC中.

由于vcpu_enter_guest返回1，所以不需要切换到用户模式，而是再次循环调用vcpu_enter_guest。

在kvm_x86_ops->run之前，调用inject_pending_event，它会将PIT中触发的中断真正地注入到vcpu中(通过调用kvm_x86_ops->set_irq)。

```c
inject_pending_event(vcpu);
->kvm_x86_ops->set_irq(vcpu);
.set_irq = vmx_inject_irq,
vmx_inject_irq（）将vmcs相应位按规范设置好即可
vmcs_write32(VM_ENTRY_INTR_INFO_FIELD, intr);
```

再次VMEntry时，硬件检测到虚拟时钟中断，然后调用guest OS timer interrupt handler进行处理。
对PIT 中断的ACK通过写寄存器完成，guest OS会发生IO访问，这是另一个话题: IO虚拟化。

## Conclusion
以上为中断虚拟化 code walk throght。