# PoinTr: Diverse Point Cloud Completion with Geometry-Aware Transformers

Xumin Yu、Yongming Rao、Ziyi Wang、Zuyan Liu、Jiwen Lu、Jie Zhou。arXiv 首发 2021-08；会议记录为 ICCV 2021。阅读月份 2026-11。论文 PDF 已下载、导入 Zotero，未运行模型。

[原文 PDF](https://arxiv.org/pdf/2108.08839) · [项目页](https://github.com/yuxumin/PoinTr) · [Nature Reader](pointr-reader/paper.md) · [日报](../../2026-11-23.md)

## 1. 问题和实际重要性

PoinTr 从不完整、无序的点云恢复完整点云，把任务重写为 set-to-set translation。直接失败是单视角、遮挡、反射和有限分辨率让输入缺少表面；第一层更深的原因是 max-pooling 将局部关系压到一个全局向量，解码器之后无法找回细节；第二层是缺失区域存在多解，结构先验若没有可见性和自由空间约束，会把常见模板当成实例事实。该方法的价值在于显式保留 point proxies 和几何关系，适合检验三维 amodal 中“全局依赖是否应与局部证据分开”。

## 2. 相关工作为何不够

体素和距离场可直接使用 3D 卷积，却受分辨率立方增长与量化影响。PCN 等点集编解码器处理方便，但 global bottleneck 会丢细节。普通 Transformer 能建模长程依赖，却缺少三维邻域归纳偏置。PoinTr 选择 point proxy 将局部区域压成 token，再在 token 间做 attention，并加入 kNN 几何块。它改善的是结构表示和采样细节，不会消除未知背面的不可辨识性。

## 3. 反向重建作者的思考路径

作者先观察到点云是无序集合，不能照搬序列 Transformer 的逐点输入；于是用 FPS 选中心、DGCNN 提取局部特征、MLP 加入位置编码。再观察到语义 attention 不保证几何邻域，于是用 kNN query-key 几何关系聚合特征。最后让 query generator 从编码器状态产生缺失区域的初始中心，再由 FoldingNet 做多尺度点生成。新 benchmark 则用 55 类、25%–75% 缺失和更广视角测试先验是否只在八类数据上有效。

## 4. 核心直觉

第一，point proxy 是比单个全局向量更适合补全的中间单位：它同时带局部结构和位置。第二，Transformer 负责远距离部件依赖，geometry-aware block 负责局部邻域。第三，动态 query 让“要补哪里”由当前输入决定，而不是固定槽位。第四，预测点集只代表合理完整形状，输入射线、观测身份和隐藏不确定性仍需在目标系统中另行保留。

## 5. 真实输入走完整 pipeline

以 KITTI 车框点云为例：裁出的 LiDAR 点先 FPS 成中心；轻量 DGCNN 对中心附近的点提取局部特征；位置 MLP 与局部 feature 相加得到 point proxies。Transformer encoder 建模代理之间的长程关系，query generator 产生缺失区域中心和动态 query；decoder 通过 self-attention 与 cross-attention 预测缺失 proxies；每个 proxy 经过 FoldingNet 生成局部点，最后与输入点合并。

- 输入证据：XYZ、相机/雷达坐标、可见性、自由空间射线。
- 中间表示：带位置的 point proxies、几何邻域聚合、动态缺失 query。
- 训练信号：粗中心 CD、完整点集 CD、观测区保真、自由空间冲突和跨视角重投影。
- 推理行为：先决定缺失部件的结构位置，再细化局部曲面；未知区可输出多个 query 候选。
- 目标结果：可见三维重建更细，三维 amodal 维持候选而不是把一个 plausible mesh 当真值。
- 失效触发器：错误 kNN、同可见面异背面、非刚体、细长结构和遮挡者点混入目标。

## 6. 公式、变量和优化行为

论文式 (1) 将输入 proxy 集合 $F$ 编码成 $V=M_E(F)$，动态 query 集合 $Q$ 和编码器输出经 decoder 得到缺失 proxy $H=M_D(Q,V)$。这使缺失区域的位置和形状条件化于当前观测。式 (3) 对第 $i$ 个 proxy 的局部点写成 $P_i=f(H_i)+c_i$，其中 $c_i$ 是 query generator 预测的局部中心，$f$ 是共享 FoldingNet。

优化使用粗中心集合 $C$ 与高分辨率完整点集 $G$ 的双向 Chamfer：

$$
J_0=\frac{1}{n_C}\sum_{c\in C}\min_{g\in G}\|c-g\|+
\frac{1}{n_G}\sum_{g\in G}\min_{c\in C}\|g-c\|.
$$

完整输出 $P$ 对应：

$$
J_1=\frac{1}{n_P}\sum_{p\in P}\min_{g\in G}\|p-g\|+
\frac{1}{n_G}\sum_{g\in G}\min_{p\in P}\|g-p\|,\qquad J=J_0+J_1.
$$

CD 仍允许多个预测点共享一个最近邻；它会鼓励覆盖和靠近，但不直接惩罚错误拓扑、不可穿透或把未知区过度填满。三维 amodal 迁移必须增补射线一致性与候选覆盖指标。

## 7. 按“问题→实验→答案”解释结果

ShapeNet-55 以 55 类、25%–75% 缺失和三种难度测试多样性；PoinTr 在表 1 的 CD-Avg 为 1.09、F-Score 为 0.464，优于对照。ShapeNet-34 把 21 类完全留作未见类；表 2 中 PoinTr 在未见类 CD-Avg 为 2.05、F-Score 为 0.384。PCN 数据集表 3 中 CD-$\ell_1$ 平均为 8.38。消融表 4 显示 query generator、DGCNN 和 geometry block 逐步改善；只把 geometry block 放在第一层反而略优，提示更多几何块可能过拟合。KITTI 表 5 的 Fidelity 为 0.000、MMD 为 0.526，但这些是代理指标，不能当完整车身真值。

## 8. 论文真正留下的结论

在完整 ShapeNet 真值和训练分布覆盖足够时，point proxy、动态 query 与单层几何块能同时改善结构依赖和局部细节。多样 benchmark 比八类 PCN 设置更能暴露长尾，且 geometry block 不必堆满所有层。论文留下的是一种结构化点集生成接口，不是对隐藏实例的证据保证。

## 9. 最脆弱的隐含前提

FPS 中心与 kNN 邻域仍能代表可见结构；局部邻域错误时，geometry-aware block 会把错误近邻当作结构。训练完整形状覆盖测试形态；ShapeNet-34 中仍可见显著的难度差。动态 query 的“缺失中心”可由可见代理推断；同前壳异背部时它只能选择先验。KITTI 的目标框干净且可微调；地面、邻车和反射点会把错误几何写入 proxy。

## 10. 一周内可运行的最小复现实验

用 40 个程序化椅子与桌子、四个相机、64 个 proxy 和 $4\times4$ 局部点，比较三种等参数模型：vanilla Transformer、PoinTr、PoinTr 去掉 geometry block。遮挡分三组：随机删点、深度排序挡板、接触遮挡。固定同样的 query 数、采样点、更新步数和三次种子。报告 hidden Chamfer、细杆 F-score、自由空间冲突、可见区误差和候选覆盖。若 PoinTr 只在随机删点提升，说明结构相关缺失仍未解决；若 geometry block 降低 hidden error 但提高自由空间冲突，说明局部先验过强。

## 11. 最强反例

两辆车共享完全相同的前半部点云，一辆后部有备胎，另一辆后部平滑；同一遮挡板覆盖后部。任何 single-shot PoinTr 都只能在两个 shape posterior 中选一个。若再把 point proxy 的位置编码替换成错误坐标，attention 可能保持稳定但把部件放到错误位置。另一个反例是两把相接触的椅子被框到一个实例中，kNN 会强化错误连接。

## 12. 非增量后续方向与三任务迁移

保留 PoinTr 的 proxy/query 接口，但把缺失 query 改成候选集 $Q^{(1:K)}$，每个候选携带观测来源和未知区置信。输入先经过可见性、自由空间和遮挡者分层；geometry block 只在观测点之间传播，未知 proxy 由候选生成。每个候选用观测射线硬约束、跨视角重投影、局部曲面损失和后续新视角的 Bayesian 重评分。必要条件是目标身份、相机或雷达坐标和缺失区域的真实训练真值；最危险错配是把语义相似度当几何证据。

二维完整 mask 可把 proxy 投影成候选边界，保留可见像素硬标签，隐藏边界输出分歧范围。可见三维重建直接保留测得点和姿态，proxy 只补表面采样。三维 amodal 对每个未知区域报告候选覆盖与新增视角后的淘汰率。该迁移是研究假设，没有运行结果。

## 深层诊断卡

表面症状：完整形状细节变好但隐藏部件仍错。第一层根因：max-pooling 或固定 query 丢失局部/缺失位置。第二层根因：未知区多解被 CD 和单一 query 压成一个训练平均。方法作用点：point proxies、geometry-aware block、dynamic query、multi-scale generation。隐含前提：kNN 邻域和训练形状先验与真实结构同构。失败触发器：错邻域、同可见面异背面、非刚体、框内污染、薄部件。迁移必要条件：可见性与自由空间输入、可靠坐标和候选监督。反证实验：同前车壳异后部，以及接触椅子与分离椅子。证据边界：ShapeNet 有完整真值，KITTI 只有 Fidelity/MMD 代理；未运行目标迁移实验。
