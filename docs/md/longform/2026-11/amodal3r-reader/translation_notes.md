# Amodal3R Nature Reader 提取说明

- Source format: `pdf-text`; 原 PDF 共 22 页，正文和补充材料均有可提取文本层。
- p.1–8 页面资产已渲染并链接；p.9–13 为参考文献，p.14–22 为补充材料，关键方法、设置、限制已在 source map 中保留页码。
- E001–E005 根据 p.4–5 页面核对；文本层对上下标、连字符和字距有轻微噪声，公式转写标为 medium confidence。
- 真实场景定性图没有完整 3D ground truth；读本不把视觉合理性写成隐藏表面真值。
- 未运行官方代码、未下载权重、未启动 GPU；所有迁移方案均为研究假设。
