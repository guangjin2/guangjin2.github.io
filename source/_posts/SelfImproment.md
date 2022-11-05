---
title: 程序员的自我修养
date: 2022-10-26 00:00:00
tags: 
  - c++提高
  - 链接、装载与库
categories: c++
top_img: /img/mou.jpeg
cover: false
copyright: true
toc_number: false
---

> **声明**：本文是**《程序员的自我修养》**（俞甲子、石凡、潘爱民）的个人笔记(略去了windows部分)。另原书非常推荐在学习一段时间c/c++之后阅读。

## 1. 温故而知新
1. 计算机三大组成部件： CPU，内存，IO控制芯片

2. 硬件结构框架， 北桥PCI Bridge， 南桥ISA Bridge
![CmpArch](assets/SelfImp/CmpArch.png)

3. 系统软件分为两种：
  一为平台性质的， 比如操作系统，内核，驱动程序，运行库，系统工具等
  二为开发程序所用的,比如编译器，汇编器，链接器等开发工具和开发库
![CmpArch2](assets/SelfImp/CmpArch2.png)

4. 内存分段， 利用率太低
![mem](assets/SelfImp/mem.png)

5. 内存分页
![mem2](assets/SelfImp/mem2.png)

6. MMU 将虚拟地址转换程物理地址,一般集成在cpu 内部了
![mem3](assets/SelfImp/mem3.png)

7. 标准线程一般由线程ID, PC,寄存器集和堆栈组成
![thread](assets/SelfImp/thread.png)
频繁等待的成为IO密集型线程， 少等待的称为CPU密集型， 通常IO 密集型容易获得优先级的提升，线程的优先级改变的三种方式：
	- 用户指定
	- 根据等待频繁程度修改
	- 长时间得不到执行而提升优先级
![thread2](assets/SelfImp/thread2.png)

8. Linux 的多线程：
Linux将所有的执行实体(无论是线程还是进程)都称为任务(Task)；
       每一个任务概念上都类似于一个单线程的进程,具有内存空间、执行实体、文件资源等。
       Linux下不同的任务之间可以选择共享内存空间,因而在实际意义上,共享了同一个内存空间的多个任务构成了一个进程,这些任务也就成了这个进程里的线程
| 系统调用 | 作用 |
| ---- | ----|
| fork | 复制当前进程 |
| exec | 使用新的执行映像覆盖当前可执行映像 |
| clone | 创建子进程并从指定位置开始执行 |

![proc](assets/SelfImp/proc.png)

9. 线程同步
* 原子操作
* 二元信号量/多元信号量， 信号量初始值为1为二元， 信号量初始值为N 为多元
* 互斥量，与二元信号量相似，互斥量要求由获取者线程释放，信号量整个系统均可获取和释放
* 临界区，与互斥量相似， 但是将本进程创建的锁可见范围控制在本进程内
* 读写锁
| 读写锁状态 | 以共享获取 | 以独占方式获取 |
| ---- | ---- | ---- |
| 自由 | 成功 | 成功 |
| 共享 | 成功 | 等待 |
| 独占 | 等待 | 等待 |
* 条件变量

10. 可重入：一个函数被称为可重入的,表明该函数被重入之后不会产生任何不良后果

11. 过度优化
* 编译器优化
```c++
x = 0;
Thread1 Thread2
lock()  lock()
x++;     x++;
unlock();  unlock();

x最终的值可能为1. 
不同线程使用独立的寄存器。
当某个线程计算完x++后，编译器为了提高速度，并没有将1写回到内存中。
所以另一个线程执行完x++后。最终写回到变量x的值为1
```
* cpu 动态调度换序和编译器优化
```c++
x = y = 0;
Thread1 Thread2
x =1;     y=1;
r1=y;   r2=x;

逻辑上至少一个结果为1，但结果可能是r1=r2=0,执行顺序可能是
x = y = 0;
Thread1 Thread2
r1=y;   y=1;
x =1;   r2=x;
原因在于为了提高效率，编译器或cpu 会将两个不相关的指令换序

volatile 可以
#1.阻止编译器效率将一个变量缓存到寄存器而不写回
#2.阻止编译器调整操作volitile 变量的指令顺序
但无法解决cpu 动态调序
```
* 双检测锁的问题,C11官方推荐的单例实现方式是利用static 变量定义的排他性
```c++
volitile T* pInst = 0;
T* GetInstance(){
    if(pIns == NULL){
        lock();
        if(pInst == NULL) //防止通过第一重判断没有申请到锁的线程
            pInst = new T;
        unlock();                
    }
    return pInst;
}
new 的操作分为申请内存， 构造， 返回地址
构造和返回可以互换， 故程序可能使用到没有构造完成的地址而崩溃

使用cpu 提供的阻止换序方法可以解决
比如barrier（）能阻止该指令之前的命令被换到barrier 之后
```

