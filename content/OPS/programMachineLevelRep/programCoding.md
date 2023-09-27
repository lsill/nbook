---
title: "程序编码"
date: 2023-09-07T16:00:48+08:00
draft: true
---

### 1. 机器级代码
- 程序计数器(PC) （在x86中用%rip表示），给出将要执行下一条指定在内存中的地址
- 整数寄存器 文件包含16个命名的位置，分别存储64位的值。这些寄存器可以存储地址（对应于c语言的指针）或整数数据。
  有的寄存器被用来记录某些重要的程序状态，而其他的寄存器用来保存临时数据，例如过程的参数和局部的变量以及函数的返回值。
- 条件码寄存器 保存着最近执行的算术或者逻辑指令的状态信息。它们用来实现控制或数据流中的条件变化，比如说用来实现if和while语句。
- 一组向量寄存器可以存放一个或多个整数或者浮点数值。

程序内存包含：
1. 程序的可执行机器代码
2. 操作系统需要的一些信息
3. 用来管理过程调用和函数的运行时栈
4. 用户分配的内存块（比如malloc）


x86-64 的虚拟地址是有64位的字表示的，在目前的实现中，这些地址的高16位必须设置成0，所以一个地址实际上能够指定的是2^48或64TB范围内的一个字节。比较典型的是程序只会访问几兆或几千兆字节的数据

操作系统负责管理虚拟地址空间，将虚拟地址空间翻译成实际处理器内存中的物理地址。

```c
long mult2(long, long);

void mulstore(long x, long y, long *dest) {
    long t = mult2(x, y);
    *dest = t;
}
```
linux>gcc -Og -S mstore.c
```
mulstore:
.LFB0:
        .cfi_startproc
        pushq   %rbx  // 寄存器将%rbx的内容压入程序栈中
        .cfi_def_cfa_offset 16
        .cfi_offset 3, -16
        movq    %rdx, %rbx
        call    mult2
        movq    %rax, (%rbx)
        popq    %rbx
        .cfi_def_cfa_offset 8
        ret
        .cfi_endproc
```
如果使用-c命令，gcc会编译并汇编该代码
linux>gcc -Og -c mstore.c
会产生目标代码文件mstore.o，它是二进制格式，所以无法直接查看

要查看机器代码文件的内容，有一类称为反汇编器的程序非常有用。这些程序根据机器代码产生的一种类似汇编代码的格式。在linux系统中，
带-d的命令标志objdump可以充当这个角色:
linux> objdump -d mstore.o
解锁如下
```
[root@demo2 procodeing]# objdump -d mstore.o

mstore.o：     文件格式 elf64-x86-64


Disassembly of section .text:

0000000000000000 <mulstore>:
   0:	53                   	push   %rbx
   1:	48 89 d3             	mov    %rdx,%rbx
   4:	e8 00 00 00 00       	callq  9 <mulstore+0x9>
   9:	48 89 03             	mov    %rax,(%rbx)
   c:	5b                   	pop    %rbx
   d:	c3                   	retq
```
在左边看到前面给出的字节顺序排列的14个进制字节值，它们分成了若干组，每组1~5个字节。每组都是一条指令，右边是等价的汇编语言。
关于机器代码和它的反汇编表示的特性值注意：
- x86-64的指令长度从1到15个字节不等。常用的指令以及操作较少的指令所需的字节少，而那些不太常用或者操作数较多的指令所需的字节数较多。
- 设计指令格式的方式是，从某个给定位置开始，可以将字节唯一的解码成机器指令。例如，只有pushq %rbx是以字节值53开头的。
- 反汇编器只是给予机器代码文件中的字节序列来确定汇编代码。它不需要访问该程序的源代码或汇编代码。
- 反汇编器使用的指令命令规则与gcc生成的汇编代码使用的有些细微差别。在实例中，它省略了很多指令结果的'q'。这些后缀是大小指示符，在大多数情况中可以省略。
  相反，反汇编给call和ret指令添加了'q'后缀，同样，省略这些后缀也没有问题.

