# Go语言goroutine调度器初始化

* OS centos7.9
* go version go1.17 linux/amd64      





## 新建文件

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello World!")
}
```

1. go build main.go
2. gdb main
   1. info files

```shell
(gdb) info files
Symbols from "/root/go/wanyuqin/demo1/main".
Local exec file:
        `/root/go/wanyuqin/demo1/main', file type elf64-x86-64.
        Entry point: 0x45c1a0
        0x0000000000401000 - 0x000000000047e247 is .text
        0x000000000047f000 - 0x00000000004b3eab is .rodata
        0x00000000004b4040 - 0x00000000004b4518 is .typelink
        0x00000000004b4520 - 0x00000000004b4578 is .itablink
        0x00000000004b4578 - 0x00000000004b4578 is .gosymtab
        0x00000000004b4580 - 0x000000000050cd60 is .gopclntab
        0x000000000050d000 - 0x000000000050d020 is .go.buildinfo
        0x000000000050d020 - 0x000000000051d5e0 is .noptrdata
        0x000000000051d5e0 - 0x0000000000524df0 is .data
        0x0000000000524e00 - 0x0000000000553d08 is .bss
        0x0000000000553d20 - 0x0000000000559080 is .noptrbss
        0x0000000000400f9c - 0x0000000000401000 is .note.go.buildid
        
```

查看 runtime/rt0_linux_amd64.s     

```assembly
     1  // Copyright 2009 The Go Authors. All rights reserved.
     2  // Use of this source code is governed by a BSD-style
     3  // license that can be found in the LICENSE file.
     4
     5  #include "textflag.h"
     6
     7  TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
     8          JMP     _rt0_amd64(SB)  // 主线程第一条指令
     9
    10  TEXT _rt0_amd64_linux_lib(SB),NOSPLIT,$0
    11          JMP     _rt0_amd64_lib(SB)
```

_rt0_amd64(SB) 在 runtime/asm_amd64.s 文件中：

```assembly
// _rt0_amd64 is common startup code for most amd64 systems when using
// internal linking. This is the entry point for the program from the
// kernel for an ordinary -buildmode=exe program. The stack holds the
// number of arguments and the C-style argv.
TEXT _rt0_amd64(SB),NOSPLIT,$-8
        MOVQ    0(SP), DI       // argc 将操作系统内核传递过来的参数放在DI寄存器
        LEAQ    8(SP), SI       // argv 将操作系统内核传递过来的参数放在SI寄存器
        JMP     runtime·rt0_go(SB) // 跳到rt0_go去执行
```

* argc 表示参数数量
* argv 表示一个指向指向字符串数组的指针，每个字符串表示一个命令行参数。第一个元素 `argv[0]` 存储的是程序的名称，其余元素存储的是命令行参数

 

rt0_go(SB) 完成go程序初始化的所有工作

```assembly
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
        // copy arguments forward on an even stack
        MOVQ    DI, AX          // argc AX = argc
        MOVQ    SI, BX          // argv BX = argv
        SUBQ    $(4*8+7), SP            // 2args 2auto
        ANDQ    $~15, SP // 调整栈顶寄存器按照16字节对齐
        MOVQ    AX, 16(SP) //  argc 放在SP+16字节处
        MOVQ    BX, 24(SP) // argv 放在SP+24字节处
```

## 初始化g0

```assembly
// create istack out of the given (operating system) stack.
// _cgo_init may update stackguard.
//下面这段代码从系统线程的栈空分出一部分当作g0的栈，然后初始化g0的栈信息和stackgard
MOVQ    $runtime·g0(SB), DI // g0的地址放入DI寄存器
LEAQ    (-64*1024+104)(SP), BX
MOVQ    BX, g_stackguard0(DI)
MOVQ    BX, g_stackguard1(DI)
MOVQ    BX, (g_stack+stack_lo)(DI)
MOVQ    SP, (g_stack+stack_hi)(DI)
```

## 主线程与m0绑定

runtime/asm_amd64.s

```shell
// 下面开始初始化tls(thread local storage,线程本地存储)
LEAQ    runtime·m0+m_tls(SB), DI  //DI = &m0.tls，取m0的tls成员的地址到DI寄存器
CALL    runtime·settls(SB) //调用settls设置线程本地存储，settls函数的参数在DI寄存器中

// store through it, to make sure it works
//验证settls是否可以正常工作，如果有问题则abort退出程序
get_tls(BX)  //获取fs段基地址并放入BX寄存器，其实就是m0.tls[1]的地址，get_tls的代码由编译器生成
MOVQ    $0x123, g(BX)  //把整型常量0x123拷贝到fs段基地址偏移-8的内存位置，也就是m0.tls[0] = 0x123
MOVQ    runtime·m0+m_tls(SB), AX //AX = m0.tls[0]
CMPQ    AX, $0x123 //检查m0.tls[0]的值是否是通过线程本地存储存入的0x123来验证tls功能是否正常
JEQ 2(PC)
CALL    runtime·abort(SB) //如果线程本地存储不能正常工作，退出程序
```