12. 三种线程模型
* 1对1：用户线程与内核1对1
优：真正的实现并发，线程独立不会互相阻塞
缺：内核线程有限使得用户线程受限
　　内核线程调度时，上下文切换开销大， 使得用户线程效率低下
* 多对1
![Ｍthread](assets/SelfImp/Mthread.png)

　　　优：高效的上下文切换和几乎无限制的用户线程
　　　缺：用户线程阻塞则内核线程阻塞，所有用户进程阻塞
* 多对多
![Ｍthread2](assets/SelfImp/Mthread.png)

　　　综合多对1和1对1的优缺






## 2.编译与链接
### 2.1 编译过程（开发视角）
1. 预处理：主要处理源马中的预编译指令
命令：gcc -E hello.c -o hello.i或cpp hello >hello.i
程序：cc1/cc1plus/cc1obj/jc1
后缀:.h,.c->.i , .cpp,.hpp->.ii
过程：
　　处理预编译指令
　　删除注释
　　添加行号和文件标识，方便调试和产生错误警告信息
　　保留#pragma 编译器指令， 因为编译器需要使用他们
2. 编译：将预处理完的文件进行一系列词法分析，文法分析产生汇编代码
命令：gcc -S hello.i -o hello.s
程序：cc1/cc1plus/cc1obj/jc1
现代编译器通常预处理和编译一起执行，可用命令：
gcc -S hello.c -o hello.s
3. 汇编:将汇编代码专场机器指令
命令：gcc -c hello.s -o hello.o
程序：as
4. 链接：链接库文件得到真正的可执行程序
程序：ld
过程包括地址和空间分配，符号决议和重定位等等
![compile](assets/SelfImp/compile.png)

### 2.2 编译过程（编译器视角）
* 编译器编译过程（另，书中举了个例子， 可以配合哈工大编译原理公开课的导论来参考原文）
![compile2](assets/SelfImp/compile2.png)

* 预处理不归编译器范畴
*  词法分析：将源代码字符序列生成一系列记号，同时做将标识符放入符号表，数字、字符串常量存放到文字表等准备工作，现成工具lex
* 语法分析：根据记号进行上下文无关的语法分析，产生语法树， 同时确定优先级，发现语法错误，现成工具yacc
* 语义分析：由语义分析器进行静态语义分析，通常包括声明和类型的匹配， 类型的转换
* 源码级优化器：对源码级进行优化产生中间代码， 常见的有三址码、P代码， 比如a=2+3 可以直接优化成a=5，由于中间代码的引入，可以使编译器分为前端和后端， 前端与机器无关
* 代码生成器：将中间代码转成目标机器代码
* 目标代码优化器：对目标代码优化，例如使用位移替换乘法运算等

## 3.目标文件
### 3.1 目标文件格式

> ELF:Excutable Linkable Format

![ch3_01.png](assets/SelfImp/ch3_01.png)
![ch3_02.png](assets/SelfImp/ch3_02.png)
![ch3_03.png](assets/SelfImp/ch3_03.png)

### 3.2 目标文件基本内容
![ch3_04.png](assets/SelfImp/ch3_04.png)
- Header:文件属性，包括是否可执行，是静态还是动态及入口地址，目标硬件， 目标操作系统等
- .text: 编译后的机器代码
- .data: 已初始化的全局变量和静态变量
- .bss:未初始化(或等同于未初始化，如static int i=0被编译器优化后)的全局变量和静态变量（预留位置，没有内容，在文件中不占据空间

### 3.3 一个有代表的例子
```c++
//gcc -c SimpleSection.c
int printf(const char *format, ...);
int global_init_val = 84;
int global_uninit_val;

void func1(int i) {
   printf("%d\n",i);
}

int main(void) {
 static int static_var = 85;
 static int static_var2;
  int a = 1;
  int b ;
  func1(static_var + static_var2 + a+b);
  
  return a;
}
```
![ch3_05.png](assets/SelfImp/ch3_05.png)

> *CONTENTS 代表该段在文件中存在， ALLOC表示不存在，ox0000450即文件大小， 书中正是1104，注意.bss 的File off 

![ch3_06.png](assets/SelfImp/ch3_06.png)
![ch3_07.png](assets/SelfImp/ch3_07.png)
![ch3_08.png](assets/SelfImp/ch3_08.png)

分析：
- .text 反汇编结果大小0x5b
- .data 大小为8字节，对应代码的static_var和global_init_varabal
- .rodata大小为4字节，对应常量：“%d\n”
- .bss 大小为4字节，对应static_var2, 因为有些编译器只会对全局未初始化的变量记录一个未定义的全局变量符号

