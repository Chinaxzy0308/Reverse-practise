# Thread Environment Block
### *Definition：*
- 每个线程独有的数据结构，存储线程特定的信息。它位于进程的用户模式内存中
- 在x86架构中：FS：[0x18]
- 在x64架构中：GS
- TEB是一个操作系统定义的结构，布局和字段由windows内核和ntdll.dll共同维护
- 
### *apply:*
- ntdll! TED 表示TEB结构的符号定义来源于ntdll.dll的PDB
- 
### *BOOL GetThreadSelectorEntry( 
[in] HANDLE hThread, 
[in] DWORD dwSelector, ；特殊编号，指向某一个段
[out] LPLDT_ENTRY lpSelectorEntry );*
作用：查找某个段的具体信息

### *什么是LDT？，它里面有什么？*
- 局部描述符表，每个线程自己的表（私有部分）
### *什么是GDT？，它里面有什么？*
- 全局描述符表：所有线程共享的全局表（公共部分）

### 什么是FS？
- FS是x84架构中的一个特殊段寄存器，他存储一个选择器（指向某一个段）在win32中一般指向的“线程信息块（TIB）”
- 他在每个线程的LDT里定义，确保每个线程的FS独一无二
- ![[4adddf8c0d0ab8b75f946d5e2affdef.jpg]]

### 什么是


---
# Process Environment Block

### *Definition:*
- 与TEB类似
- PEB 位于进程的虚拟地址空间（用户模式内存），通过 TEB 的 ProcessEnvironmentBlock 字段（偏移 +0x030）访问。
- 
### *Importance*
- **NtGloablFlag**:控制进程的运行时行为，特别是在调试或开发环境中。
- **Ldr**:存储了_PEB_LDR_DATA的地址
- 
### *ntdll!_UNICODE_STRING
   +0x000 Length        : Uint2B
   +0x002 MaximumLength : Uint2B
   +0x004 Buffer        : Ptr64 Wchar*

![[298e852f5599ec57b62a77b864f1edb 1.jpg]]

---
# Windbg的使用
- !teb
- !peb
- dt 地址 结构名字
- dd ：分析dword
```cpp
//Unicode string 定义：
typedef struct _UNICODE_STRING {
    USHORT Length;        // 偏移 +0x00
    USHORT MaxLength;     // 偏移 +0x02
    ULONG  Padding;       // 偏移 +0x04 （对齐用）
    PWCH   Buffer;        // 偏移 +0x08 (总共偏移 +8)
};
```
0:000> dd 006464a8
006464a8  00646390 7772eb2c 00646398 7772eb34
006464b8  00000000 00000000 00400000 00401000
006464c8  00004000 00360034 00643a04 00120010
006464d8  00643a28 800022cc 0000ffff 7772e9b0
006464e8  7772e9b0 68060e5d 00000000 00000000
006464f8  00646578 00646578 00646578 0019f838
00646508  00000000 776010ec 00000000 00649260
这里的00360034 00643a04就是一个Unicode string：
- 0x34h=52，表示Lenth为52，可以表示26个字符
- 0x36-54:MaxLength
- 00643a04:Buffer
00646518  00648ac8 0064958c 00646d04 00000000
---

# 结构化异常处理
- 筛选器异常是异常的最后处理时机，判断要不要处理，不处理就退出了，而结构化异常就是整套异常处理逻辑
- 结构化异常处理是Windows操作系统提供的异常处理机制，但对于不同的程序使用的不同指令集有不同的实现方式：
	- x64的异常处理是用一张异常表来处理的，在x64程序中ExceptionList指向全零
	- x86是通过一个异常注册链表实现的：EXCEPTION_REGISTRATION_RECODE
- 异常处理结构体：编译器在函数进入时**自动构造在栈上的结构体**
  typedef struct _EXCEPTION_REGISTRATION_RECORD
{
     [PEXCEPTION_REGISTRATION_RECORD](https://www.nirsoft.net/kernel_struct/vista/EXCEPTION_REGISTRATION_RECORD.html) Next;
     [PEXCEPTION_DISPOSITION](https://www.nirsoft.net/kernel_struct/vista/EXCEPTION_DISPOSITION.html) Handler;
} EXCEPTION_REGISTRATION_RECORD, *PEXCEPTION_REGISTRATION_RECORD;
- 来看一个简单的异常处理：
```cpp
typedef struct _EXCEPTION_RECORD {
    DWORD ExceptionCode;
    DWORD ExceptionFlags;
    struct _EXCEPTION_RECORD* ExceptionRecord; // 支持嵌套异常
    PVOID ExceptionAddress;
    DWORD NumberParameters;
    ULONG_PTR ExceptionInformation[15];
} EXCEPTION_RECORD, *PEXCEPTION_RECORD;

DWORD scratch;

EXCEPTION_DISPOSITION __cdecl _except_handler(
    struct _EXCEPTION_RECORD *ExceptionRecord,//描述异常信息
    void *EstablisherFrame,//指向seh结构
    struct _CONTEXT *ContextRecord,//寄存器信息
    void *DispatcherContext )//上下文，通常为NULL
{
    printf("Hello from an exception handler\n");
    ContextRecord->Eax = (DWORD)&scratch;
    return ExceptionContinueExecution;
}
//这里有一个局部可用的地址：scratch,将eax指向其后即可修复错误
int main()
{
    DWORD handler = (DWORD)_except_handler;
    __asm
    {
        // 创建EXCEPTION_REGISTRATION结构：
        push handler // handler函数的地址
        push FS : [0] // 前一个handler函数的地址
        mov FS : [0] , ESP // 安装新的EXECEPTION_REGISTRATION结构
    }
    __asm
    {
        mov eax, 0     // 将EAX清零
        mov[eax], 1 // 写EAX指向的内存从而故意引发一个错误
    }
    printf("After writing!\n");
    __asm
    {
        // 移去我们的EXECEPTION_REGISTRATION结构
        mov eax, [ESP]    // 获取前一个结构
        mov FS : [0] , EAX // 安装前一个结构
        add esp, 8       // 将我们的EXECEPTION_REGISTRATION弹出堆栈
    }
    return 0;
}
```







---
# Think Deeply
- 怎么理解：_编译器在函数进入时**自动构造在栈上的结构体**
- 为什么简单的异常处理代码总是卡在mov [eax],1