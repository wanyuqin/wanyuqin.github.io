---
title: GO调度器
 
categories: golang
comments: true
---



# GO调度器

<!--more-->



## **schedt**

保存调度器自身的状态信息，另一方面它还拥有一个用来保存goroutine的运行队列。

```go
type schedt struct {
   // accessed atomically. keep at top to ensure alignment on 32-bit systems.
   goidgen   uint64
   lastpoll  uint64 // time of last network poll, 0 if currently polling
   pollUntil uint64 // time to which current poll is sleeping

   lock mutex

   // When increasing nmidle, nmidlelocked, nmsys, or nmfreed, be
   // sure to call checkdead().
	 // 由空闲的工作线程组成链表
   midle        muintptr // idle m's waiting for work
   // 空闲的工作线程的数量
   nmidle       int32    // number of idle m's waiting for work
   nmidlelocked int32    // number of locked m's waiting for work
   mnext        int64    // number of m's that have been created and next M ID
   // 最多只能创建maxmcount个工作线程
   maxmcount    int32    // maximum number of m's allowed (or die)
   nmsys        int32    // number of system m's not counted for deadlock
   nmfreed      int64    // cumulative number of freed m's

   ngsys uint32 // number of system goroutines; updated atomically
   // 由空闲的p结构体对象组成的链表 
   pidle      puintptr // idle p's
   //  空闲的p结构体对象的数量
   npidle     uint32
   nmspinning uint32 // See "Worker thread parking/unparking" comment in proc.go.

   // Global runnable queue. goroutine全局运行队列
   runq     gQueue
   runqsize int32

   // disable controls selective disabling of the scheduler.
   //
   // Use schedEnableUser to control this.
   //
   // disable is protected by sched.lock.
   disable struct {
      // user disables scheduling of user goroutines.
      user     bool
      runnable gQueue // pending runnable Gs
      n        int32  // length of runnable
   }

   // Global cache of dead G's.
   // gFree是所有已经退出的goroutine对应的g结构体对象组成的链表
   // 用于缓存g结构体对象，避免每次创建goroutine时都重新分配内存
   gFree struct {
      lock    mutex
      stack   gList // Gs with stacks
      noStack gList // Gs without stacks
      n       int32
   }

   // Central cache of sudog structs.
   sudoglock  mutex
   sudogcache *sudog

   // Central pool of available defer structs.
   deferlock mutex
   deferpool *_defer

   // freem is the list of m's waiting to be freed when their
   // m.exited is set. Linked through m.freelink.
   freem *m

   gcwaiting  uint32 // gc is waiting to run
   stopwait   int32
   stopnote   note
   sysmonwait uint32
   sysmonnote note

   // safepointFn should be called on each P at the next GC
   // safepoint if p.runSafePointFn is set.
   safePointFn   func(*p)
   safePointWait int32
   safePointNote note

   profilehz int32 // cpu profiling rate

   procresizetime int64 // nanotime() of last change to gomaxprocs
   totaltime      int64 // ∫gomaxprocs dt up to procresizetime

   // sysmonlock protects sysmon's actions on the runtime.
   //
   // Acquire and hold this mutex to block sysmon from interacting
   // with the rest of the runtime.
   sysmonlock mutex

   // timeToRun is a distribution of scheduling latencies, defined
   // as the sum of time a G spends in the _Grunnable state before
   // it transitions to _Grunning.
   //
   // timeToRun is protected by sched.lock.
   timeToRun timeHistogram
}
```

## gobuf

gobuf结构体用于保存goroutine的调度信息，主要包括CPU的几个寄存器的值