### 3.4 其他常见的段
![ch3_09.png](assets/SelfImp/ch3_09.png)

gcc指定自定义段：
```c++
__attribute__((section("FOO"))) int global = 42;
__attribute__((section("BAR"))) void foo{}
```

> 注意不要以"."开头，容易冲突

### 3.5 ELF 文件结构描述
![ch3_10.png](assets/SelfImp/ch3_10.png)

#### 3.5.1 ELF文件头
![ch3_11.png](assets/SelfImp/ch3_11.png)

```c++
// /usr/include/elf.h
typedef struct {
  unsigned char e_ident[16]; // 输出图中version以上的内容
  Elf32_Half e_type;    //ELF文件类型
  Elf32_Half e_machine; //ELF文件的CPU平台属性
  Elf32_Word e_version; //ELF版本号
  Elf32_Addr e_entry;   //ELF文件的入口虚拟地址。重定向文件入口地址为0
  Elf32_Off  e_phoff;        //暂时不关心，链接和执行视图介绍
  Elf32_Off  e_shoff;   //段表在文件中的偏移
  Elf32_Word e_flags;   //文件头标志位
  Elf32_Half e_ehsize;  //文件头本身的大小 sizeof(ELF32_Ehdr)
  Elf32_Half e_phentsize;     //暂时不关心，链接和执行视图介绍
  Elf32_Half e_phnum;         //暂时不关心，链接和执行视图介绍
  Elf32_Half e_shentisize; //段表描述符大小 sizeof(ELF32_Shdr)
  Elf32_Half e_shnum;      //段表描述符数量
  Elf32_Half e_shstrndx;   //段表字符串表所在的段在段表中的下标
} Elf32_Ehdr;
```

>  OS加载可执行文件时会检查魔数，其构成为：0x7f,‘E’,‘L’,‘F’,文件字长类型，字节序类型，主版本号，其余9字节未定义，一般为0，有些平台会用做扩展标志

#### 3.5.2 段表
![ch3_12.png](assets/SelfImp/ch3_12.png)

```c++
// ‘/usr/include/elf.h’ struct Elf32_Shdr
typedef struct
{
  Elf32_Word  sh_name;  //段名在字符串表 ".shstrtab"的偏移量
  Elf32_Word  sh_type;  //段的类型
  Elf32_Word  sh_flags; //段的标志位
  Elf32_Addr  sh_addr;  //段虚拟地址:如果可以被加载，则为加载后的虚拟地址。否则为0
  Elf32_Off   sh_offset;//段在文件中的偏移
  Elf32_Word  sh_size;  //段的大小
  Elf32_Word  sh_link;  //段链接信息
  Elf32_Word  sh_info;  //段链接信息
  Elf32_Word  sh_addralign; //段地址对齐
  Elf32_Word  sh_entsize;   //项的长度
} Elf32_Shdr;
```
![ch3_13.png](assets/SelfImp/ch3_13.png)

* 中间不连续相差几个字节是因为内存对齐，段表长=11个段*sizeof(Elf32_Shdr)
* rel.text 为重定位表，sh_type=SH_REL, sh_link=符号表的下标， sh_info=作用段的下标
* 字符串表（.shstrtab,.strtab等）保存ELF 中的字符串，下标选择

### 3.6 符号表(.stmtab)
链接的接口，可以用nm 直接查看
```c++
//符号表时Elf32_Sym 的数组
typedef struct {
  Elf32_Word    st_name; //符号名，包含了符号名在字符串表中的下标
  Elf32_Addr    st_value; //符号相对应的值。
  Elf32_Word    st_size; //符号大小
  unsigned char st_info; //符号类型（低4位）和绑定信息（高28位）
  unsigned char st_other;//目前为0，暂时没用
  Elf32_Half    st_shndx;//符号所在的段
} Elf32_Sym;
```
![ch3_14.png](assets/SelfImp/ch3_14.png)

- num=2 是一个段，Ndx=1=>段表中的下标为2的段 ， 即.text
- func1和main位于xxx.c中， 位于.text段,ndx=1 （objdump -x/readelf -a可以验证）
- printf 是引用变量， ndx=SHN_UNDEF
- statuc_var.1533 是静态变量,仅编译单元内可见，bind=LOCAL位于.data
- global_init_var位于.data,ndx=3(书中写成了.bss是错的)
- statuc_var2 位于.bss, ndx=4, global_uninit_var 位于COMMON，ndx=COM（呼应第一节） 
 特殊符号 ：连接器产生可执行文件时，由链接器产生，可以直接使用，后续章节再详细介绍
如：__excutable_start 程序起始地址,__etext 代码段结束地址等

### 3.7 符号修饰与函数签名
gcc的c++为例，可以通过c++file _Z4funci 解析被修饰过的名称
![ch3_15.png](assets/SelfImp/ch3_15.png)
_Z开头， N开始域名，4结束域名，E结束函数体， E后加参数类型 详略

