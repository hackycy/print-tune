# DEVMODE 兼容分级与仅标准字段降级研究

- 日期：2026-10-24
- 范围：Windows Win32 打印 API；仅研究，不涉及应用实现。
- 方法：以 Microsoft Learn / Win32 / Windows 驱动官方文档为依据。下文将**官方事实**与**工程建议**分开；未把 API 调用成功解释为打印效果或私有格式兼容保证。

## 结论摘要

1. **驱动名称 + 驱动版本 + 架构相同，不足以从官方文档推出私有 `dmDriverExtra` 二进制兼容。**官方只说明私有区的内容和大小随驱动及驱动版本变化，并未给出上述三元组的兼容保证。完全匹配只能作为“完整恢复候选”条件，还应要求该驱动包/设备组合已被纳入实测白名单。
2. **跨版本或跨架构降级时，不复制任何来源私有字节，也不沿用来源的 `dmSize`、`dmDriverExtra` 或私有区长度。**由目标驱动生成完整、有效的基准 DEVMODE，在基准上按字段语义合入少量允许的标准字段。
3. 官方推荐的修改链路是：`DocumentProperties(fMode=0)` 查询完整长度 → `DM_OUT_BUFFER` 取得当前设置（含私有数据）→ 修改公共字段和 `dmFields` → `DM_IN_BUFFER | DM_OUT_BUFFER` 让目标驱动合并并返回结果。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)
4. “标准字段”不等于“可盲目复制”。纸张编号、表单名、分辨率、份数、纸盒和介质类型均有能力或设备相关性；其中自定义纸张编号、纸盒和介质值可以具有设备/驱动特定含义。[DEVMODEA](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodea)
5. `DocumentProperties` 返回 `IDOK`、合并后字段一致、写回后再次读取一致，是三个不同层次的技术检查；它们都不能单独证明浓度、切刀、钱箱等私有设置或实际纸面效果正确。

---

## 已证实事实

### 1. DEVMODE 的公共区、私有区和长度边界

1. `dmSize` 是公共 `DEVMODEW` 部分的字节数，**不含**私有驱动数据；`dmDriverExtra` 是紧跟公共区之后的私有驱动数据字节数。私有区起点为 `缓冲区起点 + dmSize`，完整可读布局长度为 `dmSize + dmDriverExtra`。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew)
2. `dmDriverExtra` 为 0 表示没有私有数据；私有成员由驱动定义并供该驱动专用。应用不能用固定的 `sizeof(DEVMODE)` 代替完整缓冲区长度。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew)
3. `dmFields` 的位标识哪些成员正在使用、含有有效数据。例如设置 `DM_ORIENTATION` 表示 `dmOrientation` 有效；因此不能只读取字段值而忽略其来源位。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew)
4. 打印驱动实际使用的 DEVMODE 由设备无关部分和驱动特定部分组成，后者的大小和内容会随**每个驱动及驱动版本**变化。微软因此要求应用在分配缓冲区前向驱动查询正确大小。[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)
5. Windows 在相关路径上只确认 DEVMODE 公共部分有效；损坏的私有部分仍可能使应用进程或 spooler 中的驱动代码崩溃。`dmSize + dmDriverExtra` 的边界可读不等于私有内容有效。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew)
6. 微软的修改指南明确说明：有效的设备 DEVMODE 含有私有数据，私有数据只能可靠地通过 `DocumentProperties()` 修改；缺少私有数据的 DEVMODE 传给 `CreateDC()`、`ResetDC()` 或 `DocumentProperties()` 可能失败。[Modify printer settings with DocumentProperties](https://learn.microsoft.com/en-us/troubleshoot/windows/win32/modify-printer-settings-documentproperties)
7. `DEVMODEW` 是 Unicode 版本，应用可使用 ANSI 或 Unicode API，但驱动必须使用 Unicode 版本。配置包若保存原始字节，必须明确其 A/W 编码和实际缓冲区长度，不能把不同表示当作同一种二进制格式。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew)

### 2. 官方推荐的目标驱动基准、合并与返回结果

微软给出的本地修改打印设置顺序如下：[DocumentProperties](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)

