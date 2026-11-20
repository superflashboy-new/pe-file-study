# PE 文件学习归档

信息安全方向的学习记录。仓库按「思维导图 → 周报」分类，每期周报单独成目录，便于按周查阅和追加。

## 仓库结构

| 目录 | 内容 | 文件数 |
| --- | --- | --- |
| [`01-PE文件思维导图/`](01-PE文件思维导图/) | PE 文件结构思维导图 | 1 |
| [`02-第二次周报/`](02-第二次周报/) | 第二次周报（扩展版） | 2 |
| [`03-第三次周报/`](03-第三次周报/) | 第三次周报（标准版 + 扩展版） | 4 |
| [`04-第四次周报/`](04-第四次周报/) | 第四次周报（扩展版） | 2 |
| [`05-第五次周报/`](05-第五次周报/) | 第五次周报（扩展版） | 2 |
| [`06-第六次周报/`](06-第六次周报/) | 第六次周报（标准版 + 扩展版） | 3 |

约定：每个周报目录内同时放 `PDF`（阅读版）与 `DOCX`（可编辑源文件）；同一期有扩展版时，文件名带「（扩展版）」后缀以示区分。

---

## 01 · PE 文件思维导图

| 文件 | 格式 | 说明 |
| --- | --- | --- |
| [`PE文件.pdf`](01-PE文件思维导图/PE文件.pdf) | PDF（1 页） | PE 结构思维导图，覆盖 DOS 头 / NT 头 / 可选 PE 头 / 数据目录 / 区段头，以及导出表与导入表 |

### 要点

- **DOS 头（IMAGE_DOS_HEADER）**：`e_magic` 固定为 `5A4D`（`MZ`），`e_lfanew` 给出 NT 头相对文件开头的偏移。
- **NT 头（IMAGE_NT_HEADERS）**：由 `Signature`、文件头（标准 PE 头）与可选 PE 头组成。
- **文件头（IMAGE_FILE_HEADER）**：`Machine`、`NumberOfSections`、`SizeOfOptionalHeader`、`Characteristics`。
- **可选 PE 头（IMAGE_OPTIONAL_HEADER）**：`Magic`、`SizeOfCode`、`AddressOfEntryPoint`（真实入口地址 = `OEP` + `ImageBase`）、`ImageBase`、`SectionAlignment`、`FileAlignment`、`SizeOfImage`、`NumberOfRvaAndSizes`。
- **数据目录（IMAGE_DATA_DIRECTORY[16]）**：每项含 `VirtualAddress`（RVA）与 `Size`，静态读取前需先把 RVA 换算成 FOA。
- **区段头（IMAGE_SECTION_HEADER）**：`Name`（8 字节）、`VirtualSize`、`VirtualAddress`、`SizeOfRawData`、`PointerToRawData`、`Characteristics`。
- **导出表（索引 0）** 与 **导入表（索引 1）**：导出表含 `Characteristics`、`TimeDateStamp`、`Name`、`Base`、`NumberOfFunctions` 等字段；导入表以 `OriginalFirstThunk` / `FirstThunk`（对应 `IAT`）指向 `IMAGE_IMPORT_BY_NAME` 结构。

---

## 02 · 第二次周报（扩展版）

| 文件 | 格式 | 说明 |
| --- | --- | --- |
| [`第二次周报（扩展版）.pdf`](02-第二次周报/第二次周报（扩展版）.pdf) | PDF（8 页） | 阅读版 |
| [`第二次周报（扩展版）.docx`](02-第二次周报/第二次周报（扩展版）.docx) | DOCX | 可编辑源文件（WPS Office） |

### 内容摘要

学习 PE 文件的区段头、导入表和导出表，并编写 C++ 代码实现三者解析。

- **地址换算**：文件对齐与内存对齐不同，静态解析必须先把 RVA 换算成 FOA，这是解析导入表、导出表的前提。
- **区段表定位**：DOS 头找 NT 头、NT 头找标准 PE 头，再用可选 PE 头大小定位区段表；实践中用 `IMAGE_FIRST_SECTION` 宏直接定位首个区段头。
- **细节处理**：区段名是 8 字节 `BYTE` 数组，不能直接 `printf`，需放入数组后再输出。
- **实验工具**：Visual Studio、PE-bear；代码解析结果与 PE-bear 结果比对一致，实验验证通过。