### 3.8 强弱符号
多个目标文件有相同的全局符号 链接时会出现重复定义的错误，则称为强符号
强符号：编译器默认函数和初始化了的全局变量
弱符号：未初始化的全局变量
可以通过__attribute__((weak)) 来定义
```c++
extern int ext; //外部引用，非强非弱
int weak;//弱
int strong = 1;// 强
__attribute__((weak)) weak2=2;//弱
int main() {//强
   return 0
}
```
链接器处理规则：
- 不允许强符号多次定义， 否则报重定义错
- 多个目标文件，一强多弱，选用强符号
- 全为弱符号则选取占用空间最大的一个

### 3.9 强弱引用
 必须在外部找到符号的定义，否则报未定义错误，称为强引用，后续详述
可以通过__attribute__((weakref))指定弱符号
- 这种弱符号和弱引用对于库来说十分有用：
程序可以对某些扩展功能模块的引用定义为弱引用，当我们将扩展模块与程序链接在一起时,功能模块就可以正常使用如果我们去掉了某些功能模块,那么程序也可以正常链接,只是缺少了相应的功能,这使得程序的功能更加容易裁剪和组合。
```c++
__attribute__((weakref)) void foo ();
int main() {
       if(foo) //弱引用默认编译器设为0
          foo();
   return 0;
}
//未找到定义时，可以编译运行
```

- 库中定义的弱符号可以被用户定义的强符号所覆盖,从而使得程序可以使用自定义版本的库函数


## 4. 静态链接

### 4.1 合并.o文件
![ch4_01.png](assets/SelfImp/ch4_01.png)

两步链接：
- 空间与地址分配
    - 扫描所有的输入目标文件,并且获得它们的各个段的长度属性和位置
    - 将输入目标文件中的符号表中所有的符号定义和符号引用收集起来,统放到一个全局符号表
    - 通过上面的过程, 链接器将能够获得所有输入目标文件的段长度,并且将它们合并,计算出输出文件中各个段合并后的长度与位置,并建立映射关系
- 符号解析与重定位
    - 用上面第一步中收集到的所有信息,读取输入文件中段的数据、重定位信息,并且进行符号解析与重定位、调整代码中的地址等

### 4.2 简单的例子
```c++
/* a.c */                        
extern int shared;
int main() {
  int a = 100;
  swap(&a, &shared);
}

/* b.c */
int shared = 1;
void swap(int *a, int *b) {
   *a ^= *b ^= *a ^= *b;
}
/****
$ gcc -c a.c b.c
$ ld a.o b.o -e main -o ab
-e main 表示将main函数作为程序的入口。ld默认的程序入口为_start。
-o ab   表示链接输出文件名为ab, 默认为a.out。
**/
```
![ch4_02.png](assets/SelfImp/ch4_02.png)

> VMA (VIrtual Memery Address)， LMA（Load Memory Address），有些嵌入式中会不一样， ld 之后ab 的VMA 变为0x8048094,Linux 中的ELF默认从 0x8048000 开始， 后续介绍

**空间与地址分配**
![ch4_03.png](assets/SelfImp/ch4_03.png)

符号地址的确定：链接第一步时已经计算好了.text=0x08048094,.data = 0x08049108的起始位置，基于此以及合并前的偏移量计算各符号的虚拟地址，如：
main = 0x08048094[.text]+0x00
swap = 0x08048094[.text]+0x34
shared= 0x08049108[.data]+0x00


**符号解析与重定位**
![ch4_04.png](assets/SelfImp/ch4_04.png)

1. 编译器不确定shared 和swap 的地址， 所以给了捏造的0x0000和0xfffffc
![ch4_05.png](assets/SelfImp/ch4_05.png)

2. 由.rel.data 段（表）确定有哪些需要重定位的数据， .rel.text 段（表）确定有哪些需要重定位的代码
![ch4_06.png](assets/SelfImp/ch4_06.png)
```c++
typedef struct {
   Elf32_Addr r_offset; //重定位入口在对应段中的偏移
   Elf32_Word r_info;   //重定位入口的类型和符号
} Elf32_Rel;
```
![ch4_07.png](assets/SelfImp/ch4_07.png)

3. 符号解析在重定位时查找全局符号表（每一个UND均已经被消灭合并）
![ch4_08.png](assets/SelfImp/ch4_08.png)

4. 指令修正：
R_386_32 方式= S+A
R_386_PC32方式 = S+A -P
A = 保存在被修正位置的值
S = 符号实际地址， 即r_info 高24位 
P = 被修正的位置， 可通过r_offset 计算得到

