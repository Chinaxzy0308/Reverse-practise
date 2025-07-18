自己写插件是很有必要的，以后分析软件的时候少不免需要重复更改的地方
### 如何写OllyDbg插件：
- ODBGPlugindata（必需）仅返回一个字符串指针，该字符串就用于标识该插件的唯一名称。
- ODBGPugininit（必需）该团数检查插件版本是否与当前OIlyDbg对应，如果对应就执行函数中的其它操作，并执行其它回调函数，表示插件安装成功。
- Writememory：修改被调试进程的内存
- ODBG_Paused:This function triggers when a breakpoint is hit, during single-step operations like step-over, step-in, or trace completion, and for debug events or manual pauses.
- GetWindowLong/GetClassLong:可以获取窗口相关的属性

---

ollydbg有一个bug：拿过程函数有可能拿不到，是因为窗口是unicode版的，这个时候因该用getclasslonga（）而不是getclasslong（）。
思路:将保存getclasslong的地址的值改为自己写的插件里的函数的地址

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