生成实际可执行的代码需要对一组目标代码文件运行链接器，而这一组目标文件中必须含有一个main函数
```c
#include<stdio.h>

void multstore(long,long,long*);

int main() {
        long d;
        multstore(2,3,&d);
        printf("2 * 3 --> %ld\n",d);
        return 0;
}

long mult2(long a,long b) {
        long s = a * b;
        return s;
}
```
linux> gcc -Og -0 prog main.c mstore.c
反会半期会抽取各种代码序列，包含下面这段：
```
00000000004005d4 <multstore>:
  4005d4:	53                   	push   %rbx
  4005d5:	48 89 d3             	mov    %rdx,%rbx
  4005d8:	e8 ef ff ff ff       	callq  4005cc <mult2>
  4005dd:	48 89 03             	mov    %rax,(%rbx)
  4005e0:	5b                   	pop    %rbx
  4005e1:	c3                   	retq
  4005e2:	66 2e 0f 1f 84 00 00 	nopw   %cs:0x0(%rax,%rax,1)
  4005e9:	00 00 00
  4005ec:	0f 1f 40 00          	nopl   0x0(%rax)
```
这段反汇编产生的代码几乎完全一样，
- 其中一个主要的区别是左边列出的地址不同————丽娜姐器将这段代码的地址移到了一段不同的地址范围中，
- 第二个不同在于链接器填上了callq指令调用函数mult2需要的地址。
  **链接器** 的任务之一就是为了函数调用找到匹配的函数的可执行代码的位置
- 最后一个区别是多了两行代码，这两条指令对程序没有影响，应为它们出现在返回执行后面。插入这些指令
  是为了是函数代码变为16字节，使得就操作系统性能而言，能更好的防止下一代码块

### 2. 格式注解
mstore.s 完整内容：
```
        .file   "mstore.c"
        .text
        .globl  multstore
        .type   multstore, @function
multstore:
.LFB0:
        .cfi_startproc
        pushq   %rbx 
        .cfi_def_cfa_offset 16
        .cfi_offset 3, -16
        movq    %rdx,
        call    mult2
        movq    %rax, (%rbx)
        popq    %rbx
        .cfi_def_cfa_offset 8
        ret
        .cfi_endproc
.LFE0:
        .size   multstore, .-multstore
        .ident  "GCC: (GNU) 8.5.0 20210514 (Red Hat 8.5.0-4)"
        .section        .note.GNU-stack,"",@progbits
```
所有'.'开头的行都是指导汇编器和链接器工作的伪指令。（通常可以忽略）
```
void multstore(long x,long y, long *dest)
x in %rdi, y in %rsi, dest in %rdx

multstore:
        pushq   %rbx  保存*dest到%rbx中（栈压入）
        movq    %rdx, 复制*dest到rbx中
        call    mult2 调用函数mult2
        movq    %rax, (%rbx)  存储结果到*dest
        popq    %rbx  恢复rbx (栈弹出)
        ret 返回
```

### 3. 数据格式
| c声明    | inter数据类型 | 汇编代码后缀 | 大小（字节） |
|--------|-----------|--------|--------|
| char   | 字节        | b      | 1      |
| short  | 字         | w      | 2      |
| int    | 双字        | l      | 4      |
| long   | 四字        | q      | 8      |
| char*  | 四字        | q      | 8      |
| float  | 单精度       | s      | 4      |
| double | 双精度       | l      | 8      |


### 4. 访问信息
一个x86-64的cpu包含一组16个存储64位值的通用目的寄存器，这些寄存器用来存储整数数据和指针。
16位：%ax-%bp
32位:%eax-%ebp
64位:%rax-%rbp 还增加了8个新的寄存器%r8-%r15

| 63   | 31    | 15    | 7     | 0      |
|------|-------|-------|-------|--------|
| %rax | %eax  | %ax   | %al   | 返回值    |
| %rbx | %ebx  | %bx   | %bl   | 被调用者保存 |
| %rcx | %ecx  | %cx   | %cl   | 第4个参数  |
| %rdx | %edx  | %dx   | %dl   | 第三个参数  |
| %rsi | %esi  | %si   | %sil  | 第二个参数  |
| %rdi | %edi  | %di   | %dil  | 第一个参数  |
| %rbp | %ebp  | %bp   | &bpl  | 被调用者保存 |
| %rsp | %esp  | %sp   | %spl  | 栈指针    |
| %r8  | %r8d  | %r8w  | %r8b  | 第五个参数  |
| %r9  | %r9d  | %r9w  | %r9b  | 第六个参数  |
| %r10 | %r10d | %r10w | %r10b | 调用者保存  |
| %r11 | %r11d | %r11w | %r11b | 调用者保存  |
| %r12 | %r12d | %r12w | %r12b | 被调用者保存 |
| %r13 | %r13d | %r13w | %r13b | 被调用者保存 |
| %r14 | %r14d | %r14w | %r14b |     被调用者保存   |
| %r15 | %r15d | %r15w | %r15b |   被调用者保存     |
整数寄存器，所有16个寄存器的低位部分都可以作为字节、字、双字和四字数字来访问。

#### 1.操作数指示符
大多数指令有一个活多个操作数，指示出执行一个操作中要使用的元数据值，以及放置结果的目标位置。
源数据值可以以常数形式给出、或是从寄存器或内存中读出。
结果可以存放在寄存器或内存中。