5. COMMON 块， 例如前面提到到的未初始化的全局变量 这种弱符号在链接时通过全局符号表才能确定最终大小并将其输出到.bss 段
int global __attribute__((nocommon)) //声明不将之放在common 块中， 此时global 为强符号，也可以通过gcc -fnocommon进行设置

### 4.3 C++ 相关问题
- 重复代码消除
    - 例如模板， 多个编译单元互相不知道对方生成了那些模板实例
    一个通用 的做法是将实例代码放到单独的段中，以此，a生成的.temp.add<int>和b 生成的.temp.add<float>能被正确的区分、合并
	gcc 中命名为.gnu.linkonce.name,name 表示函数模板修饰后的名称
虚函数，内联，构造，拷贝构造， 赋值类似。
这样做的一个问题， 如果多目标使用不同编译器版本或选项，则相同的段内容不同，只能随机选择一个并发出一个warning
    - 函数级别链接可以让编译器把所有的函数输出到独立的段中，链接器使用时可以过滤没有用到的函数，以此减少了文件长度， 但会减慢链接过程， 因为链接器需要极端函数间的关系以及复杂的重定位关系
- 全局构造和析构
    - .init 保存进程初始化代码， 全局初始化代码放在这里， main 执行之前执行
    - .fini 保存进程结束代码， 全局析构代码放在这里， main 执行结束时执行
- 目前c++ ABI 标准混乱

### 4.4 静态库链接
- 静态库可以看成是一组目标文件的压缩集合， ar -t {file} 可以看到里面包含的.o
- 查找目标函数是否在该静态文件中可以
![ch4_09.png](assets/SelfImp/ch4_09.png)
- 链接自动关联相关的.o
![ch4_10.png](assets/SelfImp/ch4_10.png)

### 4.5 链接过程控制
一般情况下不需要特别关注， 但有时有特殊要求时需要处理， 比如OS 内核， BIOS,一些没有os 的情况下运行的程序
方式：
- 使用命令行指定参数，例如ld 的-o,-e等
- 将链接指令放在目标文件里，比如PE 编译器中的.drectve用来传递参数
- 使用链接脚本，最灵活强大的方法
    - ld -verbose 查看脚本， 默认的脚本位于/usrl/lib/ldscript 下
    - ld -T 指定脚本
    - 示例及脚本解析略

### 4.6 BFD库
- 这个库目的在于希望通过统一的接口来处理不同的文件格式，以此隔离编译器，链接器和具体的目标文件
- 现在的gcc ， 链接器，调试器均使用BSD来处理目标文件，而不是直接操作

## 6. 可执行文件的装载进程
### 6.1 进程的创建
1. 创建一个独立的虚拟空间
    - linux 内核只分配一个页目录（Page Directory），等后面程序发生页错误时再设置
2. 读取可执行文件头， 建立虚拟空间与可执行文件的映射关系
    - 例如ELF 只有一个.text, 虚拟地址为0x08048000， 大小为0x000e1, 对齐为0x1000(4kb),则映射如图：
    - 这种映射关系保存在OS 内部的一个数据结构中， linux 中称为VMA
![ch6_01.png](assets/SelfImp/ch6_01.png)

3. 将cpu 的指令寄存器设置成可执行文件的入口地址，启动运行
4. 程序运行时，发现物理页为空，则OS 页错误处理例程通过VMA 找到页面对应于可执行文件的偏移，将其载入物理页
*想到一个问题，程序运行时， 删除文件并不会造成程序崩溃是为什么呢？
因为linux  在删除磁盘文件， 必须等所有使用该文件的进程都关闭才会真正删除

### 6.2 进程寻存空间的分布
- 操作系统并不关心ELF各段的内容，只关注装载相关的问题， 最主要的就是权限，权限组合基本上是三种：
    - 以代码段为代表的权限为可读可执行的段
    - 以数据段和BSS段为代表的权限为可读可写的段
    - 以只读数据段为代表的权限为只读的段
是故，可以合并相同权限的段(sections)，把它当成一个段(Segments)进行映射以节省空间
![ch6_02.png](assets/SelfImp/ch6_02.png)

- 一个例子
```c++
#include <stdlib.h>
int main() {
   while(1) {
       sleep(1000);
   }
}

$gcc -static SectionMapping.c -o SectionMapping.elf
```
![ch6_03.png](assets/SelfImp/ch6_03.png)
![ch6_04.png](assets/SelfImp/ch6_04.png)
![ch6_05.png](assets/SelfImp/ch6_05.png)

- 程序头表
```c++
typedef struct
{
  Elf32_Word    p_type;                 /* Segment type */
  Elf32_Off     p_offset;               /* Segment file offset */
  Elf32_Addr    p_vaddr;                /* Segment virtual address */
  Elf32_Addr    p_paddr;                /* Segment physical address */
  Elf32_Word    p_filesz;               /* Segment size in file */
  Elf32_Word    p_memsz;                /* Segment size in memory */
  Elf32_Word    p_flags;                /* Segment 权限属性 */
  Elf32_Word    p_align;                /* Segment alignment */
} Elf32_Phdr;
```