```go
type gobuf struct {
   // The offsets of sp, pc, and g are known to (hard-coded in) libmach.
   //
   // ctxt is unusual with respect to GC: it may be a
   // heap-allocated funcval, so GC needs to track it, but it
   // needs to be set and cleared from assembly, where it's
   // difficult to have write barriers. However, ctxt is really a
   // saved, live register, and we only ever exchange it between
   // the real register and the gobuf. Hence, we treat it as a
   // root during stack scanning, which means assembly that saves
   // and restores it doesn't need write barriers. It's still
   // typed as a pointer so that any other writes from Go get
   // write barriers.
   sp   uintptr // CPU的rsp寄存器的值
   pc   uintptr // CPU的rip寄存器的值
   g    guintptr // 当前这个gobuf属于那个goroutine
   ctxt unsafe.Pointer
   ret  uintptr // 保存系统调用的返回值，因为从系统调用返回之后如果p被其它工作线程抢占，则这个goroutine会被放入全局运行队列被其它工作线程调度，其它线程需要知道系统调用的返回值。
   lr   uintptr
   bp   uintptr // for framepointer-enabled architectures 保存CPU的rip寄存器的值
}
```

## P

局部运行队列被包含在**p结构体**的实例对象之中，每一个运行着go代码的工作线程都会与一个p结构体的实例对象关联在一起

```go
type p struct {
   id          int32
   status      uint32 // one of pidle/prunning/...
   link        puintptr
   schedtick   uint32     // incremented on every scheduler call
   syscalltick uint32     // incremented on every system call
   sysmontick  sysmontick // last tick observed by sysmon
   m           muintptr   // back-link to associated m (nil if idle)
   mcache      *mcache
   pcache      pageCache
   raceprocctx uintptr

   deferpool    []*_defer // pool of available defer structs (see panic.go)
   deferpoolbuf [32]*_defer

   // Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
   goidcache    uint64
   goidcacheend uint64

   // Queue of runnable goroutines. Accessed without lock.
   //本地goroutine运行队列
   runqhead uint32  // 队列头
   runqtail uint32 // 队列尾
   runq     [256]guintptr //使用数组实现的循环队列
   // runnext, if non-nil, is a runnable G that was ready'd by
   // the current G and should be run next instead of what's in
   // runq if there's time remaining in the running G's time
   // slice. It will inherit the time left in the current time
   // slice. If a set of goroutines is locked in a
   // communicate-and-wait pattern, this schedules that set as a
   // unit and eliminates the (potentially large) scheduling
   // latency that otherwise arises from adding the ready'd
   // goroutines to the end of the run queue.
   //
   // Note that while other P's may atomically CAS this to zero,
   // only the owner P can CAS it to a valid G.
   runnext guintptr

   // Available G's (status == Gdead)
   gFree struct {
      gList
      n int32
   }

   sudogcache []*sudog
   sudogbuf   [128]*sudog

   // Cache of mspan objects from the heap.
   mspancache struct {
      // We need an explicit length here because this field is used
      // in allocation codepaths where write barriers are not allowed,
      // and eliminating the write barrier/keeping it eliminated from
      // slice updates is tricky, moreso than just managing the length
      // ourselves.
      len int
      buf [128]*mspan
   }

   tracebuf traceBufPtr

   // traceSweep indicates the sweep events should be traced.
   // This is used to defer the sweep start event until a span
   // has actually been swept.
   traceSweep bool
   // traceSwept and traceReclaimed track the number of bytes
   // swept and reclaimed by sweeping in the current sweep loop.
   traceSwept, traceReclaimed uintptr

   palloc persistentAlloc // per-P to avoid mutex

   _ uint32 // Alignment for atomic fields below

   // The when field of the first entry on the timer heap.
   // This is updated using atomic functions.
   // This is 0 if the timer heap is empty.
   timer0When uint64

   // The earliest known nextwhen field of a timer with
   // timerModifiedEarlier status. Because the timer may have been
   // modified again, there need not be any timer with this value.
   // This is updated using atomic functions.
   // This is 0 if there are no timerModifiedEarlier timers.
   timerModifiedEarliest uint64

   // Per-P GC state
   gcAssistTime         int64 // Nanoseconds in assistAlloc
   gcFractionalMarkTime int64 // Nanoseconds in fractional mark worker (atomic)

   // gcMarkWorkerMode is the mode for the next mark worker to run in.
   // That is, this is used to communicate with the worker goroutine
   // selected for immediate execution by
   // gcController.findRunnableGCWorker. When scheduling other goroutines,
   // this field must be set to gcMarkWorkerNotWorker.
   gcMarkWorkerMode gcMarkWorkerMode
   // gcMarkWorkerStartTime is the nanotime() at which the most recent
   // mark worker started.
   gcMarkWorkerStartTime int64

   // gcw is this P's GC work buffer cache. The work buffer is
   // filled by write barriers, drained by mutator assists, and
   // disposed on certain GC state transitions.
   gcw gcWork

   // wbBuf is this P's GC write barrier buffer.
   //
   // TODO: Consider caching this in the running G.
   wbBuf wbBuf

   runSafePointFn uint32 // if 1, run sched.safePointFn at next safe point

   // statsSeq is a counter indicating whether this P is currently
   // writing any stats. Its value is even when not, odd when it is.
   statsSeq uint32

   // Lock for timers. We normally access the timers while running
   // on this P, but the scheduler can also do it from a different P.
   timersLock mutex

   // Actions to take at some time. This is used to implement the
   // standard library's time package.
   // Must hold timersLock to access.
   timers []*timer

   // Number of timers in P's heap.
   // Modified using atomic instructions.
   numTimers uint32

   // Number of timerDeleted timers in P's heap.
   // Modified using atomic instructions.
   deletedTimers uint32

   // Race context used while executing timer functions.
   timerRaceCtx uintptr

   // scannableStackSizeDelta accumulates the amount of stack space held by
   // live goroutines (i.e. those eligible for stack scanning).
   // Flushed to gcController.scannableStackSize once scannableStackSizeSlack
   // or -scannableStackSizeSlack is reached.
   scannableStackSizeDelta int64

   // preempt is set to indicate that this P should be enter the
   // scheduler ASAP (regardless of what G is running on it).
   preempt bool

   // Padding is no longer needed. False sharing is now not a worry because p is large enough
   // that its size class is an integer multiple of the cache line size (for any of our architectures).
}
```