有不同的`寻址模式`，允许不同形式的内存引用，Imm(rb,ri,s)表示的是最常用的形式，这样的引用有四个组成部分：
一个立即数便宜Imm,一个基址寄存器rb，一个变址寄存器ri和一个比例因子s，这里s必须是1，2，4或者8.
基址和变址寄存器都必须是64位寄存器。
有效地址：`Imm+R[rb]+R[ri]*s`。引用数组元素时会用到这种通用形式。其他形式都是这种通用形式的特殊情况，
只是省略了某些部分。

| 类型  | 格式           | 操作数值                 | 名称         |
|-----|--------------|----------------------|------------|
| 立即数 | $Imm         | Imm                  | 立即数寻址      |
| 寄存器 | ra           | R[ra]                | 寄存器寻址      |
| 存储器 | Imm          | M[Imm]               | 绝对寻址       |
| 存储器 | (ra)         | M[R[ra]]             | 间接寻址       |
| 存储器 | Imm(rb)      | M[Imm+R[rb]]         | （基址+偏移量）寻址 |
| 存储器 | (rb,ri)      | M[R[rb]+R[ri]]       | 变址寻址       |
| 存储器 | Imm(rb, ri)  | M[Imm + R[rb]+R[ri]] | 变址寻址       |
| 存储器 | (,ri,s)      | M[R[ri]*s]           | 比例变址寻址     |
| 存储器 | Imm(,ri,s)   | M[Imm+R[ri]*s]       | 比例变址寻址     |
| 存储器 | (rb,ri,s)    | M[R[rb]+R[ri]*s]     | 比例变址寻址     |
| 存储器 | Imm(rb,ri,s) | M[Imm+R[rb]+R[ri]*s] | 比例变址寻址     |

#### 2.数据传送指令mov
数据传输指令是使用最频繁的指令。

| 指令          | 效果   | 描述     |
|-------------|------|--------|
| MOV  S,D    | S->D | 传送     |
| movb        |      | 传送字节   |
| movw        |      | 传送字    |
| movl        |      | 传送双字   |
| movq        |      | 传送四字   |
| movabsq I,R | I->R | 传送绝对四字 |

- 源操作数指定的值是一个立即数，存储在寄存器或者内存中
- 目的操作室指定的一个位置，要么是一个寄存器，要么是个内存地址
- x86-64传送指令的两个操作数不能都指向内存位置
  将一个值从内存位置复制到另一个内存位置需要两条指令
  1. 将源值加载到寄存器中
  2. 将寄存器值写入目的内存值中
- 指令的寄存器操作数可以使16个寄存器有标号部分中的任意一个
- 寄存器部分的大小必须与指令最后一个字符(b,w,l,q)指定的大小匹配
- 大多数情况中mov指令只会更新目的操作数指定的那些寄存器字节或内存位置，唯一
  例外是movl指令以寄存器作为目的时，它会把该寄存器的高4字节设置成0。造成这个
  例外的原因是x86-64的惯例————任何寄存器生成32位值的指令都会把该寄存器的高位
  不分置成0
- 常规的movq指令只能以表示为32位补码数字的立即数作为源操作数，然后把这个值符号
  扩展到64位的值，放到目的位置。movabsq指令能够以任意64位立即数值作为源操作室，
  并且只能以寄存器作为目的。

将较小的源值复制到较大的目的时使用两类数据移动指定。所有这些指令都把数据从源（在寄存器或内存中）
复制到目的寄存器。
- movz类中的指令把目的中剩余的字节填充位0
- movs类中的指令通过符号来填充，把源操作的最高位进行复制。
  例如：movsbq $AA,%rax  %rax = FFFFFFFFFFFFFAA (A = 1010 符号位是1)

每条指令名字的最后两个字符都是大小指示符：第一个字符指定源的大小，第二个指明目的的大小。

| 指令        | 效果        | 描述             |
|-----------|-----------|----------------|
| MOVZ  S,R | 零扩展(S)->R | 以零扩展进行传送       |
| movzbw    |           | 将做了零扩展的字节传送到字  |
| movzbl    |           | 将做了零扩展的字节传送到双字 |
| movzwl    |           | 将做了零扩展的字传送到双字  |
| movzbq    |           | 将做了零扩展的字节传送到四字 |
| movzwq    |           | 将做了零扩展的字传送到四字  |

零扩展数据传送指令：这些指令以寄存器或内存地址作为源，以寄存器为目的。

| 指令        | 效果               | 描述              |
|-----------|------------------|-----------------|
| MOVS  S,R | 符号扩展扩展(S)->R     | 以零扩展进行传送        |
| movsbw    |                  | 将做了符号扩展的字节传送到字  |
| movsbl    |                  | 将做了符号扩展的字节传送到双字 |
| movswl    |                  | 将做了符号扩展的字传送到双字  |
| movsbq    |                  | 将做了符号扩展的字节传送到四字 |
| movswq    |                  | 将做了符号扩展的字传送到四字  |
| movslq    |                  | 将做了符号扩展的双字传送到四字 |
| cltp      | 符号扩展（%rax）->%rax | 把%eax符号扩展到%rax  |