---

## 03 · 第三次周报

| 文件 | 格式 | 说明 |
| --- | --- | --- |
| [`第三次周报.pdf`](03-第三次周报/第三次周报.pdf) | PDF（8 页） | 标准版 |
| [`第三次周报.docx`](03-第三次周报/第三次周报.docx) | DOCX | 标准版源文件 |
| [`第三次周报（扩展版）.pdf`](03-第三次周报/第三次周报（扩展版）.pdf) | PDF（10 页） | 扩展版，补充 TLS 表结构、IAT 生效条件与 INLINE 偏移细节 |
| [`第三次周报（扩展版）.docx`](03-第三次周报/第三次周报（扩展版）.docx) | DOCX | 扩展版源文件 |

### 内容摘要

围绕「改变程序原有执行路径」做了三个实验，切入点一层比一层底层。

**实验一 · TLS 反调试**

- TLS 位于数据目录索引 9（`IMAGE_TLS_DIRECTORY`），`AddressOfCallbacks` 指向以空指针结尾的回调函数数组，回调原型为 `void NTAPI callback(PVOID DllHandle, DWORD Reason, PVOID Reserved)`。
- 执行顺序：映射 PE 到内存 → 加载依赖 DLL → **执行 TLS 回调** → 到达入口点（OEP），因此回调可以先于初始断点运行。
- 回调在 `DLL_PROCESS_ATTACH`(1)、`DLL_THREAD_ATTACH`(2)、`DLL_THREAD_DETACH`(3)、`DLL_PROCESS_DETACH`(0) 时都会触发，这也是截图中多次输出的原因；加上 `Reason == DLL_PROCESS_ATTACH` 判断后只执行一次。
- 该手法并非绝对可靠：调试器可单独给回调下断点、跳过系统断点直接跑到 OEP，或手工摘掉 TLS 目录。

**实验二 · IAT HOOK**

- IAT 与 INT 在程序未加载到内存时内容一致；Hook 本质是劫持执行流，把原本调用的函数 A 换成自己的函数 B。
- 三步走：找到目标函数（如 `MessageBoxW`）在 IAT 中的表项 → 保存原地址 `g_unHookAddr = *g_iatAddr` → 把表项改写为自定义函数地址。
- IAT 所在页通常只读，安装与卸载钩子都需先 `VirtualProtect` 改为可写，改完再恢复原属性。
- 实验以 `MyMessageBoxW` 替换 `MessageBoxW`，输出与预期钩子函数一致，实验成功（中文「提示」乱码，故替换为 `TEST`）。

**实验三 · INLINE HOOK**

- 适用于目标函数不在 IAT 中的情况：直接改目标函数开头的指令，用一条 `jmp` 跳到自己的 Hook 函数。
- `jmp` 硬编码为 `E9`，后跟 4 字节相对偏移：`目标地址 − (jmp 指令所在地址 + 5)`，减 5 是因为 `E9` 加 4 字节偏移共占 5 字节，相对量从下一条指令起算。
- 4 字节相对偏移的跳转范围约 ±2GB，Hook 函数过远时需改用 `FF 25` 一类的绝对跳转。
- 代码段一般不可写，`memcpy` 前要先 `VirtualProtect` 开写权限；同时必须备份被覆盖的原始字节，卸载时原样写回。

---

## 04 · 第四次周报（扩展版）

| 文件 | 格式 | 说明 |
| --- | --- | --- |
| [`第四次周报（扩展版）.pdf`](04-第四次周报/第四次周报（扩展版）.pdf) | PDF（13 页） | 阅读版 |
| [`第四次周报（扩展版）.docx`](04-第四次周报/第四次周报（扩展版）.docx) | DOCX | 可编辑源文件 |

### 内容摘要

接续第三次周报的三个实验，把 TLS 回调、IAT HOOK、INLINE HOOK 从「纸面原理」落到 x64dbg / x32dbg 的内存地址与字节上。

