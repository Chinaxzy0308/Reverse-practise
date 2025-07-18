# Patch
找入口——过程函数
- 必须先懂正向开发
- 在逆向的过程中分析正向逻辑在哪

# Hook
- 基本方式：修改函数开始的五个字节：跳到自己的函数那里，自己的函数帮忙做这个五个字节原来做的事情，然后跳回来。
- 改变代码执行逻辑，让其跳转到指定位置
- jmp的偏移是从下一条指令开始的
- 我们想要改变某一个动态链接库里面某个函数的功能，并且只对特定进程有效
### 具体实现思路：
- 修改dllmain：执行InstallHook
- InstallHook:获取特定dll句柄-获取特定函数地址-计算特定函数地址到HookCode地址-修改特定函数逻辑使其跳转到HookCode(注意要修改特定函数内存可读可写可执行权限)
### 主动防御：
- 主动监控进程干了什么（比如在别人程序里申请了内存，写入了代码，创建线程...)
- 早期主动防御就是给api挂钩子，查看进程调用了哪些api


### 函数总结表

| **函数名**                      | **功能**                | **参数**                                                                                                                                                                                                                                                                                                                                                                  | **返回值**                             | **备注**                                       |
| ---------------------------- | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- | -------------------------------------------- |
| **FindWindowA**              | 查找具有指定类名或窗口名称的顶层窗口句柄。 | 1. lpClassName (LPCSTR): 窗口类名（可为 NULL）。<br>2. lpWindowName (LPCSTR): 窗口标题（此处为 offset windowName）。                                                                                                                                                                                                                                                                       | 成功：窗口句柄 (HWND)。<br>失败：NULL。         | 用于通过窗口标题（如 "target"）定位目标进程的窗口。需检查返回值以确保窗口存在。 |
| **GetWindowThreadProcessId** | 获取指定窗口的线程 ID 和进程 ID。  | 1. hWnd (HWND): 窗口句柄（来自 FindWindowA 的 eax）。<br>2. lpdwProcessId (LPDWORD): 接收==进程 ID 的指针==（此处为 offset dwProcId）。                                                                                                                                                                                                                                                        | 成功：创建窗口的==线程 ID ==(DWORD)。<br>失败：0。 | 用于从窗口句柄获取目标进程的 PID，存储到 dwProcId 中。           |
| **OpenProcess**              | 打开指定进程并返回其句柄。         | 1. dwDesiredAccess (DWORD): 访问权限（此处为 PROCESS_ALL_ACCESS）。<br>2. bInheritHandle (BOOL): 句柄是否可继承（FALSE）。<br>3. dwProcessId (DWORD): 进程 ID（应为 dwProcId，代码中错误使用了 eax）。                                                                                                                                                                                                      | 成功：进程句柄 (HANDLE)。<br>失败：NULL。       | 获取目标进程句柄，用于后续内存操作和线程创建。建议使用更具体的权限组合。         |
| **VirtualAllocEx**           | 在指定进程的虚拟地址空间中分配内存。    | 1. hProcess (HANDLE): 目标进程句柄（hTargetProcess）。<br>2. lpAddress (LPVOID): 建议地址（NULL 让系统选择）。<br>3. dwSize (SIZE_T): 分配大小（100h = 256 字节）。<br>4. flAllocationType (DWORD): 分配类型（MEM_COMMIT）。<br>5. flProtect (DWORD): 保护属性（PAGE_READWRITE）。                                                                                                                                  | 成功：分配内存的基地址 (LPVOID)。<br>失败：NULL。   | 为 DLL 路径分配内存。建议使用 MEM_COMMIT or MEM_RESERVE。 |
| **WriteProcessMemory**       | 将数据写入目标进程的内存区域。       | 1. hProcess (HANDLE): 目标进程句柄（hTargetProcess）。<br>2. lpBaseAddress (LPVOID): 目标地址（pRemoteMem）。<br>3. lpBuffer (LPCVOID): 源数据地址（addr dllPath）。<br>4. nSize (SIZE_T): 写入字节数（ebx）。<br>5. lpNumberOfBytesWritten (SIZE_T*): 实际写入字节数（代码中缺失）。                                                                                                                                  | 成功：非零值 (BOOL)。<br>失败：0。             | 将 DLL 路径写入目标进程的分配内存。需提供第五个参数。                |
| **GetModuleHandleA**         | 获取指定模块（DLL 或 EXE）的句柄。 | 1. lpModuleName (LPCSTR): 模块名（addr kernel32Name）。                                                                                                                                                                                                                                                                                                                       | 成功：模块句柄 (HMODULE)。<br>失败：NULL。      | 获取 kernel32.dll 的句柄，用于查找 LoadLibraryA 地址。    |
| **GetProcAddress**           | 获取指定模块中导出函数的地址。       | 1. hModule (HMODULE): 模块句柄（GetModuleHandleA 的 eax）。<br>2. lpProcName (LPCSTR): 函数名（addr loadLibraryName）。                                                                                                                                                                                                                                                               | 成功：函数地址 (FARPROC)。<br>失败：NULL。      | 获取 LoadLibraryA 的地址，存储到 loadLibraryAddr。     |
| **CreateRemoteThread**       | 在目标进程中创建并运行线程。        | 1. hProcess (HANDLE): 目标进程句柄（hTargetProcess）。<br>2. lpThreadAttributes (LPSECURITY_ATTRIBUTES): 安全属性（NULL）。<br>3. dwStackSize (SIZE_T): 线程栈大小（0 表示默认）。<br>4. lpStartAddress (LPTHREAD_START_ROUTINE): 线程入口函数（loadLibraryAddr）。<br>5. lpParameter (LPVOID): 传递给线程的参数（pRemoteMem）。<br>6. dwCreationFlags (DWORD): 创建标志（0）。<br>7. lpThreadId (LPDWORD): 接收线程 ID 的指针（NULL）。 | 成功：线程句柄 (HANDLE)。<br>失败：NULL。       | 创建线程以调用 LoadLibraryA，加载 DLL。                 |