符号扩展数据传送令：movs指令以寄存器或内存地址作为源，以寄存器作为目的。cltp指令制作用于
寄存器%eax和%rax
cltp效果和movslq %eax，%rax 完全一致

#### 3.数据传送指令实例
c代码
```c
long exchange(long *xp, longy)
{
    long x = *xp;
    *xp = y;
    return x;
}
```

汇编代码
```
long exchange(long *xp, long y)
xp in %rdi, y in %rsi
exchange:
  movq (%rdi), %rax  // 内存中读出x，把它放在寄存器%rax中（%rax是返回值）
  movq %rsi, (%rdi)  // 将y写入到寄存器%rdi中xp指向的内存位置
  ret
```
间接引用指针就是将该指针放在一个寄存器中，然后内存引用使用这个寄存器。

**练习题：**
src_t *sp;
dest_t *dp;

设sp和dp的值分别存储在%rdi和%rsi，汇编实现*dp = (dest_t) *sp;
需要两条指令
1. 从内存中读数，做适当转换，并设置到寄存器%rax的适当部分（%rax,%eax,%ax,%al）
2. 把%rax的适当部分写入到内存中。

| src_t         | dest_t        | 指令                                  | 详解                 |
|---------------|---------------|-------------------------------------|--------------------|
| long          | long          | movq (%rdi),%rax;movq %rax,(%rsi)   | 不用做扩展，正常4字移动       |
| char          | int           | movsbl (%rdi),%eax;movl %eax,(%rsi) | char有符号，需要符号扩展到双字  |
| char          | unsigned      | movsbl (%rdi),%eax;movl %eax,(%rsi) | char有符号，需要符号扩展到双字  |
| unsigned char | long          | movzbl (%rdi),%eax;movq %rax,(%rsi) | 读一个字节，无符号char扩展到4字 |
| int           | char          | movl (%rdi),%eax;movb %al,(%rsi)    | 不需要扩展，双字缩减到字节      |
| unsigned      | unsigned char | movl (%rdi),%eax;movb %al,(%rsi)    | 不需要扩展，双字缩减到字节      |
| char          | short         | movsbw (%rdi),%ax;movw %ax,(%rsi)   | 符号char扩展到字         |

### 5.压入和弹出栈数据
**栈指针(%rsp)保存着栈顶元素地址**
压入和弹出都是对栈顶的地址进行操作，栈顶元素的地址是栈中所有元素地址中最低的。

| 指令      | 效果                               | 描述     |
|---------|----------------------------------|--------|
| pushq S | R[%rsp]-8->R[%rsp];S->M[R[%rsp]] | 将四字压入栈 |
| popq D  | M[R[%rsp]]->D;R[%rsp]+8->R[%rsp] | 将四字弹出栈 |

pushq %rbp 指令等价于
subq $8 %rsp
movq %rbp,(%rsp)
这两个区别是pushq指令编码为1个字节，而上面2个指令一共需要8个字节

popq %rax等价于
movq (%rsp),%rax
addq $8,%rsp

### 6.算术和逻辑操作
指令类有各种大小不同操作数的变种（只有leaq没有其他大小的变种）。
例：addb,addw,addl,addq分别是字节加法，字加法，双字加法，四字加法

| 分组     | 指令       | 效果        | 描述         |
|--------|----------|-----------|------------|
| 加载有效地址 | leaq S,D | &S->D     | 加载有效地址     |
| 一元操作   | INC D    | D+1->D    | 加1         |
|        | DEC D    | D-1->D    | 减1         |
|        | NEG D    | -D->D     | 取反         |
|        | NOT D    | ~D->D     | 取补         |
| 二元操作   | ADD S,D  | D+S->D    | 加          |
|        | SUB S,D  | D-S->D    | 减          |
|        | IMUL S,D | D*S->D    | 乘          |
|        | XOR S,D  | D^S->D    | 异或         |
|        | OR S,D   | D或S->D    | 或          |
|        | AND S,D  | D&S->D    | 与          |
| 移位     | SAL k,D  | D<<K->D   | 左移         |
|        | SHL k,D  | D<<k->D   | 左移（等同于SAL） |
|        | SAR k,D  | D>>q k->D | 算术右移       |
|        | SHR k,D  | D>>l k    | 逻辑右移       |

leaq:通常用来执行简单的算数操作