## G

为了实现对goroutine的调度，需要引入一个数据结构来保存CPU寄存器的值以及goroutine的其它一些状态信息，在Go语言调度器源代码中，这个数据结构是一个名叫**g的结构体**，它保存了goroutine的所有信息，该结构体的每一个实例对象都代表了一个goroutine，调度器代码可以通过g对象来对goroutine进行调度，当goroutine被调离CPU时，调度器代码负责把CPU寄存器的值保存在g对象的成员变量之中，当goroutine被调度起来运行时，调度器代码又负责把g对象的成员变量所保存的寄存器的值恢复到CPU的寄存器

```go
type g struct {
   // Stack parameters.
   // stack describes the actual stack memory: [stack.lo, stack.hi).
   // stackguard0 is the stack pointer compared in the Go stack growth prologue.
   // It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
   // stackguard1 is the stack pointer compared in the C stack growth prologue.
   // It is stack.lo+StackGuard on g0 and gsignal stacks.
   // It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
   stack       stack   // offset known to runtime/cgo 记录该G使用的栈
   // 下两个成员用于检查栈溢出，实现栈自动伸缩，抢占调度也会用到stackguard0
   stackguard0 uintptr // offset known to liblink
   stackguard1 uintptr // offset known to liblink

   _panic    *_panic // innermost panic - offset known to liblink
   _defer    *_defer // innermost defer
   m         *m      // current m; offset known to arm liblink 用来确定该当前的g正在被那个工作线程执行
   sched     gobuf // goroutine 相关的栈信息  调度信息 主要是几个寄存器的值
   syscallsp uintptr // if status==Gsyscall, syscallsp = sched.sp to use during gc
   syscallpc uintptr // if status==Gsyscall, syscallpc = sched.pc to use during gc
   stktopsp  uintptr // expected sp at top of stack, to check in traceback
   // param is a generic pointer parameter field used to pass
   // values in particular contexts where other storage for the
   // parameter would be difficult to find. It is currently used
   // in three ways:
   // 1. When a channel operation wakes up a blocked goroutine, it sets param to
   //    point to the sudog of the completed blocking operation.
   // 2. By gcAssistAlloc1 to signal back to its caller that the goroutine completed
   //    the GC cycle. It is unsafe to do so in any other way, because the goroutine's
   //    stack may have moved in the meantime.
   // 3. By debugCallWrap to pass parameters to a new goroutine because allocating a
   //    closure in the runtime is forbidden.
   param        unsafe.Pointer
   atomicstatus uint32
   stackLock    uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
   goid         int64
   schedlink    guintptr // schedlink字段指向全局运行队列中的下一个g，所有位于全局运行队列中的g形成一个链表
   waitsince    int64      // approx time when the g become blocked
   waitreason   waitReason // if status==Gwaiting

   preempt       bool // preemption signal, duplicates stackguard0 = stackpreempt 抢占调度标志，如果需要抢占调度，设置		preempt为true
   preemptStop   bool // transition to _Gpreempted on preemption; otherwise, just deschedule
   preemptShrink bool // shrink stack at synchronous safe point

   // asyncSafePoint is set if g is stopped at an asynchronous
   // safe point. This means there are frames on the stack
   // without precise pointer information.
   asyncSafePoint bool

   paniconfault bool // panic (instead of crash) on unexpected fault address
   gcscandone   bool // g has scanned stack; protected by _Gscan bit in status
   throwsplit   bool // must not split stack
   // activeStackChans indicates that there are unlocked channels
   // pointing into this goroutine's stack. If true, stack
   // copying needs to acquire channel locks to protect these
   // areas of the stack.
   activeStackChans bool
   // parkingOnChan indicates that the goroutine is about to
   // park on a chansend or chanrecv. Used to signal an unsafe point
   // for stack shrinking. It's a boolean value, but is updated atomically.
   parkingOnChan uint8

   raceignore     int8     // ignore race detection events
   sysblocktraced bool     // StartTrace has emitted EvGoInSyscall about this goroutine
   tracking       bool     // whether we're tracking this G for sched latency statistics
   trackingSeq    uint8    // used to decide whether to track this G
   runnableStamp  int64    // timestamp of when the G last became runnable, only used when tracking
   runnableTime   int64    // the amount of time spent runnable, cleared when running, only used when tracking
   sysexitticks   int64    // cputicks when syscall has returned (for tracing)
   traceseq       uint64   // trace event sequencer
   tracelastp     puintptr // last P emitted an event for this goroutine
   lockedm        muintptr
   sig            uint32
   writebuf       []byte
   sigcode0       uintptr
   sigcode1       uintptr
   sigpc          uintptr
   gopc           uintptr         // pc of go statement that created this goroutine
   ancestors      *[]ancestorInfo // ancestor information goroutine(s) that created this goroutine (only used if debug.tracebackancestors)
   startpc        uintptr         // pc of goroutine function
   racectx        uintptr
   waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
   cgoCtxt        []uintptr      // cgo traceback context
   labels         unsafe.Pointer // profiler labels
   timer          *timer         // cached timer for time.Sleep
   selectDone     uint32         // are we participating in a select and did someone win the race?

   // Per-G GC state

   // gcAssistBytes is this G's GC assist credit in terms of
   // bytes allocated. If this is positive, then the G has credit
   // to allocate gcAssistBytes bytes without assisting. If this
   // is negative, then the G must correct this by performing
   // scan work. We track this in bytes to make it fast to update
   // and check for debt in the malloc hot path. The assist ratio
   // determines how this corresponds to scan work debt.
   gcAssistBytes int64
}
```



