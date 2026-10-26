# Nature Reader｜CompMVR: Compression-Aware Multi-View Restoration Using Diffusion Models for Geometrically Consistent 3D Reconstruction

> source_format: `pdf-text`；PDF 为可选中文本层。以下是基于 PDF/HTML 的 source-grounded 章节读本，保留关键英文术语，并把“原文事实 / 基于证据的解释 / 迁移推断”分开。未运行官方代码，未下载权重。

## 论文定位

【论文原文事实】CompMVR 将有损压缩下的多视角 restoration 定义成几何问题，而不仅是 2D 画质问题。它用 compression prior、spatial artifact estimator、cross-view correspondence loss 和 3D attention 恢复更适合三维估计的多视角证据。

【证据边界】PDF 与 arXiv HTML 可提取，公式以 HTML MathML/正文核对；本读本不声称逐页复现所有图像裁剪，也不把多视角一致性当作隐藏表面真值。

## 术语表

| Canonical term | 中文解释 | 作用和边界 |
|---|---|---|
| compression prior | 压缩先验 | 编码 codec/QP 的全局退化上下文；不等于完整压缩历史 |
| CAE | compression artifact estimator | 预测局部空间伪影；可能把真实细结构误当伪影 |
| correspondence loss | 跨视角对应损失 | 约束多视角关系；依赖匹配正确 |
| 3D attention | 三维感知注意力 | 让视图 token 共享证据；不等于显式体积真值 |
| MEt3R | multi-view consistency metric | 测视角一致性；不能单独证明真实性 |

## 公式索引

- [E001](#E001) Stage 1 compression-aware representation objective
- [E002](#E002) local gradient energy
- [E003](#E003) Stage 2 joint objective

<a id="E001"></a>
**Source:** PDF/HTML §3.1 · Eq. (1–3)

**Original (source-faithful excerpt):** “The final Stage 1 loss combines the reconstruction loss and the compression-aware losses.”

$$
\mathcal{L}_{\mathrm{CaRL}}=\lVert I_{\mathrm{recon}}-I_{\mathrm{gt}}\rVert_1+
\lambda_{\mathrm{QP}}\mathcal{L}_{\mathrm{QP}}+
\lambda_{\mathrm{codec}}\mathcal{L}_{\mathrm{codec}}
$$

**中文：** Stage 1 不仅要求重建图像，还要求 latent 能预测 QP 和 codec。这样网络必须编码退化条件，而不是只记住外观。

<a id="E002"></a>
**Source:** PDF/HTML §3.2 · spatially-variant artifact estimation

**Original (source-faithful excerpt):** “Low-energy regions correspond to smooth areas, whereas high-energy regions capture texture-rich or edge-dominant structures.”

$$
e(I)=|\partial_x I|+|\partial_y I|
$$

**中文：** 梯度能量用来说明局部纹理复杂度与压缩残差相关。它是分析线索，不是伪影真值。

<a id="E003"></a>
**Source:** PDF/HTML §3.2.1 · Eq. (10)

**Original (source-faithful excerpt):** “The overall objective in Stage 2 is jointly defined as”

$$
\mathcal{L}=\mathcal{L}_{\mathrm{CAE}}+\mathcal{L}_{\mathrm{corr}}+\mathcal{L}_{\mathrm{per}},
\qquad
\mathcal{L}_{\mathrm{per}}=\mathcal{L}_{\mathrm{recon}}+\mathcal{L}_{\mathrm{DISTS}}
$$

**中文：** Stage 2 同时优化局部伪影估计、跨视角几何和感知恢复。公式没有创造新的观测，只改变网络如何使用已有多视角证据。

## 章节对照

### S001｜摘要与引言

**Original:** The paper identifies compression-induced cross-view geometric inconsistency as a critical failure mode.

**中文：** 作者把核心失败定义为“压缩导致的跨视角几何不一致”。关键不是图像单独变糊，而是匹配关系被破坏，随后影响位姿、深度和新视角。

### S002｜相关工作缺口

**Original:** Existing compression restoration methods are primarily image or video centric.

**中文：** 既有方法主要优化 image/video restoration 的视觉质量，缺少面向三维重建的 correspondence objective。作者以 SIR-Diff 等多视角恢复为对照，强调它们没有使用 codec/QP-specific prior。

### S003｜方法总览

**Original:** Stage 1 learns a compression-aware latent; Stage 2 performs multi-view restoration conditioned on it.

**中文：** Stage 1 给出全局退化上下文；Stage 2 用 CAE 给局部上下文，并通过 cross-view loss 与 3D attention 共享证据。输入是 4 个压缩视图，输出是 4 个互相更可解释的恢复视图。

### S004｜实验结果

**Original:** On HEVC QP42, the reported AUC@5 improves from 22.59 for compressed input to 24.34 for Ours.

**中文：** Table 4 的 HEVC QP42 view matching AUC@5 从压缩输入 22.59 提到 24.34，而未压缩 GT 为 24.55。这说明恢复接近未压缩几何证据，但仍有差距。

### S005｜消融与讨论

**Original:** Adding correspondence supervision improves the geometry-side metrics.

**中文：** 去掉 correspondence loss 后，inlier ratio 0.805→0.801，Sampson 0.078→0.082，AUC@5 0.531→0.516，MEt3R 0.132→0.138。增益支持几何损失的作用，但数值幅度也说明它是补充项，不是完整解决方案。

### S006｜限制

**Original:** The framework requires additional computation and GPU memory and is not yet fully feed-forward.

**中文：** 论文承认 276.171M 参数、约 1.073 秒、11.01 GB 峰值显存的代价；codec-adaptive priors 仍需加强。它没有证明真实动态场景、所有传输链和不可见表面恢复。

## Nature Reader 结论

论文写清楚的是：压缩会破坏三维所需的跨视角证据，恢复模块必须把几何可靠性加入目标。作者没有写成、也没有证据证明的是：模型能从完全不可见的表面恢复该实例的真实几何。迁移到三维 amodal 时，应保留 observed/unknown/generated 的来源分离，并允许新视角撤销生成先验。
