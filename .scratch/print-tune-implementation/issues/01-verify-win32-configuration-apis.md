# 核实 Windows 打印配置读写 API 与权限边界

Type: research
Status: resolved
Blocked by: none

## Question

基于微软官方文档，核实 PrintTune 的打印机枚举、驱动元数据、完整 DEVMODE 获取、原厂首选项窗口、驱动校验合并、PRINTER_INFO_8/9 读写、所需权限、回读验证与测试打印调用链。重点确认 `SetPrinter` 对 Level 8 和 Level 9 的实际支持、每用户与队列默认值语义、32/64 位及 Windows 版本注意事项，并指出讨论纪要中任何错误或过度简化。结论须能直接约束后续架构和 UI，不做应用实现。

## Answer

研究结果：[win32-configuration-apis.md](../research/win32-configuration-apis.md)。

1. `GetPrinter` / `SetPrinter` 的官方 Level 列表均包含 8 和 9；Level 9 的每用户写入在本地 Win32 API 中有明确依据，但远程协议规范明确不使用 `PRINTER_INFO_9`，远程场景须另验。
2. Level 8 是队列全局默认值，不是强制该队列所有任务使用相同设置；微软还说明 `SetPrinter(PRINTER_INFO_8)` 会把每用户 DEVMODE 重置为全局默认值，因此应用前须同时备份当前用户 Level 9。
3. `OpenPrinter` 权限由 `DesiredAccess` 与队列 ACL 决定，不能简单等同“本地管理员 / 普通用户”。Level 8 走管理权限路径；Level 9 对标准用户的最低权限没有通用官方保证，须实机验证。
4. `EnumPrinters` Level 2 提供队列名、端口和驱动名等，但不提供完整驱动版本/环境；应另查 `GetPrinterDriver(Level 6)` / `DRIVER_INFO_6`，并处理版本为 0、字段为 `NULL` 及潜在本地驱动安装副作用。
5. `DocumentProperties` 的正确流程是 `fMode=0` 探长度、分配完整缓冲区、`DM_OUT_BUFFER` 读取，修改公共字段和 `dmFields` 后以 `DM_IN_BUFFER | DM_OUT_BUFFER` 合并；原厂窗口返回 `IDOK` 才是确认，`IDCANCEL` 和负值不得当成功。
6. 完整 DEVMODE 是连续缓冲区的 `dmSize + dmDriverExtra`，必须深复制并保留完整私有段；`IsValidDevmode` 只验证公共成员，配置包仍须先做长度/溢出检查，不能据此信任私有数据。
7. “同驱动名、版本、架构即可完整恢复”不是微软兼容性保证；即使元数据一致，仍需驱动合并、同层回读和测试打印，部分匹配恢复公共字段也需 `DeviceCapabilities` 验证。
8. `SetPrinter` 成功、回读结果、作业提交成功和物理打印成功是不同状态；驱动可修正或拒绝消费 DEVMODE，UI 不得只显示一个笼统“成功”。
9. 可用 GDI `CreateDC` 显式传入合并后的 DEVMODE 并打印测试页；RAW 路径不使用 `PRINTER_DEFAULTS.pDevMode`，不能用于证明 DEVMODE 导入生效，最终纸面、切刀和钱箱结果须人工确认。
