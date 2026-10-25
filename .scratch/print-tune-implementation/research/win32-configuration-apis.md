# 核实 Windows 打印配置读写 API 与权限边界

- 日期：2026-10-24
- 范围：仅研究 Win32 打印配置 API；不实现 PrintTune。
- 主要依据：Microsoft Learn 的 Win32 / Windows 驱动文档及微软打印协议规范。
- 表述约定：“已证实事实”只写官方文档直接支持的内容；“对设计的约束”包含基于这些事实形成的工程建议，不冒充微软兼容性保证。

## 已证实事实

### 1. 队列枚举与驱动元数据不是同一次调用

1. `EnumPrinters` 支持 Level 1、2、4、5。Level 4 可快速取得本机队列和用户已建立的远程连接的名称等最少信息；Level 2 返回 `PRINTER_INFO_2` 的详细队列信息，但会对每个远程连接执行 `OpenPrinter`，远程连接失效时可能等待 RPC 超时。因此不能把 Level 2 枚举当作无阻塞的轻量列表操作。[EnumPrinters](https://learn.microsoft.com/en-us/windows/win32/printdocs/enumprinters)
2. `PRINTER_ENUM_LOCAL` 枚举本地安装的队列，`PRINTER_ENUM_CONNECTIONS` 枚举用户此前连接过的队列；`Name == NULL` 时二者组合还包括本机已连接的远程队列。3D 打印设备默认不在结果中，需相应 category 标志。[EnumPrinters](https://learn.microsoft.com/en-us/windows/win32/printdocs/enumprinters)
3. Level 2 的 `PRINTER_INFO_2` 含队列名、服务器名、端口名、驱动名、打印处理器、默认数据类型、属性及 `pDevMode` 等，但没有驱动包完整版本号和驱动环境/架构字段。`EnumPrinters` 也不返回安全描述符，Level 2 结果中的该指针为 `NULL`。[PRINTER_INFO_2](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-2)、[EnumPrinters](https://learn.microsoft.com/en-us/windows/win32/printdocs/enumprinters)
4. 驱动版本和环境应对所选队列另行调用 `GetPrinterDriver(..., Level=6, ...)`，读取 `DRIVER_INFO_6`；它包含 `pName`、`pEnvironment`、`ftDriverDate`、`dwlDriverVersion`、`pszHardwareID`、制造商和提供程序等。也可使用 `EnumPrinterDrivers` 枚举服务器上已安装的驱动，而不是把全部驱动与队列的对应关系交给 `EnumPrinters`。[GetPrinterDriver](https://learn.microsoft.com/en-us/windows/win32/printdocs/getprinterdriver)、[DRIVER_INFO_6](https://learn.microsoft.com/en-us/windows/win32/printdocs/driver-info-6)、[EnumPrinterDrivers](https://learn.microsoft.com/en-us/windows/win32/printdocs/enumprinterdrivers)
5. `DRIVER_INFO_6.dwlDriverVersion` 来自驱动的版本结构，`pEnvironment` 是驱动编写目标环境。若驱动当初不是以 Level 6 添加，随后查询 Level 6 时制造商等字段可为 `NULL`，版本和日期也可为 0，故这些元数据不能假定始终存在。[DRIVER_INFO_6](https://learn.microsoft.com/en-us/windows/win32/printdocs/driver-info-6)
6. `GetPrinterDriver` 的 `pEnvironment == NULL` 表示调用应用及客户端机器的当前环境，不是目标打印服务器的环境。该函数还有明确副作用：若驱动未安装在本地计算机，它会安装该驱动。因此“读取元数据”不能笼统宣称绝对无副作用。[GetPrinterDriver](https://learn.microsoft.com/en-us/windows/win32/printdocs/getprinterdriver)
7. `EnumPrinters`、`GetPrinter`、`GetPrinterDriver` 都需容纳结构及其指向的字符串或其他数据；缓冲区不足时通过 `pcbNeeded` 得到所需大小。`GetPrinterDriver` 首次以 `cbBuf=0` 调用会失败并给出 `ERROR_INSUFFICIENT_BUFFER`，这属于探长度流程，不应误报为驱动故障。[EnumPrinters](https://learn.microsoft.com/en-us/windows/win32/printdocs/enumprinters)、[GetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/getprinter)、[GetPrinterDriver](https://learn.microsoft.com/en-us/windows/win32/printdocs/getprinterdriver)

### 2. `OpenPrinter` 权限由请求的访问掩码和队列 ACL 决定

1. `PRINTER_DEFAULTS.DesiredAccess` 用于请求权限；`pDefault == NULL` 时请求的是 `PRINTER_ACCESS_USE`。`PRINTER_ACCESS_USE` 对应基本打印，`PRINTER_ACCESS_ADMINISTER` 对应包括 `SetPrinter` 在内的管理任务，`PRINTER_ALL_ACCESS` 还包含其他标准权限。[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter)
2. `PRINTER_ACCESS_MANAGE_LIMITED` 可用于 `SetPrinter` / `SetPrinterData` 等管理任务，但仅从 Windows 8.1 起可用。它是打印管理权限，不应与“机器本地 Administrators 组成员”画等号。[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter)
3. 调用者没有所请求的权限时，`OpenPrinter` 返回 0，`GetLastError()` 为 `ERROR_ACCESS_DENIED`。所以“通常需管理员”只是现场经验概括，准确的 API 边界是：是否具有队列 ACL 授予的相应管理权限。[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter)
4. 官方 `OpenPrinter` 文档把 `SetPrinter` 整体列为管理任务，但没有给出“任何标准用户以 `PRINTER_ACCESS_USE` 打开后，Level 9 在所有队列上都必能写入”的保证。因此 Level 9 的最低可用访问掩码及受限 ACL 下的行为必须实测，不能仅凭“每用户”三字推断无需权限。[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter)、[SetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/setprinter)
5. 打印机句柄不是线程安全的；并发使用需调用方同步，或各线程分别打开。多个打印 API 也是同步/阻塞调用，耗时会受网络、服务器及驱动影响。[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter)、[GetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/getprinter)

### 3. `DocumentProperties` 的探长度、合并、提示与返回值

1. `fMode=0` 时返回驱动初始化数据所需缓冲区字节数；该大小可大于公共 `DEVMODE`，因为可能附带私有数据。分配后，以 `DM_OUT_BUFFER` 调用的输出包含驱动当前设置及私有数据，输出缓冲区由调用方分配。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)
2. 三个标志的职责不同：

   | 标志 | 官方语义 |
   |---|---|
   | `DM_IN_BUFFER` | 在提示、复制或更新前，把驱动当前设置与 `pDevModeInput` 合并；只更新输入 `DEVMODE.dmFields` 指定的公共成员，冲突时输入值优先。 |
   | `DM_OUT_BUFFER` | 把当前设置（包括私有数据）写入调用方提供的 `pDevModeOutput`。 |
   | `DM_IN_PROMPT` | 显示驱动的 Print Setup 属性表，让用户修改设置。 |

   来源：[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)。
3. 修改应用本地打印设置的官方顺序是：探长度 → 分配完整缓冲区 → `DM_OUT_BUFFER` 取当前设置 → 修改公共字段并设置相应 `dmFields` 位 → 以同一缓冲区作为输入和输出，使用 `DM_IN_BUFFER | DM_OUT_BUFFER` 让驱动合并。得到的 `DEVMODE` 可传给 `CreateDC`。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)
4. `DM_IN_BUFFER | DM_OUT_BUFFER | DM_IN_PROMPT` 可用于先以输入初始化原厂界面，再取回用户确认后的完整输出。`IDOK` 才表示用户确认；`IDCANCEL` 是取消，不能当成功导出。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)
5. 返回值必须按调用形态解释：`fMode=0` 返回所需大小；显示属性表时返回 `IDOK` 或 `IDCANCEL`；不显示属性表且成功时返回 `IDOK`；失败返回值小于 0。不能用统一的“返回字节数”规则解释所有调用。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)
6. `DocumentProperties` 的上述本地修改流程得到的是可供应用使用的 `DEVMODE`；要把默认值持久化到 Level 8 或 9，仍需相应的 `SetPrinter`。对全局默认 `DEVMODE` 调用 `SetPrinter` 前，官方明确要求先以 `DocumentProperties` 验证。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)、[SetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/setprinter)
7. `dmFields` 表示哪些成员被初始化/参与合并，不等于设备支持能力的完整证明。能力值应通过 `DeviceCapabilities` 查询，例如纸张、纸仓、双面、分辨率、份数等。[DeviceCapabilities](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-devicecapabilitiesw)

### 4. `GetPrinter` / `SetPrinter` 的 Level 8、9 均受支持，但语义不同

| Level | `GetPrinter` 读取 | `SetPrinter` 写入 | 官方语义 |
|---|---|---|---|
| 2 | `PRINTER_INFO_2` 详细队列信息，含默认 `pDevMode` | 支持 | 每用户 DEVMODE 也可通过 Level 2 设置；不应把 Level 2 当成 Level 8 的替代品。 |
| 8 | `PRINTER_INFO_8` | 支持 | 全局默认打印机设置，由打印机管理员设置，可供任何用户使用。 |
| 9 | `PRINTER_INFO_9` | **支持** | 当前用户配置文件的每用户默认设置，`DEVMODE` 存在用户注册表中。 |

来源：[GetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/getprinter)、[SetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/setprinter)、[PRINTER_INFO_8](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-8)、[PRINTER_INFO_9](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-9)、[Per-User DEVMODE](https://learn.microsoft.com/en-us/windows/win32/printdocs/per-user-devmode)。

补充边界：

1. `SetPrinter` 的 Level 参数官方列表明确包含 8 和 9；Level 非零时 `Command` 必须为 0，成功返回非零，失败返回 0。[SetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/setprinter)
2. 每用户默认值只影响特定用户或使用同一配置文件的人；它不是机器级设置。因此服务账户、计划任务账户和交互式桌面账户不能视为同一个 Level 9 作用域。[PRINTER_INFO_9](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-9)
3. 微软的每用户专题还说明：可用 `PRINTER_INFO_2` 或 `PRINTER_INFO_9` 设置每用户 DEVMODE；以 `PRINTER_INFO_8` 调用 `SetPrinter` 会把每用户 DEVMODE 重置为全局默认 DEVMODE。该文没有说会重置所有其他用户，故只能按当前调用上下文理解，其他用户的影响需实测。[Per-User DEVMODE](https://learn.microsoft.com/en-us/windows/win32/printdocs/per-user-devmode)
4. Level 8 的“全局默认”不是“该队列所有任务都被强制使用这些值”。应用可在单次打印中显式提供 `DEVMODE`；`CreateDC` 的 `pdm != NULL` 即使用传入的设备初始化数据，`pdm == NULL` 才使用用户指定的默认初始化。[CreateDCW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-createdcw)
5. `GetPrinter` 明确允许 `PRINTER_INFO_2`、`PRINTER_INFO_8`、`PRINTER_INFO_9` 的 `pDevMode` 为 `NULL`，调用方必须处理，不能无条件解引用。[GetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/getprinter)
6. 这些结构及 `GetPrinter` / `SetPrinter` 文档列出的最低客户端/服务器均为 Windows 2000；这只是 API 文档的最低支持标记，不是对现代每台驱动组合的行为保证。[PRINTER_INFO_9](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-9)
7. 打印远程协议规范明确写道 `PRINTER_INFO_9` “is not used remotely”。因此不能把本地 Win32 Level 9 支持直接外推成任意远程打印服务器都支持相同每用户读写。[MS-RPRN: PRINTER_INFO_9](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rprn/6a5df266-40c9-4661-966f-27caa209b1fd)

### 5. DEVMODE 的所有权、完整长度与私有段

1. 公共 `DEVMODEW` 后面可紧跟驱动私有数据。`dmSize` 是公共部分长度，不含私有段；`dmDriverExtra` 是私有段长度。完整连续缓冲区的逻辑长度是 `dmSize + dmDriverExtra`，私有段起点是缓冲区起点加 `dmSize`。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew)、[The DEVMODEW Structure](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/the-devmodew-structure)
2. 只复制 `sizeof(DEVMODE)` 或只复制 `dmSize` 都会漏掉私有段。缺少私有数据的缓冲区传给 `CreateDC`、`ResetDC`、`DocumentProperties` 等可能失败。[微软官方 DocumentProperties 修改指南](https://learn.microsoft.com/en-us/troubleshoot/windows/win32/modify-printer-settings-documentproperties)
3. 私有成员专属于驱动，应用可以修改公共成员，但不应解析或直接修改私有成员；其大小和内容随驱动及驱动版本变化。公共字段变化后应再由驱动合并/修正。[The DEVMODEW Structure](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/the-devmodew-structure)、[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)
4. `GetPrinter` 的输出由调用方分配，结构中的指针指向同一块返回缓冲区内的字符串或其他数据；`DocumentProperties` 的输出缓冲区同样由调用方分配。`PRINTER_INFO_8/9` 自身只含一个 `pDevMode` 指针，所以导出或暂存时必须深复制完整 DEVMODE，并在原缓冲区释放/复用前完成；不能只保存指针。[GetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/getprinter)、[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)、[PRINTER_INFO_9](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-9)
5. `SetPrinter` 是同步函数，其 Level 8/9 输入通过指针提供；调用方应在调用返回前保持结构及 DEVMODE 缓冲区有效。文档没有规定它接管调用方缓冲区所有权。[SetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/setprinter)
6. `IsValidDevmode` 只验证公共成员，不验证私有成员；只有确认实际输入缓冲区至少为 `dmSize + dmDriverExtra` 时，才可把该和作为 `DevmodeSize` 传入。因此它不能证明来自配置包的私有段可信或兼容。[IsValidDevmodeW](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/winspool/nf-winspool-isvaliddevmodew)
7. ANSI 与 Unicode 版本是不同结构/API 形态，驱动使用 `DEVMODEW`。应用应统一使用 W 系列并记录配置包的数据形态，不能混用 A/W 缓冲区或仅按本机 `TCHAR` 布局猜测文件格式。[The DEVMODEW Structure](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/the-devmodew-structure)

### 6. 回读验证与“成功”的边界

1. `DocumentProperties` 的合并步骤是必要的，因为某些公共字段变更可能不能正确工作，驱动需修正 DEVMODE；即使到了 `CreateDC`，驱动仍会验证 DEVMODE，无法转换或消费时可改用默认 DEVMODE。所以“写入 API 返回成功”与“所有输入字节均被驱动原样采用”不是同一命题。[The DEVMODEW Structure](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/the-devmodew-structure)、[CreateDCW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-createdcw)
2. `GetPrinter` 可用相同 Level 重新读取 Level 8 或 9，这为持久化回读提供了官方接口。官方没有承诺回读结果一定与输入逐字节相同。[GetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/getprinter)
3. `StartDoc` 成功返回的是打印作业标识符，`EndDoc` 成功只表示结束打印作业的 API 调用成功；`WritePrinter` 成功表示数据已交给打印后台处理路径。这些返回值都不是纸张已正确出纸、切刀或打开钱箱的证明。[StartDocW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-startdocw)、[EndDoc](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-enddoc)、[WritePrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/writeprinter)

### 7. 测试打印存在可行路径，但必须选对数据路径

1. GDI 路径可行：用 `DocumentProperties` 取得/合并完整 DEVMODE → `CreateDC(..., pdm)` 显式传入 → `StartDoc` / `StartPage` / GDI 绘制 / `EndPage` / `EndDoc` → `DeleteDC`。`StartDoc` 返回的作业 ID 可用于 `GetJob` 查询。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)、[CreateDCW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-createdcw)、[StartDocW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-startdocw)
2. RAW 直传也有官方调用链：`StartDocPrinter` → `StartPagePrinter` → `WritePrinter` → `EndPagePrinter` → `EndDocPrinter`。但 `DOC_INFO_1.pDatatype == "RAW"` 时不使用 `OpenPrinter(PRINTER_DEFAULTS).pDevMode`，RAW 文档必须自己以设备语言完整描述打印设置。因此 RAW 测试不能用来证明 DEVMODE 导入生效。[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter)、[WritePrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/writeprinter)
3. `WritePrinter` 不支持 XPS/OpenXPS 打印路径；XPS 应使用 XPS Print API。故不能把一种测试打印实现宣称为覆盖所有业务打印栈。[WritePrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/writeprinter)

### 8. 架构、远程缓存与 Windows 版本注意事项

1. “进程架构”“操作系统架构”“驱动 `pEnvironment`”是不同元数据。`GetPrinterDriver` 默认环境来自调用应用和客户端；不能由 `EnumPrinters` 的队列记录直接推断目标驱动架构。[GetPrinterDriver](https://learn.microsoft.com/en-us/windows/win32/printdocs/getprinterdriver)
2. 私有 DEVMODE 的大小和内容随驱动及版本变化；微软提供跨 OS 版本的 DEVMODE 转换机制，但这不构成“私有段可跨驱动版本、架构或 OS 任意复制”的保证。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)、[The DEVMODEW Structure](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/the-devmodew-structure)
3. 自 Windows Vista 起，`GetPrinter` 对由打印服务器托管且已有服务器连接的打印机可能从本地缓存取数；`EnumPrinters` Level 4 也使用缓存。`OpenPrinter` 的远程句柄同样可能经本地缓存，需准确远端数据时可用 `OpenPrinter2W` 加 `PRINTER_OPTION_NO_CACHE`，但会阻塞等待网络通信。[GetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/getprinter)、[EnumPrinters](https://learn.microsoft.com/en-us/windows/win32/printdocs/enumprinters)、[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter)
4. `PRINTER_ACCESS_MANAGE_LIMITED` 有 Windows 8.1 起点；Level 8/9 文档有 Windows 2000 起点；远程协议又不使用 `PRINTER_INFO_9`。版本兼容不能简化成一个“Windows 2000 以上都相同”的结论。[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter)、[PRINTER_INFO_9](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-9)、[MS-RPRN: PRINTER_INFO_9](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rprn/6a5df266-40c9-4661-966f-27caa209b1fd)
5. 对 Windows 10/11 的新驱动开发，微软推荐现代打印平台（IPP 收件箱类驱动与 Print Support Apps）。这不否定上述 Win32 API 的桌面应用支持，但意味着 v3/v4、现代 IPP/PSA 及厂商驱动都应进入兼容性验证矩阵，不能预设原厂 UI 和私有 DEVMODE 行为完全一致。[Writing 64-Bit Printer Drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/print/writing-64-bit-printer-drivers)

## 对讨论稿的修正

以下行号对应当前 `.scratch/TALK.md`。

| 讨论稿位置 | 错误或过度简化 | 修正 |
|---|---|---|
| `.scratch/TALK.md:94` | “Level 8 影响该队列的所有任务”过强。 | 应写“设置队列全局默认值”。单次打印可显式携带 DEVMODE，每用户默认值也是独立层，不能把默认值说成对所有任务的强制覆盖。[CreateDCW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-createdcw)、[PRINTER_INFO_9](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-9) |
| `.scratch/TALK.md:94-96` | 权限栏把“管理员 / 普通用户”当成固定 API 规则。 | 应改为队列 ACL 与 `DesiredAccess`。Level 8 走管理权限路径；Level 9 虽为每用户设置，但普通用户最低权限和自定义 ACL 行为仍需实测；单次打印也需要基本打印权限。[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter) |
| `.scratch/TALK.md:103-107` | 标准链一律先 `OpenPrinter(PRINTER_ACCESS_ADMINISTER)` 过粗。 | 读取、原厂 UI、Level 8、Level 9、测试打印应分别确定最小权限；只有需要管理操作的句柄才按管理权限打开。Level 9 不应预设与 Level 8 完全相同的权限策略。[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter) |
| `.scratch/TALK.md:94-98` | 未提及 Level 8 对每用户层的重置语义。 | 微软专题明确说 `SetPrinter(PRINTER_INFO_8)` 会把每用户 DEVMODE 重置为全局默认值；备份和 UI 风险提示不能只覆盖 Level 8，至少要同时保存当前调用用户的 Level 9。[Per-User DEVMODE](https://learn.microsoft.com/en-us/windows/win32/printdocs/per-user-devmode) |
| `.scratch/TALK.md:125` | `EnumPrinters → ... → 附带描述信息` 没有说明版本和架构的来源。 | `EnumPrinters` / `PRINTER_INFO_2` 只给驱动名等队列信息；完整版本和环境需另查 `GetPrinterDriver(Level 6)`，且字段可能缺失。[PRINTER_INFO_2](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-2)、[DRIVER_INFO_6](https://learn.microsoft.com/en-us/windows/win32/printdocs/driver-info-6) |
| `.scratch/TALK.md:117` | “第一版靠它白嫖所有厂商的设置界面”把可用性绝对化。 | API 确实能显示驱动 Print Setup 属性表，但必须处理 `IDCANCEL`、负返回值、阻塞及具体驱动/现代打印栈差异，不能承诺所有厂商均正常。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties) |
| `.scratch/TALK.md:86-88` | 完整保存方向正确，但 `devModeSize` 的含义未固定，“驱动会直接忽略”也略显笼统。 | 文件中的完整长度应明确为实际连续缓冲区的 `dmSize + dmDriverExtra`；`dmFields` 决定哪些公共成员参与合并，并非能力支持清单。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew)、[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties) |
| `.scratch/TALK.md:164-166` | “名称 + 版本 + 架构全一致 ⇒ 完整恢复最可靠”“部分匹配 ⇒ 只恢复标准字段”容易被读成微软兼容性保证。 | 三档可以是 PrintTune 的保守策略，但官方只保证私有段依驱动和版本而变，未保证相同元数据必兼容，也未保证跨版本公共字段值均受支持。完全匹配后仍需驱动合并、同层回读及测试打印；部分匹配需 `DeviceCapabilities` 和驱动验证。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)、[DeviceCapabilities](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-devicecapabilitiesw) |
| `.scratch/TALK.md:108-109,179,221` | “回读 + 打印测试”方向正确，但“明确成功/失败原因”容易把后台提交成功当成物理成功。 | UI 应分开显示“已写入/回读一致或有差异”“作业已提交/作业状态”“用户确认纸面结果”。RAW 直传不消费 DEVMODE，不能作为 DEVMODE 测试。[StartDocW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-startdocw)、[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter) |
| `.scratch/TALK.md:14-16,164` | “完整快照到别的电脑一键导入即可用”是产品愿景，不是 Win32 已证实能力。 | 应保留兼容性拦截、驱动验证、回读、同业务路径测试及人工纸面确认；相同驱动名称也不是充分的兼容性证明。[The DEVMODEW Structure](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/the-devmodew-structure) |

讨论稿中以下关键点则得到官方文档直接支持：`SetPrinter` 支持 Level 8 和 9；`DocumentProperties` 先以 `fMode=0` 探长度；修改后以 `DM_IN_BUFFER | DM_OUT_BUFFER` 合并；`GetPrinter` 需要按所需大小分配缓冲区；DEVMODE 必须包含完整私有段。[SetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/setprinter)、[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)、[GetPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/getprinter)

## 对设计的约束

以下是工程建议，不是微软对所有驱动的兼容性承诺。

1. **把来源层显式建模。** “队列全局默认”和“当前用户默认”必须是不同导出类型；Level 9 配置包还应记录导出时的用户作用域提示，导入时显示目标当前用户，避免服务账户写错配置文件。[PRINTER_INFO_9](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-9)
2. **不要默认整程序提权。** 列表/读取、Level 8 写入、Level 9 写入和测试打印分别处理权限。Level 8 请求管理访问并展示 `ERROR_ACCESS_DENIED`；Level 9 的最小权限通过实机矩阵确认，不能为规避未知问题而一律请求 `PRINTER_ALL_ACCESS`。[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter)
3. **Level 8 应用前同时备份 Level 8 与当前用户 Level 9。** 否则无法完整评估/恢复官方所述的每用户重置副作用；回滚也应分别回读两层。[Per-User DEVMODE](https://learn.microsoft.com/en-us/windows/win32/printdocs/per-user-devmode)
4. **元数据按真实来源采集。** 队列名/端口/驱动名取自 `PRINTER_INFO_2`；驱动版本、日期和环境取自 `DRIVER_INFO_6`；另记 OS 版本、进程架构、DEVMODE A/W 形态。缺失版本字段必须显示“未知”，不能伪造为 0.0.0.0 后参与精确匹配。[PRINTER_INFO_2](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-2)、[DRIVER_INFO_6](https://learn.microsoft.com/en-us/windows/win32/printdocs/driver-info-6)
5. **配置包是不可信输入。** 在交给驱动前，先根据实际文件长度检查公共头是否完整、`dmSize + dmDriverExtra` 是否溢出及是否不超过实际缓冲区，再校验预期长度和 SHA-256。`IsValidDevmode` 不能替代私有段信任与兼容性检查。[IsValidDevmodeW](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/winspool/nf-winspool-isvaliddevmodew)
6. **保存实际输出，不只保存请求输入。** 原厂界面确认后保存 `DM_OUT_BUFFER` 的完整结果；导入经 `DocumentProperties` 合并后，以合并输出调用 `SetPrinter`，再以相同 Level 用 `GetPrinter` 回读。任何差异都要保留诊断信息。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)
7. **回读判定分层。** 至少分别比较目标公共字段、完整长度和私有段摘要；不要把“驱动规范化后不完全逐字节相等”自动等同为 API 失败，也不要仅凭 `SetPrinter == TRUE` 宣布配置生效。哪些差异可接受需按目标驱动实测形成规则。[CreateDCW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-createdcw)
8. **测试打印走与业务一致的配置消费路径。** 若业务通过 GDI 默认设置打印，可用 GDI 测试页；若要验证单次 DEVMODE，测试页也必须显式把该 DEVMODE 传给 `CreateDC`。RAW/ESC/POS 测试只验证其自身指令和通道，不验证 DEVMODE；最终状态需用户确认纸面、切刀和钱箱结果。[CreateDCW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-createdcw)、[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter)
9. **UI 状态不要合并成一个“成功”。** 至少区分：校验通过、写入返回成功、同层回读结果、作业已提交、物理效果已确认。`IDOK` / `IDCANCEL` 和负返回值也需分别处理。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)、[StartDocW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-startdocw)
10. **远程与后台调用单独设计。** 不把远程 Level 9 当作已支持能力；需要远端准确数据时评估 `OpenPrinter2W(PRINTER_OPTION_NO_CACHE)`。打印句柄按线程同步，阻塞 API 不应直接拖住 UI 线程。[MS-RPRN: PRINTER_INFO_9](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rprn/6a5df266-40c9-4661-966f-27caa209b1fd)、[OpenPrinter](https://learn.microsoft.com/en-us/windows/win32/printdocs/openprinter)
11. **兼容矩阵不能只列 x86/x64。** 至少记录 Windows 版本、进程/驱动环境、本地或连接队列、驱动模型与实际业务打印栈；ARM64 或现代 IPP/PSA 是否支持目标流程只能由实机结果回答，本文不作推断。[Writing 64-Bit Printer Drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/print/writing-64-bit-printer-drivers)

## 仍需实机验证

| 优先级 | 验证项 | 需要记录的结果 |
|---|---|---|
| P0 | 标准用户以 `PRINTER_ACCESS_USE` 打开本地队列后执行 Level 9 读写；再测试自定义/收紧 ACL | `OpenPrinter` / `SetPrinter` 返回值和 `GetLastError`；不得只记录是否管理员 |
| P0 | Level 8 在具有管理权限但非本地管理员、以及 `PRINTER_ACCESS_MANAGE_LIMITED`（Windows 8.1+）下的写入 | 最小权限边界、UAC 是否需要、失败码 |
| P0 | Level 8 写入前后当前用户 Level 9 的变化；另一登录用户是否变化 | 验证官方“重置每用户 DEVMODE”在当前环境的准确范围，并验证备份/回滚 |
| P0 | 每种目标驱动的完整导出/导入：探长度、`dmSize`、`dmDriverExtra`、实际返回长度、合并输出、同层回读 | 私有段是否完整保留，驱动是否规范化公共/私有数据，哪些字段发生变化 |
| P0 | 原厂窗口 `DM_IN_BUFFER \| DM_OUT_BUFFER \| DM_IN_PROMPT` | OK/Cancel/失败返回，取消是否污染输出，确认后是否仍需 `SetPrinter` 才能持久化 |
| P0 | GDI 默认 DEVMODE、GDI 显式 DEVMODE、业务实际路径、RAW 路径分别打印测试票 | 作业提交状态与纸张、浓度、切刀、钱箱的人工结果分别记录 |
| P1 | 同名同版本同环境但不同 Windows build；不同驱动版本/架构；公共字段-only 导入 | 不把名称匹配当结果；记录合并、回读和纸面差异，并用 `DeviceCapabilities` 对照 |
| P1 | 本地 USB 队列、网络直连队列、共享连接队列；远程缓存开关 | Level 8/9 支持情况、缓存导致的旧值、`OpenPrinter2W(NO_CACHE)` 的差异 |
| P1 | x86、x64、ARM64 进程与相应驱动环境；统一 W API | `DRIVER_INFO_6` 实际值、完整 DEVMODE 兼容性；ARM64 不预设结果 |
| P1 | v3、v4、厂商驱动及现代 IPP/PSA 组合 | 原厂 UI、私有 DEVMODE、公共字段和测试打印是否遵循同一流程 |
| P2 | 元数据异常与损坏输入 | `pDevMode == NULL`、Level 6 版本/日期为 0、制造商为 `NULL`、截断/超长/长度字段溢出的配置包均被安全拒绝 |

在以上 P0 项完成前，不能把“普通用户 Level 9 一定可写”“Level 8 不影响每用户配置”“相同驱动元数据即可完整恢复”或“测试作业提交即物理成功”写成产品事实。
