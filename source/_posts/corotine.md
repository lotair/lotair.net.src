---
title: 协程简介
date: 2016-12-31 10:45:44
author: livexmm
tags:
  - corotine  
categories:
  - C/C++  
---

## 场景引入 ##
对于高性能服务端程序来说事件驱动模型是非常典型的模型，这个模型主要思路为注册事件处理函数，等待事件触发，事件触发后调用注册的处理函数，流畅较为清晰。但是当场景稍微复杂一点点，比如一个登陆操作，先验证登陆是否是攻击等行为的操作，然后访问缓存看是否存在，如果不存在就要访问数据库。如果要做到尽可能的把请求做得流水，也就是把IO操作相关时间尽量让出来，这个时候我们采用事件驱动模型。  
<!-- more -->

伪代码可能会如下的：  

	handleLoginEvent(logRequest) {
    	viaRequest = build_verify_if_attack_request(logRequest)  
    	verify_if_attack_request(viaRequest, callback(viaResponse) {
			if (if_pass(viaResponse)) {
				gucRequest = build_get_user_info_from_cache_request(loginRequest)
				get_user_info_from_cache(gucRequest, callback(gucResponse) {
					if (!if_got(gucResponse)) {
						gudRequest = build_get_user_info_from_db_request(loginRequest)
						get_user_info_from_db(gudRequest, callback(gudRequest) {
							// return pass or not pass info to client
						})
					} else {
						// return pass info to client
					}
				})
			} else {
				//some fail logic
			}
		})
  	}

这个代码是极其繁琐麻烦的，可以说是一种对机器友好，对人不太友好的方式。这个时候我们就要想有没有其他方式。当然显然是有的，比如我一个请求我一个线程，在线程中阻塞等待，这就以为的可能需要非常多的线程去做这个事情，而线程本省固有的代价（没个线程需要分配线程栈，当线程比较多的时候，这个内存代价是不小的），以及更重要的是线程数的增加直接导致上下文切换的代价增加，这不是我想看到的现象。那是否有存在同时满足更友好的编程体验，且内存CPU代价小的方案呢？其实有，也是这篇博客要介绍的协程(corotine)。

## 概念理论 ##
对于协程的概念，wikipedia解释为 “Coroutines are computer program components that generalize subroutines for nonpreemptive multitasking, by allowing multiple entry points for suspending and resuming execution at certain locations.” 大概意思是说协程是非抢占式多任务执行序（可以和抢占式多任务中的线程对比一下），有多个入口点（可以resume回去），支持在确定的位置挂起和唤醒。OK，从定义上来关键点是它不是抢占式的，它的切换是在确定的点主动让出的。  
协程按管理方式来分，分为对称和非对称2中，所谓对称的意味所有协程地位都是等价，可以在任意2个协程之间直接切换，而非对称的协程存在管理者，协程让出cpu资源总是回到管理者处，由其恢复不同的协程。  
协程按实现方式大的又分为2类，stackless和stackfull，stackless顾名思义协程并不具有栈，也就是无法定义局部变量，最著名的是Duff Device。而相反的stackfull是每个线程都有栈空间