**补充实验一 · TLS 反调试的执行流程与内存表现**

- 通过 `#pragma data_seg` 写入回调段时需以 `read` 方式保留可读写段——CRT 的 TLS 回调要求只读，被编译器优化掉会导致 Windows 加载时找不到 callback。
- x64dbg 下 `bp TLS_CALLBACK` / `bp main`：TLS 回调先命中，`MessageBoxA` 弹窗后 `ExitProcess(0)`，`main` 根本没有执行；调用堆栈停在 `ntdll.LdrInitializeThunk+E`，说明进程仍在加载器初始化阶段。
- 进入回调时 `RCX` 存放 `ImageBase`，`RDX` = 1 表示 `DLL_PROCESS_ATTACH`。
- **踩到的坑**：VS 默认开启随机基址（ASLR），PE-bear 解析出的 `0x1400111A4` 与 x64dbg 里的运行时地址 `0x00007FF72CB511A4` 对不上；两者后四位都是 `11A4`，说明 RVA 一致，关掉 ASLR 后才对齐。关闭后：`OEP = 0x14001105A`、`TLS_CALLBACK = 0x1400117D0`、`main = 0x140011B40`。

**补充实验二 · 两种 Hook 的内存变化**

- **IAT HOOK** 动的是表项：`ImageBase + RVA = 0x400000 + 0x1B090 = 0x0041B090`。
  - Hook 前：`B0 19 D1 75`（小端）= `0x757419B0`，即 `user32!MessageBoxW`；
  - Hook 后：`AF 10 01 10` = `0x100110AF`，指向自己写的 `HookMessageBoxW`；
  - 函数本体一个字节都没变，卸载时把保存的原地址写回表项即可恢复。
- **INLINE HOOK** 动的是函数本体：`0x757419B0` 处首 5 字节由 `8B FF 55 8B EC`（`mov edi,edi` / `push ebp` / `mov ebp,esp`）改为 `E9 C3 F6 8C 9A`（`jmp`）。
  - 跳转目标 = `0x757419B5 + 0x9A8CF6C3 = 0x10011078`；
  - 此时 IAT 里仍然写着 `0x757419B0`，从表项完全看不出异常。

**三种手法的区别**

| 手法 | 改动位置 | 生效时机 | 特点与代价 |
| --- | --- | --- | --- |
| TLS 回调 | 数据目录索引 9 的 TLS 目录 | 早于入口点（OEP） | 抢时间，常规断点尚未建立；可被单独下断点或摘掉 TLS 目录 |
| IAT HOOK | IAT 表项（调用链指针） | 程序运行后 | 简单可逆，但只对经由 IAT 的调用有效 |
| INLINE HOOK | 函数开头的机器码 | 程序运行后 | 覆盖面最广；需备份原始字节、处理指令长度与 ±2GB 相对偏移限制 |

**本周感悟**：调试时环境本身就是实验条件——地址对不上，先怀疑 ASLR；判断对手用了哪种 Hook，看内存里留下的是「表项变了、函数没变」还是「表项没变、函数开头多了 `E9`」，比背结论更可靠。这几周的关注点也从文件里的结构，逐步转向程序跑起来之后的行为。

---

## 05 · 第五次周报（扩展版）

| 文件 | 格式 | 说明 |
| --- | --- | --- |
| [`第五次周报（扩展版）.pdf`](05-第五次周报/第五次周报（扩展版）.pdf) | PDF（5 页） | 阅读版 |
| [`第五次周报（扩展版）.docx`](05-第五次周报/第五次周报（扩展版）.docx) | DOCX | 可编辑源文件 |

### 内容摘要

从「PE 结构 → Hook」转到「逆向分析 → 加密识别 → 样本观察」，把前四周的内容串起来。

**一、逆向分析的基本流程**

- 先判断「它是什么」：PE-bear 看 DOS 头、NT 头与区段表，靠区段名判断是否加壳，再决定主要用 IDA 还是别的工具。
- **静态分析**解决顺序问题：IDA 里从字符串窗口和导入表出发，顺着交叉引用往上找。
- **动态分析**解决「值到底是多少」：x64dbg 在关键 API、字符串读取和比较指令处下断点，看寄存器和栈。

