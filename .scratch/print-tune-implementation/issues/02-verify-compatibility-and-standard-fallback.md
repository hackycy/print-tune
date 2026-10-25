# 核实兼容分级与仅标准字段降级的安全做法

Type: research
Status: resolved
Blocked by: none

## Question

基于微软官方文档，研究如何判定 DEVMODE 的来源与目标兼容性，以及不同驱动版本/架构下“仅恢复标准字段、不复制 dmDriverExtra”的安全实现方式。明确能否仅依靠驱动名称、版本和架构分三档；如何通过目标驱动获取有效 DEVMODE、合入标准字段与 dmFields、调用 DocumentProperties 校验并回读；哪些字段不能盲目复制（纸张编号、表单、私有长度等）；哪些情况必须要求实机原型验证。输出推荐判定规则及不可证实的假设，不做应用实现。

## Answer

研究结果：[compatibility-and-standard-fallback.md](../research/compatibility-and-standard-fallback.md)。

1. **驱动名称 + 真实驱动文件版本 + 架构一致，不足以证明 `dmDriverExtra` 私有格式兼容。**完全匹配只能成为完整恢复候选，还必须位于经过实机验证的精确设备/驱动组合支持矩阵中。
2. `dmSize` 仅表示公共区长度，私有区从 `dmSize` 偏移开始、长度为 `dmDriverExtra`；完整缓冲区边界是二者之和。长度相同不证明布局相同，私有内容损坏也不能依赖 Windows 的公共区校验发现。
3. 标准字段降级必须由目标驱动以 `DocumentProperties(fMode=0)` 查询完整长度、以 `DM_OUT_BUFFER` 生成当前有效基准，再仅合入允许字段及对应 `dmFields`，最后以 `DM_IN_BUFFER | DM_OUT_BUFFER` 合并并取回结果。
4. 降级时保留目标驱动的头部、长度和私有区，**不复制来源任何私有字节，不整块复制公共区，也不整体照搬来源 `dmFields`**。版本或架构不同默认不得完整恢复私有段。
5. 标准字段不是普遍安全白名单：方向可作为最小候选项；份数、预定义纸张、颜色、双面和分辨率须经目标能力查询；正 DPI 按 `(x,y)` 对检查；设备/驱动自定义纸张、纸盒、介质编号默认阻断。
6. `dmPaperLength`/`dmPaperWidth` 必须成组处理；`dmFormName` 必须在目标 `EnumForms` 中存在，并核对 `FORM_INFO_1.Size` 与 `ImageableArea`。表单名不是完整自定义表单定义。
7. `DocumentProperties` 返回 `IDOK` 不代表所有值均被原样接受，更不代表纸面效果正确；还须比较合并输出，`SetPrinter` 后按相同 Level 8/9 作用域重读，并执行测试票。
8. 三档应定义为“完全匹配且已验证 / 受限标准字段降级 / 阻断”，不使用“型号接近”模糊判定。若浓度、切刀、钱箱等业务必需项依赖未恢复的私有段，应判为不完整或阻断，不能宣称导入成功。
