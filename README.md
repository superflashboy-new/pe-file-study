# PE 文件学习归档

信息安全方向的学习记录：**PE 文件结构思维导图** + **《第二次周报（扩展版）》**。

## 文件清单

| 文件 | 格式 | 说明 |
| --- | --- | --- |
| [`PE文件.pdf`](PE文件.pdf) | PDF（1 页） | PE 文件结构思维导图，覆盖 DOS 头 / NT 头 / 可选 PE 头 / 数据目录 / 区段头，以及导出表与导入表 |
| [`第二次周报（扩展版）.pdf`](第二次周报（扩展版）.pdf) | PDF（8 页） | 第二次周报（扩展版）阅读版 |
| [`第二次周报（扩展版）.docx`](第二次周报（扩展版）.docx) | DOCX | 第二次周报（扩展版）可编辑源文件，由 WPS Office 编辑 |

## 思维导图要点

`PE文件.pdf` 以思维导图形式梳理 PE 结构，重点包括：

- **DOS 头（IMAGE_DOS_HEADER）**：`e_magic` 固定为 `5A4D`（`MZ`），`e_lfanew` 给出 NT 头相对文件开头的偏移。
- **NT 头（IMAGE_NT_HEADERS）**：由 `Signature`、文件头（标准 PE 头）与可选 PE 头组成。
- **文件头（IMAGE_FILE_HEADER）**：`Machine`、`NumberOfSections`、`SizeOfOptionalHeader`、`Characteristics`。
- **可选 PE 头（IMAGE_OPTIONAL_HEADER）**：`Magic`、`SizeOfCode`、`AddressOfEntryPoint`（真实入口地址 = `OEP` + `ImageBase`）、`ImageBase`、`SectionAlignment`、`FileAlignment`、`SizeOfImage`、`NumberOfRvaAndSizes`。
- **数据目录（IMAGE_DATA_DIRECTORY[16]）**：每项含 `VirtualAddress`（RVA）与 `Size`，RVA 需经 `FOA = FileBuff + (FOA -> 导出表地址)` 换算后才能静态读取。
- **区段头（IMAGE_SECTION_HEADER）**：`Name`（8 字节）、`VirtualSize`、`VirtualAddress`、`SizeOfRawData`、`PointerToRawData`、`Characteristics`。
- **导出表（索引 0）** 与 **导入表（索引 1）**：导出表含 `Characteristics`、`TimeDateStamp`、`Name`、`Base`、`NumberOfFunctions` 等字段；导入表以 `OriginalFirstThunk` / `FirstThunk`（对应 `IAT`）指向 `IMAGE_IMPORT_BY_NAME` 结构。

## 周报摘要

本周学习 PE 文件的区段头、导入表和导出表，并编写 C++ 代码实现三者解析：

- **地址换算**：文件对齐与内存对齐不同，静态解析必须先把 RVA 换算成 FOA，这是解析导入表、导出表的前提。
- **区段表定位**：通过 DOS 头找 NT 头、NT 头找标准 PE 头，再用可选 PE 头大小定位区段表；实践中使用 `IMAGE_FIRST_SECTION` 宏直接定位首个区段头。
- **细节处理**：区段名是 8 字节 `BYTE` 数组，不能直接 `printf`，需放入数组后再输出。
- **实验工具**：Visual Studio、PE-bear；代码解析结果与 PE-bear 解析结果比对一致，实验验证通过。

## 说明

本仓库仅用于个人学习记录与课程作业存档。
