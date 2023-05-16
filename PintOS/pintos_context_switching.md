 Context Switching은 현재 실행 중인 프로세스(스레드)의 상태 정보를 보관하고, 대기중인 프로세스(스레드)의 상태 정보를 읽어서 실행하는 작업을 말한다.
 
 이번 포스팅은 PintOS code 를 살펴보며 Context Switching이 어떤식으로 이뤄지는지 살펴보고자 한다.
 
 * 참고사항
 	__KAIST PintOS 베이스로 작성된 글입니다. 하지만 project 1 정답코드는 포함되어있지 않습니다.__

* 목차 
		1.Thread 메모리 구조
		2. Thread 구조체
        3. context switching
 	   	4. PintOS Context Switching 과정
  
  
  
# Thread 메모리 구조

![](https://velog.velcdn.com/images/mingyu/post/4bd1ed0d-0a38-4097-8eb8-328729473e06/image.png)


위 그림은 thread 메모리 구조를 도식화 한 것이다. 각 thread는 독립적인 kernel space 와 user space를 갖는다. 

* user space : user mode 에서 사용할 수 있는 공간으로 stack 영역이 존재한다.
* kernel space :커널 모드에서 실행되며, 시스템 자원에 직접적으로 접근할 수 있는 영역.

특히 kernel space 의 경우 Context Switching 가 밀접하게 연관 되어 있다. 그 이유는 Context Switching에 동작과정을 살펴보면 알 수 있다.


>현재 실행 중인 프로세스(스레드)의 상태 정보를 보관하고, 대기중인 프로세스(스레드)의 상태 정보를 읽어서 실행하는 작업을 말한다.

여기서 현재 실행 중인 thread의 상태 정보를 __보관__ 하는 곳이 바로 __kernel space이다!__

![](https://velog.velcdn.com/images/mingyu/post/d1d33a91-2876-47e2-885f-48e0e2e06511/image.png)


kernel space 에서 주목할 공간은 kernel stack이다.
thread의 정보는 kernel stack 가장 낮은 주소값에 저장되어, 높은 주소에서 낮은 주소로 grow 하는 kernel stack과 충돌을 방지한다.

충돌할 경우 thread 정보중 가장 높은 주소값에 저장된 magic이 변경되면서 thread가 stack over flow가 발생함을  알 수 있다.

* tip : 만약 thread 구조체의 너무 큰 size를 할당하게 된다면 kernel stack이 사용할 수 있는 영역이 줄어들고, 충돌이 일어날 수 있음을 주의해야한다.



# Pintos Thread 구조체

```c
struct thread {
	/* Owned by thread.c. */
	tid_t tid;                          /* Thread identifier. */
	enum thread_status status;          /* Thread state. */
	char name[16];                      /* Name (for debugging purposes). */
	int priority;                       /* Priority. */
	int origin_priority;

	/* Shared between thread.c and synch.c. */
	struct list_elem elem;              /* List element. */
	int64_t wake_time;

	/* Owned by thread.c. */
	struct intr_frame tf;               /* Information for switching */
	unsigned magic;                     /* Detects stack overflow. */
};
```

위는 pintOS thread 구조체 이며, 우리가 중점적으로 살펴볼 interrupt frame(intr_frame) 이다.



```c
  struct intr_frame {
	/* Pushed by intr_entry in intr-stubs.S.
	   These are the interrupted task's saved registers. */
	struct gp_registers R;
	uint16_t es;
	uint16_t __pad1;
	uint32_t __pad2;
	uint16_t ds;
	uint16_t __pad3;
	uint32_t __pad4;
	/* Pushed by intrNN_stub in intr-stubs.S. */
	uint64_t vec_no; /* Interrupt vector number. */
/* Sometimes pushed by the CPU,
   otherwise for consistency pushed as 0 by intrNN_stub.
   The CPU puts it just under `eip', but we move it here. */
	uint64_t error_code;
/* Pushed by the CPU.
   These are the interrupted task's saved registers. */
	uintptr_t rip;
	uint16_t cs;
	uint16_t __pad5;
	uint32_t __pad6;
	uint64_t eflags;
	uintptr_t rsp;
	uint16_t ss;
	uint16_t __pad7;
	uint32_t __pad8;
} __attribute__((packed));

struct gp_registers {
	uint64_t r15;
	uint64_t r14;
	uint64_t r13;
	uint64_t r12;
	uint64_t r11;
	uint64_t r10;
	uint64_t r9;
	uint64_t r8;
	uint64_t rsi;
	uint64_t rdi;
	uint64_t rbp;
	uint64_t rdx;
	uint64_t rcx;
	uint64_t rbx;
	uint64_t rax;
} __attribute__((packed));
```
> Interrupt frame : 인터럽트가 발생했을때 현재 thread의 context를 저장하는 구조체

위 코드를 살펴보면 intr_frame 구조체 멤버들이 __CPU 레지스터__의 대응되는 변수명을 갖고있음을 알 수 있다. 즉 thread 구조체 안에 CPU 레지스터값(context)를 저장 할 수 있다!

__interrupt frame 구조체가 thread 의 context를 의미하고, kernel space에 해당 값을 저장 하고 읽는 과정이 context switching 이라고 할 수 있겠다.__


# Context Switching
> 현재 실행 중인 프로세스(스레드)의 상태 정보를 보관하고, 대기중인 프로세스(스레드)의 상태 정보를 읽어서 실행하는 작업을 말한다.

context switching 과정을 살펴보도록 하자.

![](https://velog.velcdn.com/images/mingyu/post/b5589dd6-eac3-4598-b71e-936594b30271/image.png)



OS의 scheduling으로 인해 context switching이 일어나는 상황이다. 현재 실행중인 thread를 current thread, context switching 되어 실행될 thread가 next thread 이다. (각 thread의 그림은 thread의 kernel stack 도식화 하였다.)

CPU는 현재 thread(current thread)를 실행중이다. 즉 CPU 레지스터 값이 실행 중인 thread의 context를 의미한다. 

![](https://velog.velcdn.com/images/mingyu/post/ebd324a3-181b-4775-922e-61d3f8fd5bf6/image.png)



CPU에서 현재 thread의 context를 current thread의 interrupt frame의 저장한다. 

![](https://velog.velcdn.com/images/mingyu/post/ca438b0e-c335-47d1-a2e3-76da21b4e403/image.png)



다음 실행될 thread(next thread)의 context(next thread 구조체의 저장된 interrupt frame)를 CPU로 옮긴다.

과정을 마치면 CPU는 자연스럽게 next thread를 실행하게 된다. 


# PintOS Context Switching 과정

먼저 Context Switching 일어나는 시나리오를 설정하고. 코드를 살펴보도록 하자.

1)timer 가 timer interrupt를 호출한다.
2)thread ticks가 정해진 TIME SLICE 이상이 되어 context switching이 발생한다.

* scheduling은 라운드 로빈 방식으로 설정한다.


```c
static void
timer_interrupt (struct intr_frame *args UNUSED) {
	ticks++;
	thread_tick();
	thread_wake(ticks);
}
```

```c
void
thread_tick (void) {
	struct thread *t = thread_current ();

	/* Update statistics. */
	if (t == idle_thread)
		idle_ticks++;
	else
		kernel_ticks++;

	/* Enforce preemption. */
	if (++thread_ticks >= TIME_SLICE)
		intr_yield_on_return ();
}
```

1. timer 에서 ticks 마다 timer intterupt 를 호출하여(timer intterupt)
2. 현재 thread ticks가 TIME SLICE 이상인지 확인한다(thread_ticks)

```c
void
intr_yield_on_return (void) {
	ASSERT (intr_context ());
	yield_on_return = true;
}
```

3. TIME SLICE 이상이면 yield_on_return 값이 true로 반환된다.

```c
void
intr_handler (struct intr_frame *frame) {
	bool external;
	intr_handler_func *handler;

	/* External interrupts are special.
	   We only handle one at a time (so interrupts must be off)
	   and they need to be acknowledged on the PIC (see below).
	   An external interrupt handler cannot sleep. */
	external = frame->vec_no >= 0x20 && frame->vec_no < 0x30;
	if (external) {
		ASSERT (intr_get_level () == INTR_OFF);
		ASSERT (!intr_context ());

		in_external_intr = true;
		yield_on_return = false;
	}

	/* Invoke the interrupt's handler. */
	handler = intr_handlers[frame->vec_no];
	if (handler != NULL)
		handler (frame);
	else if (frame->vec_no == 0x27 || frame->vec_no == 0x2f) {
		/* There is no handler, but this interrupt can trigger
		   spuriously due to a hardware fault or hardware race
		   condition.  Ignore it. */
	} else {
		/* No handler and not spurious.  Invoke the unexpected
		   interrupt handler. */
		intr_dump_frame (frame);
		PANIC ("Unexpected interrupt");
	}

	/* Complete the processing of an external interrupt. */
	if (external) {
		ASSERT (intr_get_level () == INTR_OFF);
		ASSERT (intr_context ());

		in_external_intr = false;
		pic_end_of_interrupt (frame->vec_no);

		if (yield_on_return)
			thread_yield ();
	}
}
```


interrupt handler 에서 timer_intterupt는 external 이다.

```
	if (external) {
		ASSERT (intr_get_level () == INTR_OFF);
		ASSERT (intr_context ());

		in_external_intr = false;
		pic_end_of_interrupt (frame->vec_no);

		if (yield_on_return)
			thread_yield ();
```

4. external && yield_on_return 임으로 thread_yield()실행

5. thread_yield()가 schedule()함수 실행
```c
static void	
schedule (void) {
	struct thread *curr = running_thread ();
	struct thread *next = next_thread_to_run ();

	ASSERT (intr_get_level () == INTR_OFF);
	ASSERT (curr->status != THREAD_RUNNING);
	ASSERT (is_thread (next));
	/* Mark us as running. */
	next->status = THREAD_RUNNING;

	/* Start new time slice. */

	thread_ticks = 0;

#ifdef USERPROG
	/* Activate the new address space. */
	process_activate (next);
#endif

	if (curr != next) {
		/* If the thread we switched from is dying, destroy its struct
		   thread. This must happen late so that thread_exit() doesn't
		   pull out the rug under itself.
		   We just queuing the page free reqeust here because the page is
		   currently used by the stack.
		   The real destruction logic will be called at the beginning of the
		   schedule(). */
		if (curr && curr->status == THREAD_DYING && curr != initial_thread) {
			ASSERT (curr != next);
			list_push_back (&destruction_req, &curr->elem);
		}

		/* Before switching the thread, we first save the information
		 * of current running. */
		thread_launch (next);
	}
}
```

6. schedule() 함수로 다음 실행될 thread(next)가 정해지고 thread_launch(next) 호출

```c
static void
thread_launch (struct thread *th) {
	uint64_t tf_cur = (uint64_t) &running_thread ()->tf;
	uint64_t tf = (uint64_t) &th->tf;
	ASSERT (intr_get_level () == INTR_OFF);

	/* The main switching logic.
	 * We first restore the whole execution context into the intr_frame
	 * and then switching to the next thread by calling do_iret.
	 * Note that, we SHOULD NOT use any stack from here
	 * until switching is done. */
	__asm __volatile (
			/* Store registers that will be used. */
			"push %%rax\n" //tf_cur
			"push %%rbx\n" 
			"push %%rcx\n"
			/* Fetch input once */
			"movq %0, %%rax\n" //%0은 첫번쨰 인자 tf_cur
			"movq %1, %%rcx\n" //%1은 두번째 인자 tf
			"movq %%r15, 0(%%rax)\n"
			"movq %%r14, 8(%%rax)\n"
			"movq %%r13, 16(%%rax)\n"
			"movq %%r12, 24(%%rax)\n"
			"movq %%r11, 32(%%rax)\n"
			"movq %%r10, 40(%%rax)\n"
			"movq %%r9, 48(%%rax)\n"
			"movq %%r8, 56(%%rax)\n"
			"movq %%rsi, 64(%%rax)\n"
			"movq %%rdi, 72(%%rax)\n"
			"movq %%rbp, 80(%%rax)\n"
			"movq %%rdx, 88(%%rax)\n"
			"pop %%rbx\n"              // Saved rcx
			"movq %%rbx, 96(%%rax)\n"
			"pop %%rbx\n"              // Saved rbx
			"movq %%rbx, 104(%%rax)\n"
			"pop %%rbx\n"              // Saved rax
			"movq %%rbx, 112(%%rax)\n"
			"addq $120, %%rax\n"
			"movw %%es, (%%rax)\n"
			"movw %%ds, 8(%%rax)\n"
			"addq $32, %%rax\n"
			"call __next\n"         // read the current rip.
			"__next:\n"
			"pop %%rbx\n"
			"addq $(out_iret -  __next), %%rbx\n"
			"movq %%rbx, 0(%%rax)\n" // rip
			"movw %%cs, 8(%%rax)\n"  // cs
			"pushfq\n"
			"popq %%rbx\n"
			"mov %%rbx, 16(%%rax)\n" // eflags
			"mov %%rsp, 24(%%rax)\n" // rsp
			"movw %%ss, 32(%%rax)\n"
			"mov %%rcx, %%rdi\n"
			"call do_iret\n"
			"out_iret:\n"
			: : "g"(tf_cur), "g" (tf) : "memory"
			);
}
```

* tf_cur 은 현재 thread의 intr_frame
* tf 는 next thread의 intr_freme

7. 현재 thread context를 tf_cur에 저장한다.

8. do_iret함수를 호출한다.

```c
void
do_iret (struct intr_frame *tf) {
	__asm __volatile(
			"movq %0, %%rsp\n"
			"movq 0(%%rsp),%%r15\n"
			"movq 8(%%rsp),%%r14\n"
			"movq 16(%%rsp),%%r13\n"
			"movq 24(%%rsp),%%r12\n"
			"movq 32(%%rsp),%%r11\n"
			"movq 40(%%rsp),%%r10\n"
			"movq 48(%%rsp),%%r9\n"
			"movq 56(%%rsp),%%r8\n"
			"movq 64(%%rsp),%%rsi\n"
			"movq 72(%%rsp),%%rdi\n"
			"movq 80(%%rsp),%%rbp\n"
			"movq 88(%%rsp),%%rdx\n"
			"movq 96(%%rsp),%%rcx\n"
			"movq 104(%%rsp),%%rbx\n"
			"movq 112(%%rsp),%%rax\n"
			"addq $120,%%rsp\n"
			"movw 8(%%rsp),%%ds\n"
			"movw (%%rsp),%%es\n"
			"addq $32, %%rsp\n"
			"iretq"
			: : "g" ((uint64_t) tf) : "memory");
}
```
9. next thread의 tf를 복원해 context swithcing 이 일어난다.



# 요약

* context switching

1. thread는 자신의 정보를 kernel space에 저장한다
2. context switching 일어나면 current thread의 정보를 current thread 의 kernel space에 저장한다.
3. next thread의 kernle space에서 thread 정보를 불러온다.
4. next thread가 실행되면서 context switching 완료.



  










    
    