1. 以 `fMode=0` 调用 `DocumentProperties`，取得**完整** DEVMODE 所需字节数；该值可能大于公共结构体。
2. 分配该长度的缓冲区，以 `DM_OUT_BUFFER` 调用，取得当前打印设置；输出包含私有数据。
3. 修改返回缓冲区中适当的公共成员，并设置对应的 `dmFields` 位。
4. 将修改后的缓冲区作为输入和输出，以 `DM_IN_BUFFER | DM_OUT_BUFFER` 再次调用。`DM_IN_BUFFER` 的语义是把驱动当前设置与输入合并，且只更新 `dmFields` 指定的成员；冲突时输入值覆盖当前值。该步骤给驱动更新私有部分的机会。
5. 无属性页时，成功返回 `IDOK`，失败返回小于 0；`fMode=0` 时返回所需缓冲区大小。应逐步检查返回值，不能把长度查询失败当成长度。

补充事实：

- `dmFields` 位只是字段使用/支持指示，并不证明设备支持该字段的所有取值。微软要求用 `DeviceCapabilities()` 判断字段支持的设置。[Modify printer settings with DocumentProperties](https://learn.microsoft.com/en-us/troubleshoot/windows/win32/modify-printer-settings-documentproperties)
- `DeviceCapabilities` 可查询支持的纸张编号（`DC_PAPERS`）、纸张名称（`DC_PAPERNAMES`）、纸张尺寸（`DC_PAPERSIZE`）、分辨率对（`DC_ENUMRESOLUTIONS`）、份数（`DC_COPIES`）、纸盒（`DC_BINS`）、双面能力（`DC_DUPLEX`）以及驱动 `dmFields`（`DC_FIELDS`）。[DeviceCapabilities](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-devicecapabilitiesa)
- 修改设备无关字段也可能影响设备相关部分，因此在 `SetPrinter()` 前应调用 `DocumentProperties()` 形成一致 DEVMODE。[Modify printer settings with SetPrinter](https://learn.microsoft.com/en-us/previous-versions/troubleshoot/windows/win32/modify-printer-settings-setprinter-api)
- `PRINTER_INFO_8` 是全局默认 DEVMODE，`PRINTER_INFO_9` 是保存在用户注册表中的每用户默认 DEVMODE；二者作用域不同。[PRINTER_INFO_8](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-8)、[PRINTER_INFO_9](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-9)

**边界说明：**官方文档没有承诺 `IDOK` 表示所有请求值均被原样接受，也没有承诺它证明私有设置的物理效果正确。它证明的是该次 API 操作成功；请求值是否被规范化仍须检查输出，持久化结果仍须重读，纸面效果仍须测试。

### 3. 标准字段的可移植性边界

| 字段 | 官方事实 | 对跨版本/架构复制的直接含义 |
|---|---|---|
| `dmOrientation` | 取 `DMORIENT_PORTRAIT` / `DMORIENT_LANDSCAPE`，但仍须结合字段支持情况。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew) | 是相对简单的候选字段，不是无条件生效。 |
| `dmPaperSize` | 使用显式 `dmPaperLength` 和 `dmPaperWidth` 时应为 0；此外可使用预定义值，DEVMODEA 文档还明确允许 `>= DMPAPER_USER` 的**设备特定值**。[DEVMODEA](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodea) | 预定义编号也应查询目标 `DC_PAPERS`；设备特定编号不能假定跨驱动同义。 |
| `dmPaperLength` / `dmPaperWidth` | 单位为 0.1 mm，覆盖 `dmPaperSize` 对应尺寸；使用长度时必须使用宽度。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew) | 必须成组复制并处理 `dmPaperSize=0` 及相应位，不能只复制一维。物理尺寸相同也不证明标签/票据走纸效果相同。 |
| `dmFormName` | 名称必须能通过 `EnumForms` 获得；`EnumForms` 枚举指定打印机支持的表单。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew)、[EnumForms](https://learn.microsoft.com/en-us/windows/win32/printdocs/enumforms) | 来源表单名在目标上可能不存在；不能只复制字符串。 |
| 自定义表单定义 | `FORM_INFO_1` 除名称外还有 `Size` 和 `ImageableArea`；`FORM_USER` 表单定义在注册表中，`FORM_PRINTER` 与特定打印机关联。[FORM_INFO_1](https://learn.microsoft.com/en-us/windows/win32/printdocs/form-info-1) | DEVMODE 中的表单名不是完整表单定义。同名也不能替代目标几何尺寸/可打印区域核对。 |
| `dmPrintQuality` / `dmYResolution` | 负值是 `DMRES_*` 质量级别；正值时 `dmPrintQuality` 为 x DPI，`dmYResolution` 为 y DPI，二者关联。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew) | 正 DPI 应按 `(x,y)` 对查询目标 `DC_ENUMRESOLUTIONS`，不可只复制 x 或假定相同 DPI 均受支持。 |
| `dmCopies` | 在设备支持多份打印时指定份数；`DC_COPIES` 返回设备可打印份数。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew)、[DeviceCapabilities](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-devicecapabilitiesa) | 应检查目标范围；不能假定所有驱动都接受来源数值。 |
| `dmDefaultSource` | 可为 `DMBIN_*`，也可为 `>= DMBIN_USER` 的设备特定值；目标纸盒可由 `DC_BINS` 查询。[DEVMODEA](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodea) | 设备特定纸盒编号不能跨驱动直接复制。 |
| `dmMediaType` | 可为预定义值，也可为 `>= DMMEDIA_USER` 的驱动定义值。[DEVMODEA](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodea) | 驱动定义值不能假定跨版本/架构同义。 |
| `dmDriverVersion` | 打印驱动版本号由驱动开发者分配。[DEVMODEW](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew) | 它不是私有数据格式版本或兼容承诺。 |