#### 1. 加载有效地址
leaq实际上是movq指令的变形，指令形式是从内存读数据到寄存器，但实际上它根本没有引用内存。
它的第一个操作数看上去是一个内存引用，但该指令并不是从指定位置读入数据，而是将有效地址写
入到目的擦作数。

编译器经常发现leaq的一些灵活的用法，根本与有效地址计算无关。目的操作数必须是一个寄存器。
```c
long scale(long x,long y,long z) {
    long t = x + 4 * y + 12 * z;
    return t;
}
```

汇编代码:
```
long scale(long x,long y,long z)
x in %rdi, y in %rsi, z in %rdx
scale:
  leaq (%rdi, %rsi, 4), %rax  // x + 4 * y
  leaq (%rdx, %rdx, 2), %rdx  // z + 2 * x = 3*z
  leaq (%rax, %rdx, 4), %rax  // (x + 4 * y) + 4 * (3 * z) = x + 4 * y + 12 * z
```
leaq能执行加法和有限形式的乘法

#### 2. 一元和二元操作
一元操作：只有一个操作数，既是源也是目的，这个操作数可以使一个寄存器也可以是一个内存位置。
例如：incq (%rsp)会使栈顶的8个字节元素加1 
二元操作：第二个操作数既是源又是目的。
例：subq %rax, %rdx 第一个操作数可以使立即数、寄存器或是内存位置，第二个操作数是寄存器或内存位置。
注意：第二个操作数是内存地址时，处理器必须从内存读出值，执行操作，在写会内存。

#### 3. 移位操作
第一位操作数是移位量，第二个操作给出的是要移位的数。
可以进行算术和逻辑右移。
移位量可以是一个立即数，或者放在单字节寄存器%cl中。（这些指令很特别，因为只允许以这个特定的寄存器作为操作数）。
1个字节的移位量使得移位量编码范围达到2^8-1= 255，移位操作对w位长的数值进行操作，移位量是有%c寄存器的低m位决定的，
2^m = w，高位会被忽略。
当寄存器%cl的值是0xFF,salb会移动7位，salw会移动15位，sall会移动31位，salq会移动63位。

左移有两个名字：SAL和SHL效果是一样的，都是将右边填0.
右移指令不同，SAR执行算术移位（填上符号位），而SHR执行逻辑移位（填上0）。
移位操作的目的操作数可以是一个寄存器也可以是一个内存位置。

#### 4. 讨论
只有右移操作要求区分有符号和无符号数。这个特性使得补码运算成为有符号整数运算的一种比较好的方法之一。

```c
long arith(long x, long y, long z)
{
    long t1 = x ^ y;
    long t2 = z * 48;
    long t3 = t1 & 0x0f0f0f0f;
    long t4 = t2 - t3;
    return t4;
}
```

汇编代码:
```
x in %rdi, y in %rsi, z in %rdx
arith:
  xorq %rsi, %rid             // t1 = x ^ y
  leaq (%rdx, %rdx, 2), %rax  // 3 * z
  salq $4, %rax               // t2 = 16 * （3 * z） = 48 * z
  andl &252645135, %edi       // t3 = t1 & 0x0f0f0f0f
  subq %rdi, %rax             // return t2 - t2
  ret
```

#### 5. xorq %rdx, %rdx效果
1. 这个指令用来将寄存器%rdx设置为0，运用了对任意x,x^x=0这一属性。它对于c语句x=0
2. 将寄存器%rdx设置为0的更直接方式是指令moveq $0,%rdx
3. 使用xorq的版本只需要3个字节，而使用movq的版本需要7个字节。其他将%rdx设置为0的方法都依赖于这样一个属性，
即任何更新低于4字节的指令都会把高位字节设置为0.因此可以使用xorl %edx, %edx(2字节)或movl $0,%edx(5字节)

### 5. 特殊的算术操作
两个64位有符号或无符号整数相乘得到的乘积需要128位来表示。（16字节，8字）

| 指令      | 效果                                                        | 描述     |
|---------|-----------------------------------------------------------|--------|
| imulq S | S*R[%rax]->R[%rdx]:R[%rax]                                | 有符号全乘法 |
| mulq S  | S*R[%rax]->R[%rdx]:R[%rax]                                | 无符号全乘法 |
| clto    | 符号扩展（R[%rax]）->R[%rdx]:R[%rax]                            | 转换为八字  |
| idivq S | R[%rdx]:R[%rax] mod S->R[%rdx],R[%rdx]:R[%rax]/S->R[%rdx] | 有符号除法  |
| divq S  | R[%rdx]:R[%rax] mod S->R[%rdx],R[%rdx]:R[%rax]/S->R[%rdx] | 无符号除法  |

