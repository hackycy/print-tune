# 选定 .NET Framework 4.8 的 JSON Schema 与 JCS 实现

Type: research
Status: claimed
Blocked by: 08

## Question

基于项目官方文档、源码与许可证，选择可在 Windows 7 SP1 / 10 / 11 的 .NET Framework 4.8 离线应用中使用的 JSON 解析、重复键拒绝、JSON Schema Draft-07 校验和 JCS/RFC 8785 规范化方案。比较候选库的 net48 支持、维护状态、许可证、依赖大小、严格 UTF-8/数字/重复键行为与测试能力；判断 JCS 应使用成熟库还是为受限 payload 实现带 RFC/官方测试向量的专用规范化器，并给出明确选型及验证要求，不编写产品实现。