### 4. 驱动身份元数据能证明什么

`DRIVER_INFO_2` 提供驱动名称 `pName`、环境 `pEnvironment`（如 Windows x86/x64）、驱动/数据/配置文件路径等；其中 `cVersion` 是**驱动所面向的操作系统版本**，文档列出的支持值为 3，并非厂商文件版本号。[DRIVER_INFO_2](https://learn.microsoft.com/en-us/windows/win32/printdocs/driver-info-2)

因此：

- 名称、厂商文件版本、环境/架构可用于识别和筛选配置包。
- `DRIVER_INFO_2.cVersion` 不能被误当作讨论稿示例中的 `1.0.0.5` 一类厂商驱动文件版本。
- 官方资料没有定义“名称 + 版本 + 架构一致”作为 `dmDriverExtra` 的兼容契约，也没有定义“版本不同但架构相同”时的私有区转换规则。
- OEM 插件可通过 `IPrintOemUI::DevMode` 实现私有成员的取大小、初始化、转换或验证，但这是驱动/插件自身能力，不是通用应用可假定的跨版本兼容保证。[Providing DEVMODE Structure Additions](https://learn.microsoft.com/en-us/windows-hardware/drivers/print/providing-devmode-structure-additions)

---

## 推荐判定与合并规则

以下均为**工程建议**，不是微软官方兼容保证。

### 1. 先验证配置包边界，再讨论兼容性

1. 将配置包视为不可信输入：记录并核对 A/W 表示、实际总字节数、`dmSize`、`dmDriverExtra`；以足够宽的整数计算 `dmSize + dmDriverExtra`，确保不小于所需公共字段边界且不超过实际缓冲区。
2. 校验来源 `dmFields` 与待取字段是否存在；未置位的字段值不作为迁移意图。
3. 校验文件完整性、来源和身份元数据。普通 checksum 只能发现损坏/变化，不能证明发布者可信。
4. 长度异常、字段越界、字符串未正确终止、身份缺失或无法解析时直接阻断；不要把异常缓冲区交给驱动“试着验证”。

### 2. 三档判定规则

| 档位 | 必要条件 | 允许动作 | 不得宣称 |
|---|---|---|---|
| **完全匹配且已验证** | 配置包可信且结构有效；目标设备/驱动组合在明确支持矩阵内；规范化驱动名、真实厂商驱动文件版本、驱动环境/架构一致；必要时核对驱动包/关键文件身份；`dmSpecVersion`、`dmDriverVersion` 和公共/私有长度作为附加结构检查；该精确组合的完整私有恢复已通过实机验证 | 可将来源完整 DEVMODE 交给**目标驱动**的 `DocumentProperties` 合并流程，使用目标查询长度分配输出；随后检查返回、写回、重读并打印测试 | 不得仅凭三元组宣称私有区必然兼容；不得把 `IDOK` 当成纸面验收 |
| **受限标准字段降级** | 来源与目标属于明确配置的“标准字段可迁移”组合；所有业务必需项都能由允许的标准字段表达；不依赖浓度、切刀、钱箱等未恢复私有项 | 仅在下文允许列表内逐字段复制到目标基准；不复制来源任何私有字节；能力检查、合并、输出核对、重读、必要时测试 | 不得称为完整恢复；若关键私有项未恢复，不得报告配置全部成功 |
| **阻断** | 未知/未批准组合；不同品牌或无明确迁移关系；来源无效；关键字段不支持；自定义编号无法映射；必需表单缺失或定义不一致；关键业务项依赖私有段但不满足完整恢复条件；合并/写回/重读失败 | 不修改目标设置，保留原配置并给出可操作原因 | 不提供静默“尽力恢复”，也不以警告替代必需项失败 |

补充规则：

- **版本或架构不同：默认不得完整复制私有段。**只有进入已批准的标准字段迁移组合时才可降级；否则阻断。
- **名称、版本、架构相同但未实测：仍不自动获得完整恢复资格。**可按产品策略选择标准字段降级或阻断，不能标为“最可靠且已保证”。
- “型号接近”不是可执行判据。支持矩阵应由明确的来源/目标驱动包和设备组合构成，不使用模糊字符串相似度。
- `dmSize` 或 `dmDriverExtra` 相同只说明长度相同，不证明私有布局相同；不同则更不能据此做私有二进制复制。

### 3. 仅标准字段降级的推荐调用与合并顺序

1. 打开**目标**打印机句柄，以其设备名调用 `DocumentProperties(..., NULL, NULL, 0)`；检查返回长度并分配完整缓冲区。
2. 以 `DM_OUT_BUFFER` 获取目标当前 DEVMODE，作为唯一基准。保留目标生成的公共头、`dmSize`、`dmDriverExtra`、目标私有区及目标设备名。
3. 从来源仅读取业务明确请求、来源 `dmFields` 置位且属于允许列表的字段。禁止整块 `memcpy` 公共区，禁止把来源 `dmFields` 整体覆盖到目标。
4. 用目标 `DeviceCapabilities`、`EnumForms` 等核对字段和值；不支持的必需值直接阻断，不让驱动静默改成别的值。
5. 将本次输入的 `dmFields` 限定为实际提交且已校验的字段位；关联字段必须成组设置。例如显式纸张尺寸同时处理 `DM_PAPERLENGTH | DM_PAPERWIDTH | DM_PAPERSIZE`，正 DPI 同时处理 x/y 分辨率。
6. 调用目标 `DocumentProperties`，传入修改后的基准和 `DM_IN_BUFFER | DM_OUT_BUFFER`，取得驱动合并后的完整输出。输入、输出可使用文档示例中的同一缓冲区；若需逐项比较请求值与返回值，使用独立输出缓冲区更清晰。成功要求返回 `IDOK`，并再次检查输出长度边界。
7. 对请求的标准字段做语义比较。若目标驱动拒绝、规范化或改回非预期值，应判定未达到请求，不得只凭 `IDOK` 继续报告成功。私有区不做逐字节相等判断，因为驱动可合法地规范化其内容，而应用无法理解其语义。
8. 持久化到队列或用户默认值时，明确选择 Level 8 或 9；`SetPrinter` 后按同一作用域重新获取 DEVMODE，核对允许字段。写回成功不等于另一作用域也已改变。
9. 对影响实际纸张、走纸、浓度、切刀、钱箱等的配置执行测试票。任何必需项无法验证时，状态应为“需人工复核”或“失败”，而非“成功”。

### 4. 默认字段允许列表

“允许”仍以前述目标能力检查、成组设置、合并后核对和实机验证为前提。

| 字段组 | 默认策略 |
|---|---|
| `dmOrientation` | 可作为最小允许列表项；仅复制预定义方向值。 |
| `dmCopies` | 条件允许；检查正值及目标 `DC_COPIES` 范围。 |
| `dmColor`、`dmDuplex` | 条件允许；仅使用预定义值并检查目标能力。双面先用 `DC_DUPLEX` 等确认。 |
| `dmPrintQuality` + `dmYResolution` | 条件允许；正 DPI 必须匹配目标 `DC_ENUMRESOLUTIONS` 的 `(x,y)` 对。负 `DMRES_*` 虽为标准常量，也须核对合并结果并实测实际质量。 |
| 预定义 `dmPaperSize` | 条件允许；目标 `DC_PAPERS` 必须支持。`>= DMPAPER_USER` 的设备特定值默认阻断，除非存在逐驱动、经实测的显式映射。 |
| `dmPaperLength` + `dmPaperWidth`（及 `dmPaperSize=0`） | 高风险、条件允许；必须成组，检查目标合并结果并测试走纸/打印区域。 |
| `dmFormName` | 高风险、条件允许；目标 `EnumForms` 必须存在该名称，并比较 `FORM_INFO_1.Size` 与 `ImageableArea`。自定义表单未随 DEVMODE 自动迁移，不静默创建或替换。 |
| `dmDefaultSource`、`dmMediaType` | 默认不纳入跨驱动降级；设备/驱动自定义值阻断。即便预定义值也应查目标能力。 |
| `dmScale`、`dmCollate`、`dmTTOption`、ICM、`dmNup` 等 | 不纳入首版通用允许列表；只有明确业务需求、目标能力检查和对应驱动实机验证后才逐项加入。 |
| 显示器专用字段、保留字段、`dmDeviceName`、`dmSpecVersion`、`dmDriverVersion`、`dmSize`、`dmDriverExtra` | 不作为迁移值复制；保留目标基准。 |
| 来源 `dmDriverExtra` 指向的全部私有字节 | 在标准字段降级中一律不复制、不拼接、不清零后复用来源缓冲区。 |

如果来源同时存在多个纸张相关位，应按来源实际有效位识别其表示并验证整个组合，不能只取一个字段，也不能留下互相冲突的旧位。

### 5. 建议的分层成功标准

1. **结构成功**：来源边界有效，目标输出缓冲区长度有效。
2. **驱动调用成功**：每次 `DocumentProperties` 返回符合预期。
3. **语义接受**：合并后目标仍保留请求的标准字段值；未发生不可接受的改值。
4. **持久化成功**：`SetPrinter` 成功，并在相同 Level 8/9 作用域重读一致。
5. **业务验收成功**：测试票证实纸张、分辨率、份数及所需私有功能正确。

产品状态不得把前四层中的任何一层单独显示成第五层。

---

## 对讨论稿的修正

| 讨论稿位置/说法 | 建议修正 |
|---|---|
| 4.2：“Windows 标准字段 ← 可按字段名安全修改” | 改为“可按明确允许列表逐字段修改，并须检查来源 `dmFields`、目标能力和合并后结果”。标准字段并不都可移植。 |
| 4.2：“`dmFields` 必须同步置位” | 补充：不能整体照搬来源位；只提交本次允许且已校验的字段位，关联字段成组处理。位标志也不能代替 `DeviceCapabilities`。 |
| 4.4：“DocumentProperties 负责校验合法性、修正不支持的值、合并私有设置” | 改为“给目标驱动合并公共修改并更新私有部分的机会”。`IDOK` 不是每个值均被原样接受或物理效果正确的保证，必须检查输出、重读并测试。 |
| 4.4 调用链 | 保留“三次 DocumentProperties”主线；增加每步返回检查、完整缓冲区边界检查，以及 SetPrinter 后按 Level 8/9 同作用域重读。 |
| 6：配置包只列 `devModeSize` | 明确它是实际总缓冲区长度，同时记录/校验 `dmSize`、`dmDriverExtra` 和 A/W 表示；长度相等不构成私有兼容证明。 |
| 7：“驱动名称 + 驱动版本 + 系统架构全一致 → 恢复完整 DEVMODE，最可靠” | 改为“完全匹配且该精确组合已实测 → 才允许完整恢复候选”。三元组本身不是官方兼容保证；厂商文件版本也不能与 `DRIVER_INFO_2.cVersion` 混淆。 |
| 7：“型号接近但驱动版本/架构不同 → 只恢复标准字段” | 删除“型号接近”模糊规则。仅允许显式配置且经测试的标准字段迁移组合；不同架构/版本绝不复制私有段。 |
| 7：部分匹配包含“纸张、方向、份数、分辨率” | 将纸张、份数、分辨率改为条件项；自定义纸张编号、表单、正 DPI 和份数都有目标依赖，不能统一盲目复制。 |
| 7：部分匹配后仅“提示人工复核浓度、切刀” | 若浓度、切刀、钱箱等是业务必需项，私有段未恢复应判为“不完整/需人工配置”或阻断，不能显示为已成功导入。 |
| 9：“系统自定义纸张表单不一定导出” | 强化为“表单名不是表单定义”；至少需核对目标 `EnumForms`、`Size`、`ImageableArea`，自定义表单迁移应作为独立功能/流程。 |

---

## 必须实机验证的假设

以下均未由上述官方文档证明，必须在宣称支持前验证：

1. **完整私有恢复假设**：同一规范化驱动名、同一真实文件版本、同一架构下，来源私有段能被目标驱动正确解释。每个“设备型号 + 驱动包/文件版本 + 架构 + 相关 OEM 插件”组合都需验证；驱动升级后重新验证。
2. **长度与私有格式假设**：即使 `dmSize`、`dmDriverExtra`、`dmDriverVersion` 相同，也不能据此省略实测；长度相同不证明布局相同。
3. **DocumentProperties 规范化假设**：目标驱动是否真正接受请求值、是否修改相关私有数据、是否会在无错误返回时改回默认值，只能通过输出比较、重读和打印确认。
4. **跨版本/跨架构标准字段假设**：允许列表中的方向、份数、颜色、双面、分辨率和纸张在每个声明支持的迁移组合上均有效，不发生不可接受的静默映射。
5. **纸张与自定义表单假设**：同名表单的尺寸/可打印区域一致，显式 0.1 mm 尺寸能被目标正确解释，目标不存在额外表单或机身设置依赖；自定义编号不做无实测映射。
6. **实际硬件功能假设**：纸张尺寸、黑标/间隙、浓度、速度、切刀位置、钱箱、份数和分辨率在真实纸面上正确。DEVMODE API 成功无法替代这一层。
7. **作用域假设**：队列默认值、当前用户默认值和业务打印实际采用的 DEVMODE 一致；若业务显式携带另一 DEVMODE，写回 Level 8/9 可能不影响最终任务，必须在真实打印链路验证。
8. **环境组合假设**：制造商驱动与类驱动、不同驱动模型/OEM UI 插件、必要的操作系统版本组合均覆盖在测试矩阵内；未覆盖组合不得自动标为完整兼容。

最小验收应同时覆盖：合并返回检查、请求值与输出值比较、SetPrinter 后同层重读、应用重启/驱动界面复核（用于观察私有设置，如可行）、测试票和恢复备份。任一业务必需项未通过，都不能以“已导入但请复核”代替失败状态。

---

## 官方来源索引

1. [DEVMODEW structure (wingdi.h)](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodew)
2. [DEVMODEA structure (wingdi.h)](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-devmodea)
3. [DocumentProperties function](https://learn.microsoft.com/en-us/windows/win32/printdocs/documentproperties)
4. [Modify printer settings with the DocumentProperties() Function](https://learn.microsoft.com/en-us/troubleshoot/windows/win32/modify-printer-settings-documentproperties)
5. [DeviceCapabilities function](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-devicecapabilitiesa)
6. [EnumForms function](https://learn.microsoft.com/en-us/windows/win32/printdocs/enumforms)
7. [FORM_INFO_1 structure](https://learn.microsoft.com/en-us/windows/win32/printdocs/form-info-1)
8. [DRIVER_INFO_2 structure](https://learn.microsoft.com/en-us/windows/win32/printdocs/driver-info-2)
9. [Providing DEVMODE Structure Additions](https://learn.microsoft.com/en-us/windows-hardware/drivers/print/providing-devmode-structure-additions)
10. [Modify printer settings by using the SetPrinter function](https://learn.microsoft.com/en-us/previous-versions/troubleshoot/windows/win32/modify-printer-settings-setprinter-api)
11. [PRINTER_INFO_8 structure](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-8)
12. [PRINTER_INFO_9 structure](https://learn.microsoft.com/en-us/windows/win32/printdocs/printer-info-9)
