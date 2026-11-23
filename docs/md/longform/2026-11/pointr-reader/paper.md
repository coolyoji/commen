# PoinTr｜Nature Reader

Xumin Yu、Yongming Rao、Ziyi Wang、Zuyan Liu、Jiwen Lu、Jie Zhou。原文为可选中文本 PDF，14 页，arXiv:2108.08839v1（2021-08-19），会议记录 ICCV 2021。以下保留关键原文摘录、中文翻译、图表定位和公式解释。

[原文 PDF](https://arxiv.org/pdf/2108.08839) · [精读长文](../pointr.md) · [证据索引](source_map.json) · [提取说明](translation_notes.md)

## 摘要与引言

<a id="S001"></a>
**Source:** p.1 Abstract / Introduction。

**Original:** “we present a new method that reformulates point cloud completion as a set-to-set translation problem”

**中文:** 论文把点云补全重写为集合到集合的翻译问题。

**原文事实。** PoinTr 的输入是下采样的部分点云，输出缺失部分及上采样后的完整点云。作者指出点云来自有限视角、遮挡、传感器分辨率和反射等缺失原因，并提出更广的 ShapeNet-55、ShapeNet-34 benchmark。

**证据解释。** set-to-set 允许 Transformer 处理无序 proxy 集合，但并不自动解决未知表面多解。

## 方法

<a id="S002"></a>
**Source:** p.2–4 §3，Fig. 1–3。

**Original:** “we represent the point cloud as a set of unordered groups of points with position embeddings”

**中文:** 模型把点云表示成带位置嵌入的无序点组集合。

**原文事实。** FPS 得到中心，DGCNN 提取局部特征，位置 MLP 加入全局位置。geometry-aware block 通过 kNN 查询邻近 key，聚合局部几何；query generator 从 encoder 输出产生缺失中心和 decoder query；FoldingNet 在每个 proxy 周围生成局部点。[pipeline](https://arxiv.org/pdf/2108.08839#page=3) · [geometry block](https://arxiv.org/pdf/2108.08839#page=4)

<a id="E001"></a>
**E001 · p.3 式 (1)**

$$
V=M_E(F),\qquad H=M_D(Q,V).
$$

$F$ 是输入 point proxies，$V$ 是 encoder 输出，$Q$ 是动态 query，$H$ 是预测的缺失 proxies。

<a id="E002"></a>
**E002 · p.4 式 (2)**

$$
F_i=F'_i+\phi(q_i).
$$

$F'_i$ 是中心 $q_i$ 附近的局部特征，$\phi(q_i)$ 是位置嵌入。它将“局部是什么”与“在物体哪里”分开。

<a id="E003"></a>
**E003 · p.4 式 (3)**

$$
P_i=f(H_i)+c_i,\qquad i=1,\ldots,M.
$$

$H_i$ 是第 $i$ 个缺失 proxy，$c_i$ 是 query generator 产生的局部中心，$f$ 是共享 FoldingNet。

## 损失与结果

<a id="S003"></a>
**Source:** p.4–8 §3.6–4.7。

**Original:** “Our final objective function is the sum of these two objectives J = J0 + J1.”

**中文:** 最终目标函数是两个 Chamfer 目标之和。

**原文事实。** $J_0$ 监督稀疏中心集合，$J_1$ 监督高分辨率完整点集。ShapeNet-55 表 1 的 PoinTr CD-Avg 为 1.09、F-Score 为 0.464；ShapeNet-34 表 2 的未见类 CD-Avg 为 2.05、F-Score 为 0.384；PCN 表 3 的 CD-$\ell_1$ 平均为 8.38。消融表 4 显示 query generator、DGCNN、geometry block 逐步改善。KITTI 表 5 的 PoinTr Fidelity 为 0.000、MMD 为 0.526。[表 1–2](https://arxiv.org/pdf/2108.08839#page=6) · [表 3–5](https://arxiv.org/pdf/2108.08839#page=7)

**证据边界。** KITTI 没有完整车身真值，Fidelity/MMD 不能直接评价隐藏面。ShapeNet 的 CD 仍允许多个点共享最近邻，不能独立证明拓扑和实例身份。

## 讨论与术语表

<a id="S004"></a>
**Source:** p.7–8 §4.6–5。

**中文:** geometry block 只放在第一层时略优于所有层都放，提示几何归纳偏置既能帮助也可能过拟合。论文结论是结构化 proxy、动态 query 和多尺度生成适合多样点云补全。

| Term | 中文解释 |
|---|---|
| Point proxy | 局部区域的特征 token，带中心位置。 |
| Geometry-aware block | 以 kNN 几何邻域补充语义 attention。 |
| Dynamic query | 由当前输入生成的缺失区域初始状态。 |
| Set-to-set translation | 从无序输入集合预测无序输出集合。 |
| Chamfer Distance | 双向最近邻点集距离；不等同于拓扑正确。 |
