# PCN: Point Completion Network｜Nature Reader

Wentao Yuan、Tejas Khot、David Held、Christoph Mertz、Martial Hebert。arXiv 首发 2018-08，3DV 正式发表 2018-09。对照版本 v3，2019-09-26，17 页。公开归档采用少量双语摘录、中文逐节解读和原图页码链接。全文在已核验的 Zotero 附件 `DVOWU6PH` 中。

[原文 PDF](https://arxiv.org/pdf/1808.00671v3) · [精读长文](../pcn.md) · [证据索引](source_map.json) · [提取边界](translation_notes.md)

## 摘要、引言与相关工作

<a id="S001"></a>
**Source:** p.1 Abstract，p.1–2 §1–2。

**Original:** “Unlike existing shape completion methods, PCN directly operates on raw point clouds without any structural assumption (e.g. symmetry) or annotation (e.g. semantic class) about the underlying shape.”

**中文:** 与已有形状补全方法相比，PCN 直接处理原始点云，无需对底层形状提供对称性等显式结构假设或语义类别等标注。

**原文事实。** 摘要提出用密集、完整点集表达补全结果。引言以真实激光扫描中的低密度车辆为例，把缺失归因于遮挡和传感器分辨率。§2 分别讨论根据输入插值或复制结构、对齐外部模板、从训练集学习部分到完整映射。作者选择直接处理点集，以避开稠密体素分辨率的代价。

**证据解释。** “无需显式结构假设”指模型接口与标签要求。完整训练数据的形状分布仍提供强先验，而且真实扫描实验实际用了汽车专用模型。真实输入及框内对象补全见 [Fig. 1，p.1](https://arxiv.org/pdf/1808.00671v3#page=1)。

## 任务与方法

<a id="S002"></a>
**Source:** p.2 §3，p.3–4 §4，p.12 App. B–C。

**Original:** “X is not necessarily a subset of Y and there is no explicit correspondence between points in X and points in Y”

**中文:** 输入点集 $X$ 不一定是目标点集 $Y$ 的子集，两个点集之间没有显式逐点对应。

**原文事实。** 两个 PointNet 层的共享 MLP 和最大聚合将输入编码为 1024 维向量。第一层的全局特征拼接到每个点再送入第二层。全连接解码先预测 1024 个中心。每个中心共享一个 MLP，把局部 $4\times4$ 网格、中心坐标及全局特征映射成 16 个三维偏移。拼接局部点得到 16,384 个点。训练直接比较粗、细输出与目标点集，输入点没有硬保留约束。[Fig. 2，p.3](https://arxiv.org/pdf/1808.00671v3#page=3) · [Fig. 10，p.12](https://arxiv.org/pdf/1808.00671v3#page=12)

<a id="equations"></a>
### 公式索引

[E001 CD](#E001) · [E002 EMD](#E002) · [E003 多尺度损失](#E003)

<a id="E001"></a>
**E001 · p.4 式 (1)**

$$
\operatorname{CD}(S_1,S_2)=\frac{1}{|S_1|}\sum_{x\in S_1}\min_{y\in S_2}\|x-y\|_2+\frac{1}{|S_2|}\sum_{y\in S_2}\min_{x\in S_1}\|x-y\|_2.
$$

第一项约束预测点接近真值，第二项鼓励覆盖真值。用未平方的距离。点数可以不同，多点可共用最近邻。

<a id="E002"></a>
**E002 · p.4 式 (2)**

$$
\operatorname{EMD}(S_1,S_2)=\min_{\phi:S_1\rightarrow S_2}\frac{1}{|S_1|}\sum_{x\in S_1}\|x-\phi(x)\|_2.
$$

$\phi$ 是等点数集合间的双射，限制点密度集中。论文采用迭代近似求解。原图 Fig. 5 对比最近邻与一对一匹配的距离分布。[Fig. 5，p.6](https://arxiv.org/pdf/1808.00671v3#page=6)

<a id="E003"></a>
**E003 · p.4 式 (3)**

$$
L(Y_{\rm coarse},Y_{\rm detail},Y_{\rm gt})=d_1(Y_{\rm coarse},\widetilde{Y}_{\rm gt})+\alpha d_2(Y_{\rm detail},Y_{\rm gt}).
$$

$\widetilde{Y}_{\rm gt}$ 是与粗输出同规模的子采样真值。粗级可用 CD 或 EMD，细级始终用 CD。系数 $\alpha$ 调节细级监督权重。两项都从密集完整形状获得梯度。[公式原页](https://arxiv.org/pdf/1808.00671v3#page=4)

## 结果

<a id="S003"></a>
**Source:** p.4–7 §5.1–5.2，p.17 附表 4–7。

**原文事实。** 以 30,974 个 ShapeNet 模型生成八视角部分深度与密集真值。训练分开测试集实例及未见类别。表 1 的 PCN 为 6.85M 参数，FC 为 53.2M，纯 Folding 为 2.40M。附表 4 的已见类平均 CD 最好的是 PCN-CD（0.009636）。附表 5 的平均 EMD 最好的是 PCN-EMD（0.064044）。附表 6 中未见类别两组平均 CD 都由 Folding 获得最低值。Fig. 6 测试合成掩码、噪声和异常深度，不同遮掩率下误差逐步增加。[Fig. 3–4，p.5](https://arxiv.org/pdf/1808.00671v3#page=5) · [Fig. 6 与表 1，p.7](https://arxiv.org/pdf/1808.00671v3#page=7) · [附表 4–7，p.17](https://arxiv.org/pdf/1808.00671v3#page=17)

<a id="S004"></a>
**Source:** p.7–8 §5.3–5.4。

**Original:** “We do not have the complete ground truth point clouds for KITTI.”

**中文:** KITTI 实验没有完整点云真值。

**原文事实。** 车框裁出的 2,483 个部分点集在框坐标内由汽车专用模型补全。表 2 的 Fidelity、近邻模板 MMD 与输出 Consistency 为 0.02800、0.01850、0.01163，输入 Consistency 为 0.05121。2,396 个相邻帧配准对上，Fig. 7 对比补全与原输入用于 ICP 的旋转及平移误差。文章印刷的旋转口径使用两个四元数内积，表达式外层系数与常见主旋转角写法有差别。此处保留原页作为依据，未验证实现。[表 2，p.7](https://arxiv.org/pdf/1808.00671v3#page=7) · [Fig. 7，p.8](https://arxiv.org/pdf/1808.00671v3#page=8)

**证据解释。** 整体结构或点重叠变好解释了这个配准流程为何可能受益。Fidelity 只验证观测部分，模板距离奖励常见车形，帧间一致性也容许同一个错误背面在两帧重复出现。

## 讨论、结论与补充材料

<a id="S005"></a>
**Source:** p.8 §6–7，p.9 Fig. 9，p.12–16 App. D–G。

**原文事实。** Fig. 8 展示薄结构丢失以及断开部件错误连接。作者分别指出低面积带来的点稀疏、局部二维网格的表示限制，以及训练集中对象几乎均连接。Fig. 12 展示车框夹带地面或邻物点造成的失败。附录改变 PointNet 层数与全局瓶颈，展示合成测试结果、最大池化所选点及特征可视化。该组观察没有独立消融出真实车身隐藏面误差。[Fig. 8，p.8](https://arxiv.org/pdf/1808.00671v3#page=8) · [Fig. 12–14，p.13](https://arxiv.org/pdf/1808.00671v3#page=13) · [Fig. 15，p.14](https://arxiv.org/pdf/1808.00671v3#page=14)

**迁移推断。** 同样的点解码器可以给出多个未知面候选。保留点源和相机射线，用之后真正露出的表面更新候选。开放问题包括何时应保留歧义、应该让薄片和细线共享多少参数，以及真实遮挡时新增观测是否真正区分候选。[精读中的反例与实验](../pcn.md#11-最强反例)

## 术语表

| 英文术语 | 中文与作用 |
|---|---|
| Permutation invariance | 置换不变性。打乱输入点行不改特征，适合无序点集。 |
| Global max pooling | 全局最大池化。各通道读最强响应，离群点也能劫持该响应。 |
| Local folding | 局部网格折叠。共享函数将二维贴片映射成三维偏移，易表达平滑曲面，细线或裂缝需要不同支撑。 |
| Chamfer Distance (CD) | 双向最近邻距离。允许多对一，局部点堆积可能少受罚。 |
| Earth Mover’s Distance (EMD) | 等点数下的一对一移动距离，对密度不均更敏感。 |
| Minimal Matching Distance (MMD) | 本文 KITTI 的最近模板距离，衡量典型形状相似性。 |