**二、常见反调试手段**

| 类型 | 检测对象 | 代表 API / 手段 | 应对 |
| --- | --- | --- | --- |
| 调试标志 | PEB 的 `BeingDebugged`、调试端口 | `IsDebuggerPresent`、`CheckRemoteDebuggerPresent`、`NtQueryInformationProcess` | 返回值改 0，或直接清标志位 |
| 时间差 | 单步调试会被拉长的耗时 | `rdtsc`、`QueryPerformanceCounter`、`GetTickCount` | 跳过判断，或让结果落在正常范围 |
| 执行时机 | 入口点之前的回调 | TLS 回调（第三周做过） | 常规入口断点抓不到，需单独给回调下断点 |

**三、RC4 与 TEA**

- **RC4**（流密码）：256 字节状态数组 `S` + 下标 `i`、`j`。KSA 用密钥打乱 `S`，PRGA 生成密钥流后与明文逐字节异或；异或的对称性让解密流程完全一致。优点是短小快速、无需填充，恶意代码与 CTF 里很常见；缺点是密钥流一旦复用就会被破。识别特征：0~255 初始化循环、两次 `swap`、`i`/`j` 自增后取模 256、最后一步异或。
- **TEA**（分组密码）：64 位分组、128 位密钥、固定 32 轮，没有 S 盒，只用加法、异或和移位。关键常量 `delta = 0x9E3779B9`（黄金分割）让每轮 `sum` 都在变化；加密时 `sum` 从 0 累加，解密时从 `delta × 32` 递减、轮次倒过来。XTEA / XXTEA 的区别在密钥混合方式与分组长度，判断方法仍是「看常量、看轮数、看移位」。

**四、静态分析三件套**

| 工具 | 用途 |
| --- | --- |
| DIE | 查看 PE 格式与哈希值，为样本比对建立身份 |
| PE-bear | 查看 PE 头与导入表，看调用了哪些 API |
| FLOSS | 提取程序中的字符串，理解字符串与功能的关系 |

- 样本 MD5：`38994b65cd741a658f89e0d57bdacbbf`。哈希适合判断「是不是同一个文件」，不适合判断行为是否相同——重新编译、加壳或被改写都会让哈希完全变化。
- 导入表里 `CreateFileA` + `WriteFile` + `CloseHandle` + `GetCurrentProcess` 四个函数，基本可以推出这个程序会往磁盘写文件。
- 加壳特征：区段名变成 `UPX0` / `UPX1`、入口点落在无名或名字奇怪的区段、`VirtualSize` 远大于 `SizeOfRawData`（说明内存中会解压膨胀）。

**五、虚拟机与样本环境**

- 配好 Windows 10 虚拟机并安装分析软件，样本（PMNAT-labs 内、来自 GitHub 安全渠道的恶意软件）只在虚拟机里运行。
- 引爆前先打快照；网络设为仅主机或直接断开，关闭共享文件夹、剪贴板与拖放；分析结束回滚快照，避免持久化或传播污染下一次环境。

**本周小结**：第二周解析 PE 结构、第三四周做 TLS 回调与两种 Hook，这周用 DIE / PE-bear / FLOSS 看具体样本，并在虚拟机里搭出可回滚的安全观察环境。

---

## 06 · 第六次周报

| 文件 | 格式 | 说明 |
| --- | --- | --- |
| [`第六次周报.docx`](06-第六次周报/第六次周报.docx) | DOCX | 标准版（此版没有导出 PDF） |
| [`第六次周报（扩展版）.pdf`](06-第六次周报/第六次周报（扩展版）.pdf) | PDF（12 页） | 阅读版 |
| [`第六次周报（扩展版）.docx`](06-第六次周报/第六次周报（扩展版）.docx) | DOCX | 可编辑源文件 |

### 内容摘要

用一个「向文本文件写入内容」的 Windows 程序串起三次实验，分别回答「外层做了什么」「参数到底是什么」「用什么特征去检测」。

**实验一 · Procmon 动态分析文件行为**

