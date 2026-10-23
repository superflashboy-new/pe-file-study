# PE 文件学习归档

信息安全方向的学习记录。仓库按「思维导图 → 周报」分类，每期周报单独成目录，便于按周查阅和追加。

## 仓库结构

| 目录 | 内容 | 文件数 |
| --- | --- | --- |
| [`01-PE文件思维导图/`](01-PE文件思维导图/) | PE 文件结构思维导图 | 1 |
| [`02-第二次周报/`](02-第二次周报/) | 第二次周报（扩展版） | 2 |
| [`03-第三次周报/`](03-第三次周报/) | 第三次周报（标准版 + 扩展版） | 4 |

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

## 更新记录

- **2026-10-16**：收录 `PE文件.pdf` 与第二次周报（扩展版）。
- **2026-10-23**：按「导图 / 各期周报」重新分类归档，新增第三次周报（标准版 + 扩展版，共 4 个文件）。

## 说明

本仓库仅用于个人学习记录与课程作业存档。