## 不同实现方案 ##
下面我们简单介绍几种常见的协程实现方案：
### Duff Device ###
这种协程实现上面提到了属于stackless类型的协程，不能定义局部变量，好处是非常快，因为不需要保存栈等相关的信息。其利用了switch-case的trick来实现。具体可以参看[https://en.wikipedia.org/wiki/Duff's_device](https://en.wikipedia.org/wiki/Duff's_device)

### 共享栈协程 ###
这种协程采取在线程栈上为协程分配内存作为其协程栈，切换的时候保存恢复相关寄存器。这种方式有好处是切换速度很快，但缺点很明显可以分配的协程数很有限和线程栈最大内存相关，其次是一旦出现协程栈越界，问题很难查

### 复制栈协程 ###
这种协程采取当协程间发生切换，将会把当前的现场（相关寄存器信息）以及当前协程的栈内存复制出来存到堆上，再将需要切换的到的协程的栈的现场还原并把保存在堆上的栈信息恢复出来。这种方式好处很明显每个协程的栈空间能动态增长，不浪费内存，缺点也很明显每次切换代价大，需要复制栈的内容。典型的实现可以看python的greenlet库其实现了复制栈协程

### 独立栈协程 ###
这种协程采取为每个协程在堆上各自分配一块特定大小的内存空间作为栈，切换的时候只需要操作保存恢复相关寄存器信息即可。优缺点正好和复制栈协程相反，这个是切换快，但是每个的栈空间在一开始就定好了，不能动态改变。

## 让我们来实现协程库吧 ##

对于windows用户来说可以直接使用fiber，这里我们就不展开了。对于linux用户来说可以使用swapcontext相关的API实现，也比较简单。linux的swapcontext考了很多问题，可以用来实现用户态线程，支持每个处理序有可以处理不同的信号，所以在保存现场时还需要保存信号相关的信息，恢复时需要恢复相关的信息，而在恢复信号相关时，存在一把进程级的大锁。所以采用多个线程，每个线程多个协程的实现时，会出现锁竞争问题。当然有比较trick的方案绕过去，具体可以看郁白的[http://oceanbase.org.cn/?p=61](http://oceanbase.org.cn/?p=61)这篇文章。当然还有一种更加通用更加复杂的实现方案时自己操作寄存器，是的，下面我们将会实现的就是这种方式

## 系统ABI ##
自己操作寄存器，就要需要知道该平台函数调用的C规范问题，这是一个ABI问题。所以下面我们x64 linux平台为例进行简单的说明(下面的实现也会基于x64，linux)。首先摆在面前的一个问题是寄存器那么多，对于这个问题哪些我是需要的？（当然你保存所有的寄存器，总是没有错的，但是我想你不会那么无聊的，而且这是有代价的）首先我们需要保存当前函数的参数，调用规范明确表示C中基本类型（不含float/double）以及指针按参数顺序使用6个寄存器`RDI RSI RDX RCX R8 R9`，如果参数个数大于6个会使用栈，如果函数参数是有浮点数则会使用寄存器`XMM0-XMM7`。然后是需要使用的话，需要被调用方保存的寄存器是`RBX RBP R12 R13 R14 R15`。再然后当前的栈顶指针`RSP`以及指令寄存器`RIP`


## 具体实现 ##
我们参考简化glibc对swapcontext的实现，首先我们协程函数的原型我们参考pthread线程的原型定义为`void (*co_ctx_func_t)(void* argv)`,也就是只用`RDI`寄存器，但是下面我还是保存了`RDI RSI RDX RCX R8 R9`，但是没有保存更多到`RSP`以及浮点相关的，主要是觉得没有必要。下面简单讲一下实现，具体所有代码在[https://github.com/lotair/co](https://github.com/lotair/co)可以看到

	typedef struct co_ctx_t
	{
  		greg_t gregs[NGREG]; // 14个寄存器
  		struct co_ctx_t *uc_link; // 协程退出跳转到谁
  		void*  ss_sp;   // 栈
  		size_t ss_size; // 栈大小
  		int    finish;  // 栈是否已经结束
	} co_ctx_t;

	void co_ctx_make(co_ctx_t* ctx, co_ctx_func_t fn, void* argv) {
  		greg_t *sp;

		// 栈是向下生长的从高地址到低地址
  		sp = (greg_t *)((uintptr_t)ctx->ss_sp + ctx->ss_size);
  		sp -= 2;
  		sp = (greg_t *)((((uintptr_t)sp) & -16L) - 8);

  		ctx->gregs[REG_RIP] = (uintptr_t)fn;
  		ctx->gregs[REG_RBX] = (uintptr_t)&sp[1];
  		ctx->gregs[REG_RSP] = (uintptr_t)sp;

  		ctx->gregs[REG_RDI] = (uintptr_t)argv;

  		sp[0] = (uintptr_t)&__co_start;
  		sp[1] = (uintptr_t)ctx->uc_link;
  		sp[2] = (uintptr_t)&ctx->finish;
	}

	void co_switch(co_ctx_t* old_ctx, co_ctx_t* new_ctx) {
		// 各种保存到old_ctx
		// 各种从new_ctx恢复现场
 	}

	int main() {
		// ...
  		while (1) {
    		int done = 0;
    		for (int i = 0; i < UTHREAD_MAX_NUM; i++) {
      			if (ctx[i].finish == 0) {
        			done = 1;
        			co_switch(&ctx_main, ctx + i);
      			}
    		}
    		if (done == 0) break;
  		}
		// ...
	}