### 6.3 堆栈
![ch6_06.png](assets/SelfImp/ch6_06.png)

- col1:VMA 的范围
- col2:VMA 权限，rwx可读可写可执行，p 表示私有（COW）, s表示共享
- col3:VMA对应的Segment 在文件中的偏移
- col5:映射文件节点号
- vdso:内核相关

一个进程基本上可以分为如下几种ⅤMA区域
　　a. 代码VMA,权限只读、可执行;；有映像文件。
　　b. 数据VMA,权限可读写、可执行；有映像文件。
　　c. 堆VMA,权限可读写、可执行；无映像文件,匿名,可向上扩展。
　　d. 栈VMA,权限可读写、不可执行；无映像文件,匿名,可向下扩展
![ch6_06.png](assets/SelfImp/ch6_06.png)

### 6.4 Linux 内核装载ELF 过程简介
- bash 进程fork 一个进程
- 新进程调用execve族进程
```c++
//minibash
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
int main(){
    char buf[1024] = {0};
    pid_t pid;
    while(1){
        printf("minibash$");
        scanf("%s",buf);
        pid = fork();
        if(pid == 0){
            if(execlp(buf,0) < 0)
                printf("exec error\n");                        
        }else (pid > 0){
            int status;
            waitpid(pid, &status, 0);                                                
        }else{
            printf("fork error!\n");        
        }    
    }
    return 0;
}
```
- 进入对应的系统调用入口， sys_exceve(), 检查及复制参数
- 调用do_exceve(),查找被执行文件，并读取128字节
- 调用serch_binary_handle(),根据读入的128字节判断文件格式，并调用对应的处理过程
    - 如‘elf’,'cafe'(java 使用)，‘#!/bin/bash’等
- 调用load_elf_binary()
    - 检查ELF 格式的有效性， 如魔数， 程序头表中Segment的数量
    - 寻找动态链接的.interp 段，设置动态链接器路径
    - 根据可执行文件程序头的描述，对ELF 进行映射
    - 初始化ELF 进程环境
    - 将系统调用返回地址设为ELF 可执行文件的入口， 静态链接的ELF可执行文件为e_entry所指的地址， 动态链接的ELF可执行文件为动态链接器
    - 返回系统调用sys_execve,进入用户态时，EIP 寄存器直接跳转到ELF程序入口地址， 开始新程序执行
	
	
## 7. 动态链接
### 7.1 一个简单的例子
```c++
/*Program1.c*/
#include "Lib.h"
int main(){
    foobar(1);
    return 0;
}
/*Program2.c*/
#include "Lib.h"
int main(){
    foobar(2);
    return 0;
}
/*Lib.c*/
#include <stdio.h>
void foobar(int i){
    printf("Printing from Lib.so %d",i);
    sleep(-1);
}
/*Lib.h*/
#ifndef LIB_H
#define LIB_H
void foobar(int i);
#endif

gcc -fPIC -shared -o Lib.so Lib.c
gcc -o Program1 Program1.c Lib.so
gcc -o Program1 Program2.c Lib.so
```
![ch7_01.png](assets/SelfImp/ch7_01.png)

- foobar是一个定义在某个动态共享对象中的函数,链接器会将这个符号的引用标记为一个动态链接的符号,不对它进行地址重定位,把这个过程留到装载时再进行
- Lib.so中保存了完整的符号信息(运行时进行动态链接还须使用符号信息),把Lib.so也作为链接的输入文件之一,链接器在解析符号时就可以知道:foobar是一个定义在Lib.so的动态符号。这样链接器就可以对 foobar的引用做特殊的处理,使它成为一个对动态符号的引用
- 动态链接程序运行时地址空间分布
![ch7_02.png](assets/SelfImp/ch7_02.png)
![ch7_03.png](assets/SelfImp/ch7_03.png)
地址空间说明：
	- ./Lib.so同样被映射到了VMA 
    - ld-2.6.1.so linux 下的动态链接库,系统开始program 之前把控制权交给动态链接器， 完成动态链接之后再把控制权交给program1
    - libc-2.6.1.so 动态链接形式的c 语言运行库
    - 除了elf 文件类型的区别之外，动态链接模块的装载地址时从0x00开始的，但是映射文件可以看出实际地址是0xb7efc000, 由此推断， 共享对象的最终装载地址在编译时是不确定的