x86-64指令集提供了两条不同的“单操作数”乘法指令，以计算64位值的全128位乘积————一个是无符号乘法（mulq），
另一个是补码乘法(imulq)。这两条指令都要求一个参数必须在寄存器%rax中，而另一个作为指令的源操作数给出。
然后乘积存放在寄存器%rdx(高64位)和%rax(低64位)中。虽然imulq这个名字可以用于两个不同的乘法操作，但是
汇编器能够通过计算操作数的数目，分辨出想用哪条指令。

```c++
#include <inttypes.h>

// GCC提供的128位整数支持
typedef unsigned __int128 uint128_t;
// 显示的把x和y声明为64位的数字
void store_uprod(uint128_t *dest, uint64_t x, uint64_t y){
  *dest = x * (uint128_t)y;
}
```
GCC生成的汇编代码 %rax 返回值 %rsi第二个参数 %rdx 第三个参数 %rdi 第一个参数
```
void store_uprod(uint128_t *dest, uint64_t x, uint64_t y)
dest in %rdi, x in %rsi, y in %rdx
store_uprod:
  movq %rsi, %rax     // copy x to multiplicand
  mulq %rdx           // multiply by y
  movq %rax, (%rdi)   // store lower 8 bytes at dest
  movq %rdx, 8(%rdi)  // store upper 8 bytes at dest+8
  ret
```

x86-86实现除法
```c++
void remdiv(long x, long y,long *p, long *rp) {
  long q = x/y; 
  long r = x%y;
  *qp =q; 
  *rp = r;
}
```
汇编代码 %rax 返回值 %rsi第二个参数 %rdx 第三个参数 %rdi 第一个参数 %r8第五个参数
```
void remdiv(long x, long y, long *qp, long *rp)
x in %rdi,y in %rsi,qp in %rdx,rp in %rcx
remdiv:
  movq  %rdx, %r8   // copy qp
  movq  %rdi, %rax  // move x to lower 8 bytes of divi
  cqto              // sign-extend to upper 8 bytes of dividend
  idviq %rsi        // divide by y
  movq  %rax, (%r8) // store quotient at qp
  movq  %rdx, (%rcx)  // store remainder at rp
  ret
```
上述代码中
第二行：把参数qp保存到另外一个寄存器中%r8，因为除数操作要使用参数寄存器%rdx
第三四行：准备被除数，复制并符号扩展x，存放在%rax和%rdx中
第五行：除以y
第六行：寄存器%rax的商保留在qp
第七行：寄存器%rdx中的余数被保存在rp商
（通常寄存器%rdx会事先设置成0）

### 6. 控制
jump指令可以改变一组机器代码指令的执行顺序，jump指令指定控制应该被传递到程序的某个其他部分

#### 1. 条件码
除了整数寄存器，CPU还维护着一组单个位的`条件码（condition code）`寄存器，它们描述了最近的算术或逻辑操作
的属性。可以检测这些寄存器来执行条件分支指令。最常用的条件码有：
- CF:进位标志。最近的操作使最高位产生了进位。可用来检查无符号操作的溢出。
- ZF:零标志。最近的操作得出的结果位0.
- SF：符号标志。最近的操作得到的结果位负数。
- OF：溢出标志。最近的操作导致一个补码溢出————正溢出或负溢出。

比如，用一条add指令完成等价于t=a+b的功能(a,b,t都是整形)，根据c表达式来设置条件码
CF: (unsigned) t < (unsigned) a 无符号溢出
ZF: （t==0）  零
SF:  (t < 0)  负数
OF （a<0==b<0） && (t<0 != a < 0) 有符号溢出

leaq指令不改变任何条件码，因为它是用来进制地址计算的。
INC、DEC、NEG、NOT、ADD、SUB、IMUL、XOR、OR、AND、SAL、SHL、SAR、SHR 都会设置条件码。
例如：XOR，进位标志和溢出标志都会设置成0。移位操作进位标志将设置为最后一个被移除的位，而溢出标志设置0.
INC和DEC指令会设置溢出和零标志，但是不会改变进位标志。

CMP指令根据两个操作数之差来设置条件码。除了只设置条件码而不更新目的寄存器之外，cmp指令和sub指令的行为是一样的。
test指令的行为与and指令一样，除了它们只设置条件码而不改变目的寄存器的值。
典型的用法是，两个操作数是一样的（例如 testq %rax, %rax用来检查%rax时负数、零、还是整数），或者其中一个操作数
是一个掩码，用来指示哪些位应该被测试。

| 指令         | 基于    | 描述   |
|------------|-------|------|
| CMP S1, S2 | S2-S1 | 比较   |
| cmpb       |       | 比较字节 |
| cmpw       |       | 比较字  |
| cmpl       |       | 比较双字 |
| cmpq       |       | 比较四字 |
| TEST S1,S2 | S1&S2 | 测试   |
| testb      |       | 测试字节 |
| testw      |       | 测试字  |
| testl      |       | 测试双字 |
| testq      |       | 测试四字 |