---
# Detour
注入需要准备什么？- 原函数地址 - 代理函数地址
- `DetourRestoreAfterWith();`
	通常用于 DLL 注入后的恢复现场，避免主程序崩溃
- `DetourTransactionBegin();`
	开始一个 Hook 事务（Transaction）。Detours 的 Hook 操作是批处理式的，必须放在事务中间才会生效。
- `DetourUpdateThread(GetCurrentThread());`
	告诉 Detours 哪个线程正在进行 Hook 操作。通常传入当前线程。
- `DetourAttach((PVOID*)&TrueMessageBoxA, MyMessageBoxA);`
	将目标函数（原始函数）与你的钩子函数进行连接（Attach）。
	- 第一个参数是二级函数指针-原函数地址
	- 第二个参数是函数地址- 代理函数的地址
- `DetourTransactionCommit();`
	提交事务，让之前设置的 Hook 正式生效。
- `DetourDetach((PVOID*)&TrueMessageBoxA, MyMessageBoxA);`
	撤销（解除）之前的 Hook。
### 函数指针：
```cpp
// 普通函数
int myFunc(int);

// 函数指针（指向函数的变量）
int (*funcPtr)(int);

// 带调用约定的函数指针（比如 Windows 的 __stdcall）
int (__stdcall* funcPtr)(int);

```

# Detour底层
- 在detour的底层，也是做内联hook(修改函数前五个字节为跳转)，但是，在DetourAttach(PVIOD* a,PVOID b)后,她会给一个trampoline函数的地址给 a,这个trampoline函数就是被修改的原五个字节，然后跳转到原函数的接下来部分。

---
# stream hook
- 像cs2这种游戏一般用imgui（Immediate Mode GUI），里面有很多32位的dll,想要hook这些里面函数需要cpp逆向知识
- cheat engine:可以查找特定指令在运行的时候