### 7.2 地址无关代码
- share 装载时重定位，对于so来说，如果数据段中有绝对地址引用则编译器和链接器生成一个重定位表，动态链接据此进行重定位,数据部分各进程有一份副本没有问题， 但指令部分修改后无法再多个进程中共享，如此失去其节省内存的优势
- fPIC 地址无关代码，通过将指令中需要修改的部分分离出来跟数据部分放在一起实现
![ch7_04.png](assets/SelfImp/ch7_04.png)
    1. 模块内部调用或跳转
        - 相对跳转和调用，存在全局符号介入问题，后续介绍
    2. 模块内部数据访问
        - 相对地址访问
    3. 模块间数据访问
        - 间接访问GOT
![ch7_05.png](assets/SelfImp/ch7_05.png)
![ch7_06.png](assets/SelfImp/ch7_06.png)
![ch7_07.png](assets/SelfImp/ch7_07.png)
    4. 模块间的调用跳转
        - 间接跳转和调用GOT（GOT 位于数据段）,存在性能问题， elf 有更好的处理， 后续介绍
![ch7_08.png](assets/SelfImp/ch7_08.png)
![ch7_09.png](assets/SelfImp/ch7_08.png)
![ch7_10.png](assets/SelfImp/ch7_08.png)
    5. 模块间的调用跳转
  - 间接跳转和调用GOT（GOT 位于数据段）,存在性能问题， elf 有更好的处理， 后续介绍
![ch7_11.png](assets/SelfImp/ch7_11.png)
    6. 模块内全局变量
        - 如果lib.so 定义了一个全局变量G,而进程A，B 都使用了lib.so, AB 修改G 时互不影响，因为G 已经变成了副本
        - 如果将上述情况改为线程AB， 则G 的修改对AB 相互可见
        - 有种需求希望情况1 相互可见实现进程间通讯， 情况2 不可见防止线程间干扰， 后续介绍解决方案
		```
		//一个共享对象定义了global
//module.c
extern int global;
int foo(){
    global = 1;
}
//此时无法判断global是定义在其他目标文件还是另一个共享对象中
//ELF编译时将模块内的全局变量当初定义在其他模块的变量处理， 即情况4
//共享模块被加载时，如果可执行文件中有副本， 则动态链接器将GOT指向
//正确的地址，反之则指向模块内部的该变量副本
		```

> - -fpic和-fPIC的区别在于前者有平台限制， 故通常使用后者
> - 使用readelf -d foo.so |grep TEXTREL 有输出则表示该DSO 是PIC 的，TEXTREL表示代码段重定位地址
> - 对于可执行文件来说， 如果可执行文件时动态链接的，那么gcc 会使用PIC的方法来产生可执行文件的代码段

### 7.3 延迟绑定PLT
- 由动态链接器完成， 函数第一次使用时才被加载Glibc由 _dl_runtime_resolve（）进行函数的绑定工作
    - .got 保存全局变量的引用
    - .got.plt 保存函数引用的地址，即外部函数的引用全部被分离至此
        - 第一项保存”.dynamic“的段地址
        - 第二项保存本模块的ID
        - 第三项保存_dl_runtime_resolve 函数地址
            - 第二三项由动态链接器在装载共享模块时负责初始化
        - 其余项为外部函数引用
![ch7_12.png](assets/SelfImp/ch7_12.png)
	- .plt 存放plt 代码， 地址无关故与代码段合成一个Segment装入内存

### 7.4 动态链接的相关结构

- ”.interp”(interpreter 的缩写)段：ELF映射到VMA后，OS先启动一个动态链接器完成动态链接器工作， 该段负责指名动态链接器的位置
![ch7_13.png](assets/SelfImp/ch7_13.png)
- “.dynamic”段：该段保存了动态链接器的基本信息
```c++
//elf.h
typedef struct{
    Elf32_Sword d_tag;
    union {
        Elf32_Word d_val;
        Elf32_Addr d_ptr;            
    }d_un;
}Elf32_Dyn;
```
![ch7_14.png](assets/SelfImp/ch7_14.png)
![ch7_15.png](assets/SelfImp/ch7_15.png)
- “.dynsym”:动态符号表， 只保存了与动态链接相关的符号（.symtab 包含所有符号，包括.dynsym 中的）；“.dynstr”动态符号字符串表；“.hash”:加快符号的查找过程
![ch7_16.png](assets/SelfImp/ch7_16.png)
- “.rel.dyn”,".rel.plt"类似于".rel.text",".rel.data", 是专门用于动态链接重定位的段， 前者对 数据引用进行修正， 修正位置为.got 和数据段， 后者对函数引用进行修正， 作用位置于.got.plt
![ch7_17.png](assets/SelfImp/ch7_17.png)
![ch7_18.png](assets/SelfImp/ch7_18.png)


### 7.5 动态链接的步骤和实现