#### 2. 访问条件码
条件码通常不会被直接读取，常用的使用方法有三种
1. 可以根据条件码的某种组合将1个字节设置为0或1
2. 可以根据跳转到程序的某个其他的部分
3. 可以有条件地传送数据

将这一整类指令成为SET指令；它们之间的区别就在于它们考虑的条件码组合是什么，这些指令名字的不同后缀指明了
它们所考虑的条件码的组合。这些指令的后缀表示不同的条件，而不是操作数大小。
一条set指令的目的操作数是低位单字节寄存器元素之一，或是一个字节的内存位置，指令会将这个字节设置成0或者1。
为了得到一个32位或64位结果，我们必须对高位清零。一个c语言表达式a<b的典型指令序列如下所示，这里
a和b都是long类型

| 指令      | 同义名    | 效果                | 设置条件         |
|---------|--------|-------------------|--------------|
| sete D  | setz   | D<-ZF             | 相等/零         |
| setne D | setnz  | D<-~ZF            | 不等/非零        |
| sets D  |        | D<-SF             | 负数           |
| setns D |        | D<-~SF            | 非负数          |
| setg D  | setnle | D<-~(SF^OF) & ~ZF | 大于（有符号>）     |
| setge D | setnge | D<-~(DF^OF)       | 大于等于（有符号>=）  |
| setl D  | setnge | D<-SF^OF          | 小于（有符号<）     |
| setle D | setnge | D<-(SF^OF)或ZF     | 小于等于（有符号<=）  |
| seta D  | setnbe | D<-~CF&~ZF        | 超过（无符号>）     |
| setae D | setnb  | D<-~CF            | 超过或相等(无符号>=) |
| setb  D | setnae | D<-CF             | 低于（无符号<）     |
| setbe   | setna  | D<-CF或ZF          | 低于或等于（无符号<=） |

set指令。每条指令根据条件码的某种组合，将一个字节设置为0或1.这些指令有“同义名”，也就是同一条机器指令有别的名字。

汇编代码
```
int comp(data_t a, data_t b)
a in &rdi,b in %rsi
comp:
  comp %rsi, %rdi   // compare a:b
  setl %al          // set low-order byte of %eax to 0 or 1
  movzbl %al, %eax  // clear rest of %rax(and rest of %rax)
  ret
```
注意：
comp %rsi, %rdi 虽然参数列出的顺序是%rsi(b)再是%rdi(a)，实际上比较的是a和b。
movzbl 进行0扩展，不仅会吧%rax的高三个字节清零，还会把整个寄存器%rax的高4个字节都清零。

#### 3. 跳转指令
跳转`jump`指令到到导致执行切换到程序中一个全新的位置，在汇编代码中，这些跳转的目的通常用一个标号指明
(人为编造的汇编代码)
```
movq $0, %rax   // set %rax to 0
jmp .L1         // goto .L1
movq (%rax), %rdx // null pointer dereference(skipped)
.L1:
popq %rdx     // jump target
```
指令jmp .L1会导致程序跳过movq指令，而从popq指令开始继续执行。
- 直接跳转 jump .L1 ： 跳转目标作为指令的一部分编码
- 间接跳转寄存器中读出 jmp *%：用寄存器%rax的值作为跳转目标
- 间接跳转内存中读出  kmp *(%rax)：以%rax的值作为读地址，从内存读出跳转目标

| 指令           | 同义名  | 跳转条件         | 描述           |
|--------------|------|--------------|--------------|
| jmp Label    |      | 1            | 直接跳转         |
| jmp *Operand |      | 1            | 间接跳转         |
| je Label     | jz   | ZF           | 相等           |
| jne Label    | jnz  | ~ZF          | 不等           |
| js Label     |      | SF           | 负数           |
| jns Label    |      | ~SF          | 非负数          |
| jg Label     | jnle | ~(DF^OF)&~ZF | 大于（有符号>）     |
| jge Label    | jnl  | ~(SF^OF)     | 大于或等于（有符号>=） |
| jl Label     | jnge | SF^OF        | 小于（有符号<）     |
| jle Label    | jng  | (SF^OF)或 ZF  | 小于或等于(有符号<=) |
| ja  Label    | jnbe | ~CF&~ZF      | 超过（无符号>）     |
| jae Label    | jnb  | ~CF          | 超过或相等（无符号>=） |
| jb  Label    | jnae | CF           | 低于(无符号<)     |
| jbe Label    | jna  | CF或ZF        | 低于或相等（无符号<=） |

jump指令。当条件满足时，这些指令会跳转到一条带标号的目的地。有些指令有“同义名”。

