## 精读｜CompMVR：压缩不是画质问题，而是跨视角几何证据被破坏

原始出处：[arXiv:2610.32800](https://arxiv.org/abs/2610.32800)，首次公开 **2026 年 10 月**；正式信息标为 **SIGGRAPH Asia 2026 Conference Papers**。阅读版本 v1。PDF 已下载并导入 Zotero `每日精读论文/2026/10`，条目 `F81MIUGM`、附件 `VZTQC6ZF`，下载与 storage SHA256 均为 `4be2731588e3273fdcea2917cd809efbe06550d3b320e078a228e68881aaa05a`。没有在本机启动模型或下载权重。

### 1. 先说人话

多视角三维重建通常默认：不同相机看到的是同一个场景，只是角度不同。压缩会破坏这个默认条件。HEVC、AV1、JPEG 等编码在每张图上产生的块效应、边缘振铃和细节抹除并不相同；同一把椅子的边缘在视角 A 里可能被保留，在视角 B 里却被抹平。二维上看，图片只是“更糊”；三维上看，特征匹配、相机位姿、深度和新视角渲染会把这种不一致解释成错误的几何。

CompMVR 的核心不是让每张图单独变清晰，而是让一组压缩视图共同恢复出**几何上可互相解释的证据**。Stage 1 学一个 codec/QP-aware compression prior；Stage 2 用多视角 diffusion，同时估计局部压缩伪影，并用跨视角 correspondence loss 与 3D attention 约束恢复结果。

最重要的边界：论文证明了“恢复后的输入更适合下游几何估计”，没有证明被压缩或遮挡后完全不可见的表面被真实恢复。生成模型可以把错误细节在多个视角中一致地复制；一致性是必要条件，不是隐藏真值的充分条件。

### 2. 摘要拆解：它真正解决什么

【论文原文事实】作者把 compression-induced cross-view geometric inconsistency 定义为一个独立问题：压缩伪影会破坏 feature matching、dense correspondence、camera estimation 和 downstream 3D reconstruction。CompMVR 构造多 codec、多 QP 的压缩数据，并把 learned compression priors 与 cross-view correspondence 放进 diffusion restoration。

【基于证据的解释】第一层根因是压缩误差的**视角依赖性**：编码器按照局部纹理复杂度分配比特，同一三维边缘在不同视角里的二维频率、面积和遮挡关系不同。第二层根因是三维估计器使用跨图像的相对关系，而传统 restoration 的目标是逐图像素/感知质量；两种目标的梯度方向不一致，单图更漂亮可能让匹配点更不稳定。

### 3. 方法流程与因果链

1. **压缩视图 → Stage 1 latent**：CPE 预测 codec 类型与 QP，同时做图像重建。codec 分类提供离散退化类型，QP 回归提供严重程度。
2. **latent + 压缩多视图 → Stage 2 diffusion**：CAE 从输入估计 spatially varying artifact map，捕捉边缘、纹理和高频区域的局部可靠性。
3. **多视图 latent → cross-view attention/correspondence loss**：对应区域的恢复结果被拉到一致，避免每张视图各自“幻觉”。
4. **恢复视图 → VGGT/MEt3R/新视角渲染**：评估相机位姿、view matching、multi-view consistency 与 novel-view synthesis。

因果链可写成：

`压缩 RGB → codec/QP context + local artifact map → geometry-aware diffusion latent → correspondence-consistent restoration → 更稳定的匹配/位姿/新视角 → 失效于 codec OOD、错误对应、动态物体和真实不可见表面。`

### 4. 关键术语直白解释

- **Compression prior**：不是“记住一张压缩图长什么样”，而是把 codec 类型和 QP 等退化条件编码成条件变量。它解决全局退化上下文缺失；删除后模型只能从局部纹理猜压缩严重程度。QP 只描述量化强度，不包含完整编码结构，因此 QP 相同也不意味着伪影相同。
- **CAE（Compression Artifact Estimator）**：一个轻量卷积网络，输出每个位置的伪影残差图。它解决“同一 QP 下复杂边缘比平坦天空更难恢复”的空间不均匀性。删掉后，模型会用全局条件平均处理所有区域；最易失败是把真实细线、烟雾边界当成压缩噪声。
- **Cross-view correspondence loss**：不是简单让图片像 GT，而是把不同视角中对应点的恢复特征/几何关系拉近。它的作用是让优化目标包含三维可用性；但对应关系一旦错，损失会把两个不同表面强行对齐。
- **3D attention**：让多视图 token 在一个隐空间里交互，给每一张图提供其他视角的补充证据。它不等于显式重建了真实体积；当所有视角都看不到某个表面时，attention 只能调用先验。
- **MEt3R**：衡量生成视图的多视角一致性。低分说明视图间更一致，不代表每个隐藏细节真实存在。

### 5. 公式与优化行为

<a id="E001"></a>
**Source:** HTML §3.1 · Eq. (1–3)

$$
\mathcal{L}_{\mathrm{CaRL}}=\lVert I_{\mathrm{recon}}-I_{\mathrm{gt}}\rVert_1+
\lambda_{\mathrm{QP}}\mathcal{L}_{\mathrm{QP}}+
\lambda_{\mathrm{codec}}\mathcal{L}_{\mathrm{codec}}.
$$

Stage 1 要求 latent 同时保留可重建的视觉信息，并能预测 codec/QP。因果上，它把“退化是什么”从恢复网络的隐式猜测变成显式条件。

<a id="E002"></a>
**Source:** HTML §3.2 · spatial artifact estimator

$$
e(I)=|\partial_x I|+|\partial_y I|.
$$

梯度能量把边缘和纹理标为高复杂区域。它不是压缩伪影的真值；论文用它说明局部复杂度与残差规模相关，再由 CAE 从压缩输入直接估计残差。

<a id="E003"></a>
**Source:** HTML §3.2.1 · Eq. (10)

$$
\mathcal{L}=\mathcal{L}_{\mathrm{CAE}}+\mathcal{L}_{\mathrm{corr}}+\mathcal{L}_{\mathrm{per}},
\qquad
\mathcal{L}_{\mathrm{per}}=\mathcal{L}_{\mathrm{recon}}+\mathcal{L}_{\mathrm{DISTS}}.
$$

三个分量分别约束局部退化、跨视角几何和二维感知/重建质量。删除 $\mathcal{L}_{\mathrm{corr}}$，论文 Table 12 报告 AUC@5 从 0.531 降到 0.516、MEt3R 从 0.132 变为 0.138，说明其几何作用有独立贡献，但增益并不大到可以替代真实深度。

### 6. 图表和实验：问题 → 设计 → 答案

| 问题 | 实验设计 | 论文回答与边界 |
|---|---|---|
| 压缩是否真的损伤几何？ | RealEstate10K，多 codec/QP，VGGT view matching | HEVC QP42 的压缩输入 AUC@5 为 22.59；CompMVR 为 24.34，接近未压缩 GT 的 24.55。说明匹配改善，不能推出隐藏表面真实。 |
| 跨视角约束是否有用？ | Table 12 去掉/保留 $\mathcal{L}_{corr}$ | inlier ratio 0.801→0.805，Sampson 0.082→0.078，AUC@5 0.516→0.531，MEt3R 0.138→0.132。支持“几何目标”而非单纯图像变清。 |
| codec OOD 能否泛化？ | 训练未含 VVC/H.266，Free 数据集测试 | PSNR 23.4970→23.6228，NIQE 3.3412→3.2610，CLIPIQA 0.6917→0.6981。改善很小，不能说已解决任意 codec。 |
| 代价是什么？ | Table 13 比较参数、时间、显存 | Ours 276.171M、1.073 s、11.01 GB；比 DiQP 79.373M/0.422 s/1.02 GB 明显更重。 |

【原文事实】作者在多个 codec、QP 和数据上报告 novel-view synthesis、camera pose、view matching 的提升。Table 3 的多项 AUC@15/30 由 Ours 取得最高或次高，Table 5 的 MEt3R 为 0.132；同时作者明确承认不是 fully feed-forward、显存更高、codec-specific priors 仍不足。

### 7. 贡献、缺陷与深层根因

**贡献可核验**：提出压缩感知多视角 diffusion；把 global codec/QP prior、local artifact map、cross-view correspondence 放进同一恢复目标；构造跨 codec 的多视角压缩评测；用下游几何而非只用 PSNR 评估。

**缺陷 1：合成压缩和真实传输链不完全同构。** 第一层是数据只覆盖预设 codec/QP；真实链路可能经历多次转码、缩放、锐化、屏幕拍摄和时间压缩。第二层是这些操作会改变 correspondence 的空间分布与相机估计难度，训练中的 codec label 不再有唯一解释。迁移到 UAV 时还会叠加运动模糊、滚动快门、风振和极小目标。

**缺陷 2：跨视角一致可能放大共同错误。** 第一层是 correspondence loss 假定匹配正确；重复纹理、反射、透明表面和动态物体会生成错误对应。第二层是多视图网络把同一错误在多个视角中复制，MEt3R 变好而 hidden geometry 变差，形成“自洽幻觉”。

**缺陷 3：压缩恢复仍以 clean view 为目标。** 第一层是像素/感知项偏向某个参考外观；第二层是三维观测本来存在不可观测多解，clean reference 并不能告诉模型被遮挡表面哪一种是真实的。它适合恢复测得的可见证据，不应被解释为 amodal 真值生成器。

### 8. 对 amodal 补全、三维重建、三维 amodal 补全的迁移

- **源机制**：恢复输入时同时保持局部证据质量与跨视角关系。
- **目标同构**：amodal 任务也有 observed pixels/depth 与 hidden geometry；三维重建也要求跨视角一致。
- **必要条件**：正确相机/深度、可验证的 visible reprojection、把 unknown 与 generated 分开存储、允许新视角撤销先验。
- **错配**：CompMVR 处理的是压缩噪声，目标三维 amodal 还包含遮挡、反射、透明、动态和对象级多解；correspondence loss 不能替代形状先验。
- **改造**：把 $\mathcal{L}_{corr}$ 分成 visible correspondence 与 hidden belief consistency；只对可见表面做强一致，对不可见区域输出多个候选并校准不确定度。
- **失败触发器**：所有视角都看不到背面、动态物体跨帧错配、UAV 小目标没有足够像素、域外 codec/传感器使 CAE 把真实结构当伪影。

### 9. ARIS 七项诊断

1. **真正问题与缺口**：现有恢复优化二维质量，忽略压缩导致的跨视角几何错配。
2. **为什么需要该方法**：三维估计依赖相对证据；单图修复无法保证同一结构在不同视角被同样恢复。
3. **为什么可能有效**：codec/QP prior 提供退化上下文，CAE 处理空间异质性，correspondence loss 把几何可靠性写入训练信号。
4. **依赖与失败**：依赖正确对应、静态场景、压缩分布覆盖和稳定相机；论文没有证明概率校准或真实隐藏面恢复。
5. **作者贡献与证据**：Table 3/4/5/12 支持几何指标改善；Table 13 支持计算代价高；没有消融“错误匹配”或“全视角不可见”条件。
6. **未解决缺口与边界**：多解表示、动态场景、真实传输链、可撤销补全、端侧算力。
7. **研究 ideas**：见下方三张 idea 卡，全部是【我们提出的研究假设】，未运行。

#### Idea A：压缩不确定度是否应传给三维后验

- 方法步骤：保留 Stage 1/2，把 CAE 输出转成每像素 reliability；visible surface 用强重投影，低 reliability 区域输出 3 个 hidden occupancy 候选。
- 核心假设：压缩伪影最严重的区域正是隐藏几何最不确定的区域；将其当成单一 clean target 会过度自信。
- 最小实验：合成 20 个 ShapeNet/GSO 物体，四 codec、三 QP、两种遮挡比例；比较 deterministic completion 与 3-hypothesis 的 visible reprojection、hidden IoU、coverage 和 calibration。
- 阳性/阴性/无信号：阳性是 hidden coverage 提高且 visible error 不升；阴性说明主要瓶颈是形状先验；无信号说明压缩并未改变任务多解。
- 贡献：把图像恢复的 reliability 接到三维 amodal belief。
- 风险：需要真实 hidden ground truth；候选可能共享同一错误先验。
- 与原论文关系：复用 compression prior 与 correspondence，改变输出目标，不宣称原文已有。

#### Idea B：反事实错误对应诊断

- 方法步骤：保持压缩强度不变，随机替换 5%/10%/20% 的跨视角匹配；分别训练无 $\mathcal{L}_{corr}$、软权重 correspondence、硬 correspondence。
- 核心假设：论文的几何增益只有在匹配精度超过阈值时才成立；错误对应会把恢复网络训练成一致地错。
- 最小实验：RealEstate10K 小子集或合成平面/重复纹理场景，固定 backbone 与采样，报告 inlier ratio、Sampson、hidden coverage。
- 阳性/阴性/无信号：阳性是软权重在错误匹配下退化更慢；阴性是几何损失仍稳定，说明网络靠其他线索；无信号表示扰动未穿透训练。
- 贡献：回答“几何监督何时从帮助变成伤害”。
- 风险：人工错误匹配与自然失败分布不同。
- 与原论文关系：直接反事实检验 Table 12 的因果解释。

#### Idea C：跨视角可撤销的三维 amodal field

- 方法步骤：场景表示拆成 observed、uncertain、generated 三个 field；CompMVR 只修复 observed evidence；新视角到来时用深度/轮廓更新 uncertain，若冲突则撤销 generated。
- 核心假设：一致性应约束已观察部分，而不是让错误补全在所有视角中自我强化。
- 最小实验：同正面异背面的对象对，首视角压缩、第二视角揭示背面，比较单场和三场的 hidden IoU、visible error、撤销率。
- 阳性/阴性/无信号：阳性是新视角能推翻错误候选且 visible 不漂移；阴性说明生成器没有来源隔离；无信号说明第二视角没有提供区分证据。
- 贡献：把 CompMVR 的几何一致机制改造成可证伪的三维 amodal 状态。
- 风险：三场分解增加显存和候选管理成本。
- 与原论文关系：从其 cross-view consistency 外推，不是原论文声称的能力。

### 10. PaperForge 十二段式

1. **问题和实际重要性**：压缩是三维重建部署中的常见输入条件，却常被当作纯画质问题。
2. **相关工作为何不够**：单图 restoration 的目标不包含跨视角几何；通用 multi-view diffusion 又没有 codec/QP 先验。
3. **反向重建作者路径**：先观察匹配数下降，再区分全局 codec/QP 与局部纹理伪影，最后把 correspondence 写进 diffusion 目标。
4. **核心直觉**：不同视角的错误必须共同修；边缘比平坦区更脆弱；几何可靠性要单独监督；恢复结果仍只是证据，不是隐藏真值。
5. **真实输入例子**：四张 HEVC QP42 的房间图→CPE 识别压缩→CAE 标记门框/文字边缘→多视角 diffusion→VGGT 估位姿→novel view。
6. **公式来源与优化行为**：$\mathcal{L}_{CaRL}$ 让 latent 携带压缩上下文；$e(I)$ 提供局部复杂度线索；$\mathcal{L}$ 把 artifact、correspondence 与 perceptual 三个目标合并。
7. **实验设计→答案**：是否损伤几何→多 codec/QP；是否有效→VGGT/MEt3R/NVS；组件是否必要→Table 8–12；是否可部署→Table 13。答案是几何改善伴随显存开销，不能外推到真实隐藏面。
8. **论文真正留下的结论**：多视角 restoration 需要显式几何目标；单图 PSNR 不是充分指标。
9. **最脆弱隐含前提**：对应关系大体可靠，场景静态，训练压缩链覆盖部署分布。
10. **一周最小复现**：只做合成四视角、两 codec、三 QP；训练小 CAE，比较独立 restoration 与 shared correspondence 的 inlier/Sampson/可见轮廓。
11. **最强反例**：四个视角都看不到柜子背面；模型可生成一致的门，却无法知道门是否存在。
12. **非增量后续**：把 compression-aware evidence router 与可撤销的三维 amodal posterior 合并，而不是继续堆更大的 diffusion。

### 深层诊断卡

| 项目 | 内容 |
|---|---|
| 表面症状 | 压缩后匹配少、位姿漂移、新视角闪烁 |
| 第一层根因 | 视角依赖的压缩伪影破坏对应 |
| 第二层根因 | 二维 restoration 与三维几何目标错配，且不可观测区域本来多解 |
| 方法作用点 | codec/QP prior、CAE、cross-view loss、3D attention |
| 隐含前提 | 对应正确、场景静态、压缩链可覆盖 |
| 失败触发器 | 重复纹理、反射透明、动态物体、OOD codec、所有视角都遮挡 |
| 迁移必要条件 | visible/unknown/generated 分离，新增证据可撤销 |
| 反证实验 | 注入错误 correspondence，测几何损失是否放大共同错误 |
| 证据边界 | 证明几何输入可靠性提升，不证明真实隐藏几何恢复 |