- 动态链接器自举：动态链接器本身是一个共享对象，其入口地址即使自举代码入口，操作系统将进程控制权交给它时开始执行自举
    - 首先找到自己的GOT, 找到链接器本身的.daynamic
    - 通过.daynamic 找到链接器自身的重定位表和符号表，重定位它们
    - 动态链接器开始使用自己的全局变量和静态变量
- 装载共享对象
    - 完成自举后，动态链接器将可执行文件和链接器本身的符号表合成全局符号表
    - 根据.dynamic 查找需要的共享文件名字， 并将它们放入装载集合中
    - 找到文件后读取ELF 和.dynamic 将之对应的数据段代码段映射到内存空间中
    - 如果该文件还依赖于其他共享文件，则将所以依赖的共享对象名字放入装载集合，循环直至所有的共享对象被装载进来
    - 当一个共享对象被装载时，符号表会被合并入全局符号表中
         - 为避免全局符号接入，linux 采用如果存在相同的符号名则后加入的符号被忽略的策略
```c++
//a1.c
void a(){
    printf("a1.c\n");
}
//a2.c
void a(){
    printf("a2.c\n");
}
//b1.c
void a()
void b1(){
   a();
}
//b2.c
void a()
void b2(){
   a();
}
gcc -fPIC -shared a1.c -o a1.so
gcc -fPIC -shared a2.c -o a2.so
gcc -fPIC -shared b1.c a1.so -o b1.so
gcc -fPIC -shared b2.c b2.so -o b2.so
//main.c
void b1();
void b2();
int main(){
    b1();
    b2();
    return 0;
}
```
![ch7_19.png](assets/SelfImp/ch7_19.png)
- 重定位与初始化
    - 重定位完成后,如果某个共享对象有.init且可执行代码中没有该段,那么动态链接执行器会执行.init 段中的代码进行初始化， .finit 同理亦同

### 7.6 显式运行时链接
- void* dlopen(const char* filename,int flag) ：打开一个动态库并加载到进程地址空间，完成初始化工程
    - arg1 为相对路径时按照以下顺序查找动态库文件
        - LD_LIBRARY_PATH指定的路径
        - /etc/ld.so.cache指定的共享路径
        - /lib,/usr
    - arg2 RTLD_LAZY 表示延迟绑定，RTLD_NOW表示一次性加载， 二者必其一，RTLD_GLOBAL 表示将被加载的文件的符号加载到全局符号中
    - 返回句柄， dlsym，dlclose使用,失败时返回NULL
- void *dlsym(void *handle, char *symbol):查找所需要的符号
    - 根据dlerror来判断成功与否
    - 如果查找的符号是一个函数，则返回函数地址
    - 如果查找的符号时个常量，则返回该常量的值
- char *dlerror():
    - 如果成功返回 NULL, 否者返回错误信息
- dlclose()
    - 对模块采取引用计数方式，为0时真正卸载掉 

## 8. Linux 共享库的组织
1. 共享库版本命名
- libname.so.x.y.z
    - x 主版本号，表示库的重大升级，不同主版本号之间不兼容
    - y 此版本号，表示库的增量升级，向前兼容
    - z 发布版本号， 表示库的一些修正，性能的改进等，完全兼容
- SO-Name
    - 保留so的主版本号，即libname.so.x，通常使用软连接方式连接到具体版本
    - ldconfig 可以遍历所有默认共享库更新所有的软连接
    -  -lXXX 会更更具系统中的相关路径查找最新的xxx库
- 符号版本机制略

2. 共享库系统路径（FH 标准）
- /lib存放最关键和基础的库，主要是/bin,/sbin和系统启动需要的库
- /usr/lib 存放非系统运行时所需要的关键性的库，主要是一些开发时用到的共享库
- /usr/local/lib 存放OS不相关的库，主要是第三方应用程序的库
3. ldconfig 负责创建，删除或更新相应的SO-NAME, 并集中存放到/etc/ld.so.cache 里边，当动态链接器需要查找共享库时可以直接从ld.so.conf 查找
4. 环境变量
- LD_LIBRARY_PATH 
- LD_PRELOAD 预加载，尽量避免
- LD_DEBUG
5. strip 可以清除共享库或可执行文件的所有符号和调试信息

## 10. 内存
### 10.1 程序的内存布局
![ch10_01.png](assets/SelfImp/ch10_01.png)
### 10.2 堆与内存管理
- 为了加快内存分配的速度，运行库适当的向OS 申请一块较大的内存，按需分配给程序使用，运行库使用brk()或mmap 申请内存
- 堆分配算法
    - 空闲链表法
    - 位图
    - 对象池
    - 对于glibc 来说，小于64字节采用类似对象池的方法，大于512字节使用最佳适配法，大于64-512的使用上述方法折中，大于128KB 的使用mmap 直接向OS 申请