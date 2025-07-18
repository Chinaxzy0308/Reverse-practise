- 当异常发生时执行的一个回调函数
- 一般到筛选器异常就是处理异常的最终时机了，到这个时候基本上可以结束进程了
- 有些异常会解密代码，只有运行了异常函数才可以执行代码
- 有的时候，这个异常会检测有没有下断点、改变内存属性、下钩子，这种异常就不能让他跑
- **当应用程序中有未被捕获的异常（unhandled exception）发生时，系统会调用该函数**，以便开发者可以处理这些严重错误，或记录错误信息、生成 dump 文件等。
### 语法
```
LPTOP_LEVEL_EXCEPTION_FILTER SetUnhandledExceptionFilter(
  [in] LPTOP_LEVEL_EXCEPTION_FILTER lpTopLevelExceptionFilter
);
```
- lpTopLevelExceptionFilter是指向顶级异常筛选器函数的指针，当异常发生时，会调用该函数
### 顶级异常筛选器函数 语法
```
LONG UnhandledExceptionFilter(
  [in] _EXCEPTION_POINTERS *ExceptionInfo
);
```
- ExceptionInfo是指向_EXCEPTION_POINTERS的指针
```
typedef struct _EXCEPTION_POINTERS {
  PEXCEPTION_RECORD ExceptionRecord;//异常信息
  PCONTEXT          ContextRecord;//异常上下文
} EXCEPTION_POINTERS, *PEXCEPTION_POINTERS;
```
**当异常发生时，运行SetUnhandledExceptionFilter指定的函数，这个函数接收一个异常事件结构体（异常信息、异常上下文）并且把做出的改变写入这个结构体里面**

---
# WinDbg
### 自单步反软件断点
- 调试器打断点的方式：修改特定内存为cc
- TF标志位：当置为1时，程序每执行一条指令就报出异常
**由此可以设计一种反调试工具：每执行一条语句就报出异常：检查指令第一个字节是否为cc，如果是cc就证明有调试器，如果不是就继续把TF置为1，执行下一条语句**（全部检查消耗太大了，最好检测特定范围）

---

### WinDbg的使用
### 一、`.pdb` 文件的生成

`.pdb`（Program Database）文件是一种**调试符号文件**，它保存了程序中变量名、函数名、源代码行号等调试信息。

#### 如何生成 `.pdb` 文件？

在使用编译器（如 MSVC）编译时，开启调试选项即可自动生成：

- **Visual Studio** 项目中：
    
    - 选择 `Debug` 模式
        
    - 项目属性 → C/C++ → 常规 → Debug Information Format（调试信息格式）→ 设置为 `Program Database (/Zi)`
        
    - 链接器 → 调试 → 生成调试信息（/DEBUG）→ 设置为 `Yes`
        

生成的 `.pdb` 文件一般与 `.exe` 或 `.dll` 保持同名（例如 `main.exe` 对应 `main.pdb`），供 WinDbg 等调试器读取符号使用。

---

### 二、`bp` / `bl` / `bc` / `bm` / `bu` 命令

#### 1. `bp`（BreakPoint）：设置断点

语法：`bp <地址或函数名>`  
示例：

bash

复制编辑

`bp main             ; 在 main 函数下断 bp 0x00401000       ; 在指定地址下断`

#### 2. `bl`（Break List）：列出当前所有断点

显示断点编号、地址、命中次数、是否启用等信息。

#### 3. `bc`（Break Clear）：清除断点

语法：`bc <断点编号或*号>`  
示例：

`bc 0        ; 清除编号为 0 的断点 bc *        ; 清除所有断点`

#### 4. `bm`（Break on Multiple）：对多个符号同时设置断点

语法：`bm <模块名>!<符号通配符>`  
示例：


`bm mydll!*read*     ; 在 mydll 模块中所有名字包含 read 的函数下断`

#### 5. `bu`（Break on Unresolved）：设置**延迟断点**

适用于 DLL 未加载的情况，断点在模块加载后才会生效。  
语法：`bu <模块名>!函数名`  
示例：

`bu kernel32!CreateFileW     ; 延迟断点，等待 kernel32.dll 加载`

---

### 三、`x` 相关命令（Examine symbols）

用于**查看符号表（函数、变量）**，支持通配符，类似 `nm` 命令。

语法：`x [模块名!]通配符`  
示例：

`x kernel32!*read*      ; 查看 kernel32 中包含 read 的符号 
`x myapp!main           ; 查看 myapp 模块中的 main 函数`

---

### 四、`e` 相关命令（Edit memory）

`e` 用于**修改内存内容**。  
基本语法：`e<类型> <地址> <值>`  
常用类型包括：

- `eb`：编辑一个字节（byte）
    
- `ew`：编辑一个字（word）
    
- `ed`：编辑一个双字（dword）
    
- `eq`：编辑一个四字（qword）
示例：

`eb 0x00401000 90           ; 将地址 0x00401000 的内容改为 0x90（NOP） 
`ed 0x00401000 0xDEADBEEF   ; 写入一个 DWORD`

---

### 五、`$exentry`（Executable Entry）

`$exentry` 是一个**伪寄存器**，代表程序的**入口地址**（Entry Point），相当于 PE 文件中 `AddressOfEntryPoint`。常用于：

`bp $exentry       ; 在程序入口点下断`

---

### 六、`k` 相关命令（显示调用栈）

`k` 命令用于显示**当前线程的调用栈信息（Call Stack）**。

- `k`：显示基本调用栈
    
- `kP`：包含参数的调用栈（常用于 Windows x86）
    
- `kb`：带参数和符号
    
- `kp`：显示参数
    
- `kv`：显示参数 + 局部变量（带调试符号时）
示例：

`k         ; 简单调用栈 
`kb        ; 更详细的调用栈，显示参数 
`kp        ; 显示参数列表`

---