## M

Go调度器源代码中还有一个用来代表工作线程的**m结构体**，每个工作线程都有唯一的一个m结构体的实例对象与之对应，m结构体对象除了记录着工作线程的诸如栈的起止位置、当前正在执行的goroutine以及是否空闲等等状态信息之外，还通过指针维持着与p结构体的实例对象之间的绑定关系

M存放在工作线程自身的本地存储中，算作工作线程的一个本地线程稀有的全局变量

```go
type m struct {
   // g0主要用来记录工作线程使用的栈信息，在执行调度代码时需要使用这个栈 
   // 执行用户goroutine代码时，使用用户goroutine自己的栈，调度时会发生栈的切换
   g0      *g     // goroutine with scheduling stack
   morebuf gobuf  // gobuf arg to morestack
   divmod  uint32 // div/mod denominator for arm - known to liblink
   _       uint32 // align next field to 8 bytes

   // Fields not known to debuggers.
   procid        uint64            // for debuggers, but offset not hard-coded
   gsignal       *g                // signal-handling g
   goSigStack    gsignalStack      // Go-allocated signal handling stack
   sigmask       sigset            // storage for saved signal mask
   // 通过TLS实现m结构体对象与工作线程之间的绑定 
   tls           [tlsSlots]uintptr // thread-local storage (for x86 extern register)
   mstartfn      func()
   //  指向工作线程正在运行的goroutine的g结构体对象
   curg          *g       // current running goroutine
   caughtsig     guintptr // goroutine running during fatal signal
   // 记录与当前工作线程绑定的p结构体对象
   p             puintptr // attached p for executing go code (nil if not executing go code)
   nextp         puintptr
   oldp          puintptr // the p that was attached before executing a syscall
   id            int64
   mallocing     int32
   throwing      int32
   preemptoff    string // if != "", keep curg running on this m
   locks         int32
   dying         int32
   profilehz     int32
   // spinning状态：表示当前工作线程正在试图从其它工作线程的本地运行队列偷取goroutine
   spinning      bool // m is out of work and is actively looking for work
   blocked       bool // m is blocked on a note
   newSigstack   bool // minit on C thread called sigaltstack
   printlock     int8
   incgo         bool   // m is executing a cgo call
   freeWait      atomic.Uint32 // Whether it is safe to free g0 and delete m (one of freeMRef, freeMStack, freeMWait)
   fastrand      uint64
   needextram    bool
   traceback     uint8
   ncgocall      uint64      // number of cgo calls in total
   ncgo          int32       // number of cgo calls currently in progress
   cgoCallersUse uint32      // if non-zero, cgoCallers in use temporarily
   cgoCallers    *cgoCallers // cgo traceback if crashing in cgo call
   // 没有goroutine需要运行时，工作线程睡眠在这个park成员上，
   // 其它线程通过这个park唤醒该工作线程
   park          note
   // 记录所有工作线程的一个链表
   alllink       *m // on allm
   schedlink     muintptr
   lockedg       guintptr
   createstack   [32]uintptr // stack that created this thread.
   lockedExt     uint32      // tracking for external LockOSThread
   lockedInt     uint32      // tracking for internal lockOSThread
   nextwaitm     muintptr    // next m waiting for lock
   waitunlockf   func(*g, unsafe.Pointer) bool
   waitlock      unsafe.Pointer
   waittraceev   byte
   waittraceskip int
   startingtrace bool
   syscalltick   uint32
   freelink      *m // on sched.freem

   // these are here because they are too large to be on the stack
   // of low-level NOSPLIT functions.
   libcall   libcall
   libcallpc uintptr // for cpu profiler
   libcallsp uintptr
   libcallg  guintptr
   syscall   libcall // stores syscall parameters on windows

   vdsoSP uintptr // SP for traceback while in VDSO call (0 if not in call)
   vdsoPC uintptr // PC for traceback while in VDSO call

   // preemptGen counts the number of completed preemption
   // signals. This is used to detect when a preemption is
   // requested, but fails. Accessed atomically.
   preemptGen uint32

   // Whether this is a pending preemption signal on this M.
   // Accessed atomically.
   signalPending uint32

   dlogPerM

   mOS

   // Up to 10 locks held by this m, maintained by the lock ranking code.
   locksHeldLen int
   locksHeld    [10]heldLockInfo
}
```

## stack

stack结构体主要用来记录goroutine所使用的栈的信息，包括栈顶和栈底位置：

```go
// runtime/runtime2.go
type stack struct {
   lo uintptr
   hi uintptr
}
```

## 重要的全局变量

```go
allm       *m  // 所有的m构成的一个链表，包括下面的m0
gomaxprocs int32  // p的最大值，默认等于ncpu，但可以通过GOMAXPROCS修改
ncpu       int32 // 系统中cpu核的数量，程序启动时由runtime代码初始化 
sched      schedt  // 调度器结构体对象，记录了调度器的工作状态
allp       []*p // 保存所有的p，len(allp) == gomaxprocs
allgs      []*g // 保存所有的g
m0           m  // 代表进程的主线程
g0           g  // m0的g0，也就是m0.g0 = &g0

```