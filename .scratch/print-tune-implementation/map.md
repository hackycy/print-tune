# PrintTune V1 完整实现规划

Labels: wayfinder:map

## Destination

形成可直接进入开发的 PrintTune V1 实现规格与交付流程：需求边界、领域模型、技术架构、Windows 打印 API 调用、`.ptcfg` 格式、兼容与安全策略、UI 流程、测试矩阵、打包发布和验收标准均有明确结论；所有阻塞开发的问题都已决策或经技术验证。

## Notes

- 原始依据：[捷印 PrintTune · 方案讨论纪要](../TALK.md)。该文件是讨论留档，不是已验证的技术规格。
- 产品是纯离线 Windows 单机工具，核心是打印首选项 DEVMODE 的完整快照、兼容检查、备份恢复与纸面验证，不是万能驱动配置器。
- 本地 Markdown tracker：开放票据见 `issues/`；`Status: open` 表示未认领，`claimed` 表示处理中，`resolved` 表示已解决。
- 每个实施会话先读本地图，再按票据 `Type` 和 `Blocked by` 选择 frontier；技术事实优先使用微软官方文档和实机验证。
- 本阶段只规划与决策，不实现应用。研究可并行；其他票据每个会话最多解决一个。
- 技术栈为 C# / .NET Framework 4.8 + WPF（保留 Windows 7），x86/x64 分别构建；普通 UI + 短时提权 helper，Level 9 不跨用户写入。

## Decisions so far

- 产品定位（来自讨论纪要）：纯离线 Windows 单机工具，围绕 DEVMODE 快照与恢复，不包含服务端。
- 核心安全原则（来自讨论纪要）：不静默复制不兼容的厂商私有数据；应用前备份，应用后回读并测试打印。
- [核实 Windows 打印配置读写 API 与权限边界](issues/01-verify-win32-configuration-apis.md)：Level 8/9 均可读写；权限取决于队列 ACL；Level 8 写入还需备份当前用户 Level 9；写入、回读、作业提交和纸面成功必须分态。
- [核实兼容分级与仅标准字段降级的安全做法](issues/02-verify-compatibility-and-standard-fallback.md)：三元组一致不是私有兼容保证；完整恢复需精确组合实测；降级必须基于目标 DEVMODE 按允许列表合入并检查目标能力。
- [确定 V1 现场环境与硬件支持矩阵](issues/03-define-v1-support-matrix.md)：产品支持 Win7 SP1/10/11、x86/x64；自动化运行时发现已有虚拟打印机，实体私有功能通过手动实机验收后才进入完整恢复白名单。
- [确定技术栈、进程边界与管理员提权模型](issues/04-choose-stack-and-privilege-model.md)：.NET Framework 4.8 + WPF；x86/x64 分包；普通 UI 与短时 Level 8 helper 分离；Level 9 始终留在原交互用户进程。
- [确定 `.ptcfg` 信任、安全与版本演进策略](issues/05-define-package-trust-and-versioning.md)：严格校验不可信输入；受保护区 SHA-256、展示区可编辑；V1 无签名；高版本与未知必需能力阻断，扩展必须专用且可保留。
- [确定 V1 操作闭环与现场验收标准](issues/06-define-v1-workflow-and-acceptance.md)：Level 8/9 同层恢复；模板只改工作副本；备份后应用，技术失败自动回滚；内置 GDI 测试与人工纸面验收分态。
- [用最小原生验证器确认 P0 实机行为](issues/07-prototype-native-api-validation-matrix.md)：11 项证据与报告契约已原型化；写入必须显式指定队列，完整/标准恢复分别准入，跳过不算通过。
- [定义 `.ptcfg` v1 JSON Schema、测试向量与示例](issues/08-define-ptcfg-v1-schema-and-examples.md)：Schema、JCS/SHA-256 示例和 19 个向量已固定；示例长度与内部 DEVMODEW 头已实测一致。
- [制作 V1 页面流程、状态展示与关键文案原型](issues/09-prototype-v1-ui-and-status-copy.md)：用户选择 A 分步向导；四步主流程、固定分态文案、失败/手动恢复与人工验收交互已确定。

## Not yet specified

- 解决方案与模块边界、依赖注入/日志/异常策略、原生 API 互操作层及可测试性设计。
- 备份文件的保存位置、保留策略、权限、隐私与损坏恢复策略。
- 单元测试、API 假实现、集成测试、真实打印机矩阵、纸面验收和故障注入方案。
- 安装/便携分发、管理员提权、代码签名、版本号、发布产物与现场故障诊断包。
- 从 V1 到“业务打印任务显式携带 DEVMODE”的后续演进边界（单独立项，不属于本次 V1 交付）。

## Out of scope

- 服务端、账号体系、云端下发、自动在线升级。
- 自动下载安装驱动、自动创建 USB/网口端口或配置打印机 IP。
- 跨品牌/不兼容驱动的私有配置转换，以及解析厂商私有二进制偏移。
- 自绘各厂商首选项界面；首版复用驱动原厂界面。
- 打印模板和票据内容设计；它们不属于驱动配置。
