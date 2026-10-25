# 定义 `.ptcfg` v1 JSON Schema、测试向量与示例

Type: prototype
Status: resolved
Blocked by: 05

## Question

如何把已确定的信任与版本策略落成唯一的 `.ptcfg` v1 字段布局、JSON Schema、确定性序列化规则、校验伪代码和正反向测试向量？需要给出完整示例，固定受保护区/可编辑展示区/扩展区、必填与可空元数据、DEVMODEW 长度字段、枚举、大小上限、错误码及编辑往返行为，确保 UI、helper 和未来迁移器使用同一契约。

## Answer

`.ptcfg` v1 契约已落成三份可直接用于开发的文件：

- [ptcfg-v1.schema.json](../contracts/ptcfg-v1.schema.json)：JSON Schema Draft-07，固定字段、必填项、可空元数据、枚举、长度和扩展规则。
- [minimal-valid.ptcfg](../contracts/examples/minimal-valid.ptcfg)：完整示例，包含真实计算得到的 DEVMODEW Base64 和 SHA-256；明确标记为构造数据、不可导入真实设备。
- [ptcfg-v1-test-vectors.json](../contracts/ptcfg-v1-test-vectors.json)：19 个正反向向量、变更操作、目标阶段、错误码和测试大小上限。

### 1. 唯一文件布局

- 根对象仅允许 `formatVersion`、`payload`、`display`、`integrity`，其他字段拒绝。
- `payload` 是唯一受保护区，包含 `packageId`、导出时间、来源层、打印机/驱动身份、DEVMODEW、`requiredFeatures` 和 `extensions`。
- `display` 固定为 `name`、`notes`、`applicableModels`，可编辑但不参与校验和兼容判定。
- `integrity` 固定为 `algorithm: sha256`、`scope: jcs-rfc8785-payload-v1` 和 64 位小写十六进制 `value`。
- 驱动版本、驱动环境和端口允许 `null` 表示未知；关键身份为 `null` 时文件仍可通过结构校验，但完整恢复必须在兼容阶段阻断。
- V1 的 `requiredFeaturesKnown` 为空；任何非空必需特性都被视为当前程序无法理解并拒绝。未知可选扩展纳入校验并在展示编辑往返中保留。

### 2. 确定性校验

采用 **JCS（RFC 8785）对 `/payload` 对象进行规范化，再对其 UTF-8 字节执行 SHA-256**。不直接使用文件原始缩进或键顺序，不包含 `display` 和 `integrity`。

示例的固定测试值：

- 完整 DEVMODEW：224 字节。
- 公共区 `dmSize`：220 字节。
- 私有区 `dmDriverExtra`：4 字节。
- SHA-256：`1985e8311a853edc63d682c446f0fca3bb37bce50fc341ddba7c0585cebe2fd2`。

JCS 实现必须拒绝重复键、无效 UTF-8、未配对代理项、NaN/Infinity 和含糊数字。V1 数值只使用十进制整数，避免不同运行时的浮点序列化差异。所有实现共享示例哈希和测试向量，不各自发明“近似 JCS”。

### 3. 导入伪代码

```text
read file with 4 MiB limit
→ strict UTF-8 + JSON parse with duplicate-key detection and depth <= 16
→ require formatVersion == 1 before interpreting the rest
→ validate entire document against ptcfg-v1.schema.json
→ canonicalize payload using JCS/RFC 8785
→ SHA256(canonicalUtf8) and constant-time compare with integrity.value
→ strict Base64 decode devMode.data
→ use wide arithmetic to require:
     decodedLength == totalSize == publicSize + driverExtraSize
     decodedLength <= 1 MiB
→ read dmSize/dmDriverExtra from the decoded DEVMODEW header
   and require them to equal publicSize/driverExtraSize
→ reject unknown requiredFeatures
→ only then perform source/target compatibility checks
→ only then call the target driver
```

任何前序失败都停止，不向 UI 提供“强制继续”，也不把未校验数据送入 helper。Schema 的正则只作为第一道 Base64 过滤，仍须由运行库严格解码并检查实际字节长度。

### 4. 大小与结构边界

- 文件最大 4 MiB；JSON 深度最大 16；解码 DEVMODE 最大 1 MiB。
- Schema 对名称、备注、适用说明、驱动元数据、扩展数量、必需特性数量和 Base64 长度另设上限。
- `totalSize`、`publicSize`、`driverExtraSize` 均使用宽整数参与求和，禁止 16/32 位溢出。
- 文件声明的三个长度、实际解码长度和 DEVMODEW 内部两个长度必须一致；不能只信任其中一组。
- V1 只接受 `DEVMODEW` 和 `x86/x64` 进程架构元数据；其他架构需要未来格式/支持决策，不能伪造为 x64。

### 5. 展示编辑与往返

1. 完整验证原文件。
2. 只修改 `display`，保持 `payload` 和 `integrity` 不变，包括未知可选扩展。
3. 重新序列化整个根对象，不要求保持原缩进或键顺序。
4. 再次完整验证；展示编辑无需重算受保护区哈希。
5. 受保护内容不允许通过展示编辑修改。重新导出或未来显式迁移生成新的包与新校验；当前包的 `packageId` 不因纯展示编辑改变。

### 6. 固定错误阶段与代码

阶段为 `file → json → version → schema → integrity → base64 → devmode → features → compatibility`。代码包括 `PTCFG_FILE_TOO_LARGE`、`PTCFG_JSON_INVALID`、`PTCFG_DUPLICATE_KEY`、`PTCFG_VERSION_UNSUPPORTED`、`PTCFG_SCHEMA_INVALID`、`PTCFG_CHECKSUM_MISMATCH`、`PTCFG_BASE64_INVALID`、`PTCFG_DEVMODE_LENGTH_INVALID`、`PTCFG_DEVMODE_STRUCTURE_INVALID`、`PTCFG_REQUIRED_FEATURE_UNKNOWN`、`PTCFG_IDENTITY_MISSING`。

UI 可将代码翻译为友好中文，但日志/测试保留稳定代码和阶段。兼容失败与文件无效是不同类别，不能都显示“文件损坏”。

### 7. 已执行验证与边界

已实际验证三份 JSON 均可解析；示例 SHA-256 与测试套件固定值一致；Base64 严格解码为 224 字节；声明长度与实际长度一致；内部偏移读取得到 `dmSize=220`、`dmDriverExtra=4`；19 个向量 ID 唯一且引用的错误码均已声明。

本票据交付的是契约和测试向量，不表示已经在 .NET Framework 4.8 中选定 JSON Schema/JCS 库，也不表示 19 个向量已经全部由产品代码执行。库选型、重复键解析和跨运行时 JCS 一致性属于后续解决方案架构与实现测试。