这些指令的名字和跳转条件与set指令的名字和设置条件相匹配额。条件跳转只能是直接跳转。

#### 4. 跳转编令的编码
在汇编代码中，跳转目标用符号标号书写。汇编器以及后来的链接器，会产生调整目标的适当编码。
跳转指令有几种不同的编码。
- 最常用的都是PC相对的（PC-relative）。也就是，它们会将目标指令的地址与紧跟在跳转指令后面那条指令的地址
之间的差位编码。这个地址偏移量可以编码位1、2或4个字节。
- 第二种编码方法是给出“绝对”地址，用4个字节直接指定目标。

编译器和链接器会选择适当的跳转目的编码。

PC相对寻址的例子，包含两个跳转，2行的jmp指令前向跳转到更高的地址，而7行的jg指令后向跳转到较低的地址。
```
  moveq %rdi, %rax
  jmp .L2
.L3:
  sarq  %rax
.L2:
  testq %rax, %rax
  jg  .L3
  req;ret
```

汇编器产生的.o格式的反汇编版本如下
```
1   0:  48 89 f8    mov %rdi,%rax
2   3:  eb 03       jmp 8 <loop+0x8> // 03（pc相对寻址数） + 5(下一行的地址)
3   5:  48 d1 f8    sar %rax
4   8:  48 85 c0    test %rax,%rax
5   b:  7f f8       jg 5 <loop+0x5> // f8(目标用单字节、补码表示为-8)+0xd(13)
6   d:  f3 c3       repz retq
```

```
1   4004d0:  48 89 f8    mov %rdi,%rax
2   4004d3:  eb 03       jmp 4004d8 <loop+0x8> 
3   4004d5:  48 d1 f8    sar %rax
4   4004d8:  48 85 c0    test %rax,%rax
5   4004db:  7f f8       jg 4004d5 <loop+0x5> 
6   4004dd:  f3 c3       repz retq
```

#### 5. 用条件控制来实现条件分支
将条件表达式和语句从C 语言翻译成机器代码，最常用的方式是结合有条件和无条件 跳转。
(另一种方式在 6.6 节中会看到，有些条件可以用数据的条件转移实现，而不是 用控制的条件转移来实现。)
例如，a 给出了一个计算两数之差绝对值的两数的C 代码。
这个两数有一个副作用，会增加两个计数器，编码为全局变量1t cat和ge cnt 之 一。
GCC 产生的汇编代码如c 所示。把这个机器代码再转换成C 语言，我们称之为 函数got odiff_se(b)。
它使用了C语言中的got。语句，这个语句类似于汇编代 码中的无条件跳转。使用got 。
语句通常认为是一种不好的编程风格，因为它会使代码非常难以阅读和调试。本文中使用goto 语句，
是为了构造描述汇编代码程序控制流的C程序。我们称这样的编程风格为“goto代码”。
在 goto代码中(b)， 第5行中的gotox_ge_y语句会导致跳转到第9行中的标号xge_y处(当x>=y，时会进行跳转)。
从这一点继续执行，完成函数absdiff_se的else部分并返回。
另一方面，如果测试x>=y失败，程序会计算absdiff_se的if部分指定的步骤并返回。
汇编代码的实现(c) 首先比较了两个操作数(第2行)，设置条件码。
如果比较的结果表明工大于或者等于，那么它就会跳转到第8行，增加全局变量ge_cnt计算x - y作为返回值并返回。
由此我们可以看到absdiff_se对应汇编代码的控制流非常类似于gotodiff_se的goto代码。
a 源氏c代码
```c
long lt_cnt = 0;
long ge_cnt = 0;
long absdiff_se(long x, long y)
{
  long result;
  if (x > y){
    it_cnt++;
    result = y - x;
  } else {
    ge_cnt++;
    result = x - y;
  }
  return result;
}
```
b 等价的goto版本（模拟了汇编代码的控制）
```c
long gotodiff_se(long x, long y)
{
  long result;
  if (x >= y)
    goto x_ge_y;
   lt_cnt++;
   result = y - x;
   return result;
  x_ge_y:
    ge_cnt++;
    result = x - y;
    return result;
}
```
c 产生的汇编代码 %rip是计数器
```c
long absdiff_se(long x, long y)
x in %rdi, y in %rsi
absdiff_se:
  cmpq  %rsi, %rdi    // compare x:y
  jge   .L2           // if >= goto x_ge_y
  addq  $1, lt_cnt(%rip)  // lt_cnt++
  movq  %rsi, %rax
  subq  %rdi, %rax    // result = y - x
  ret
.L2:
  addq  $1, ge_cnt(%rip)  // ge_cnt++
  movq  %rdi, %rax
  subq  %rsi, %rax        // result = x - y
  ret       
```































