- 先由源码预判：`CreateFileA` 打开/覆盖 `test_file_static_analysis.txt` → `WriteFile` 写入 `This is a string.` → `CloseHandle` 释放句柄，预期写入 17 字节（不含末尾 `\0`）。
- 过滤条件用进程名 `analysis_of_static.exe` 并清空无关事件；终端与 Procmon 中的 PID 都是 `18108`，确认事件来自本次运行。
- 三组事件一一对应：`CreateFileA → CreateFile`、`WriteFile → WriteFile`、`CloseHandle → CloseFile`。
- `CreateFile` 的 `Disposition = Overwritten` 对应源码的 `CREATE_ALWAYS`；`WriteFile` 的 `Length = 17` 与 `strlen("This is a string.")` 一致。
- `GetCurrentProcessId`、`GetSystemInfo` 在文件系统视图中没有对应事件——它们本身不产生文件操作。

**实验二 · x64dbg 核对 API 参数与返回值**

- x64 调用约定：前 4 个参数走 `RCX`/`RDX`/`R8`/`R9`，第 5 个及以后在栈上；`[RSP+8]`~`[RSP+0x20]` 是影子空间，因此第 5 个参数要从 `[RSP+0x28]` 开始读。
- `CreateFileA`：`RCX` 指向文件名 `0x00007FF6E286AD08`，`RDX = 0x40000000`（`GENERIC_WRITE`），`R8`、`R9` 均为 0；栈上依次是 `0x2`（`CREATE_ALWAYS`）、`0x80`（`FILE_ATTRIBUTE_NORMAL`）、`0`（模板句柄 `NULL`）。
- 返回值链路：`CreateFileA` 返回后 `RAX = 0xD8`，进入 `WriteFile` 时 `RCX` 同样是 `0xD8`，说明句柄被原样作为第一个参数传入。
- `WriteFile`：`RDX` 指向字符串缓冲区，`R8` 低 32 位为 `0x11`（17），`R9 = 0x000000C9346FF824` 指向 `bytesWritten`；调用结束后该地址变成 `11 00 00 00`，即系统把写入字节数回写了。

**实验三 · 编写并应用 YARA 规则**

- 规则由 `meta`（作者/描述/引用）、`strings`（文本或二进制模式）与 `condition`（布尔组合）三部分组成。
- 条件为 `uint16(0) == 0x5A4D`（确认 MZ/PE 头）且 `all of them`（同时包含 `MESSAGE_A` 与 `MESSAGE_B`，`ascii`、区分大小写）。
- 批量扫描结果：目标 `yara_rule.exe` 返回 `MATCH` 并列出 3 行输出，诱饵 `yara_rule_decoy.exe` 返回 `NO MATCH`、命中数 0，说明规则确实依赖完整组合条件。
- 误报风险：任何正常程序只要同时含相同字符串和 PE 头都会被命中。更稳妥的做法是把字符串特征与导入函数、文件结构或动态行为组合，并用正常样本与变体样本检验误报漏报；改大小写、加干扰字符或换编码都会改变匹配结果。

**本周总结**：Procmon 给「事件」层证据，x64dbg 给「参数与返回值」层证据，YARA 把可观察特征固化成可重复执行的检测条件——三层互相印证，而不是孤立地记函数用法。

---

## 更新记录

- **2026-10-16**：收录 `PE文件.pdf` 与第二次周报（扩展版）。
- **2026-10-23**：按「导图 / 各期周报」重新分类归档，新增第三次周报（标准版 + 扩展版，共 4 个文件）。
- **2026-10-29**：新增第四次周报（扩展版，共 2 个文件），接续 TLS / IAT HOOK / INLINE HOOK 话题，补充内存地址与字节层面的对照。
- **2026-11-12**：新增第五次周报（扩展版，共 2 个文件），覆盖逆向分析流程、反调试手段、RC4 / TEA 加密识别与恶意样本观察环境。
- **2026-11-20**：新增第六次周报（标准版 + 扩展版，共 3 个文件），覆盖 Procmon 文件行为、x64dbg API 参数核对与 YARA 规则检测。

## 说明

本仓库仅用于个人学习记录与课程作业存档。
