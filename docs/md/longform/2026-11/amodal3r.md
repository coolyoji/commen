# Amodal3R: Amodal 3D Reconstruction from Occluded 2D Images｜精读长文

> 原始出处：arXiv:2503.13439v1，2025-03；会议记录：CVPR 2025。阅读归档：2026-11。PDF 已下载并导入 Zotero `每日精读论文/2026/11`，模型未运行。

[Nature Reader](amodal3r-reader/paper.md) · [ARIS](../../../../research/aris/2026-11-21.json) · [日报](../../2026-11-21.md)

## 1. 问题和实际重要性

Amodal3R 研究的是从一个或多个被遮挡的二维图像直接恢复完整 3D object 的 geometry 与 appearance。论文把问题定义为“可见证据、遮挡者和完整对象之间的联合推断”，而不是先把每张图独立补成完整二维图，再交给普通 image-to-3D 模型。这个问题对开放场景、机器人抓取和视频空间记忆有实际意义：真实对象常只露出局部，且不同视角的遮挡区域可能相互矛盾或永久不可见。

表面失败是背面缺失、孔洞或纹理断裂；第一层根因是 2D completion 把图像外观当作形状证据；第二层根因是遮挡导致的多解需要在 3D latent 中保持一致，而逐视角生成会把不同的先验解混成一个错误的几何。论文的贡献是把 occlusion-aware conditioning 放进 TRELLIS 的 3D latent diffusion，而不是把完整二维图像当作硬输入。

## 2. 相关工作为何不够

2D amodal completion 能补边界和纹理，却没有显式体积结构；常规 3D completion 能补几何，却通常没有纹理，也忽略产生部分观测的遮挡模式。论文的关键反例是：对每个视角独立运行 pix2gestalt，再把结果送入 3D generator，会累积多视角不一致。Amodal3R 用 single-stage 3D reconstruction 规避这个接口错位。

## 3. 反向重建作者的思考路径

作者先复用预训练 TRELLIS，使模型已有从图像到 structured latent 的能力；然后只在 transformer blocks 中插入两种轻量条件。第一种用 visibility mask 给图像 token 加权，让模型优先读取确实属于目标且未被遮挡的 patch；第二种单独读取 occlusion mask，告诉模型“不可见的原因可能是前景遮挡”，从而把需要完成的区域和背景区分开。最后用合成 3D assets 生成随机二维遮挡与跨视角一致的接触遮挡，训练模型在同一 3D latent 中完成 geometry 与 appearance。

## 4. 核心直觉

第一，visibility 不是普通 binary token，而是 attention 的证据权重：完全没有可见像素的 patch 不应直接改变 latent。第二，occlusion mask 与 visibility mask 不是同一信息；前者解释“为什么看不到”，后者只说明“现在看不到”。第三，补全与重建必须共享同一 3D latent，否则每个视角的合理生成会在几何上互相打架。第四，多次 stochastic sampling 可以表达多个 plausible assets，但仍不能把先验选择当作未观测背面的真实性证明。

## 5. 真实输入走一遍 pipeline

以一张被椅背挡住的椅子为例：先用 2D segmenter 得到目标 visibility mask `M_vis` 和遮挡者 mask `M_occ`；再用 DINOv2 提取 14×14 patch features。TRELLIS 的 sparse-structure diffusion 和 SLAT diffusion 继续在 3D latent 中去噪。每个 transformer block 先做 mask-weighted cross-attention，再做只读取 `M_occ` 的 occlusion-aware attention，最后由 decoder 输出 3D Gaussian、radiance field 或 mesh。

多视图输入不要求已知 camera pose，但论文按目标 visibility 排序输入，因为早期 diffusion steps 更影响粗几何。这个顺序是一个强先验：若高可见视角本身存在系统性误分割，模型会把错误几何写入早期 latent。迁移到研究系统时，应保存 observed/inferred/unknown 三态和候选集合，而不是只保存一个 mesh。

## 6. 公式、变量和优化行为

### E001｜TRELLIS flow matching

论文将 noisy latent 写为

$$\ell(t)=(1-t)\ell^{(0)}+t\epsilon,\qquad \epsilon\sim\mathcal N(0,I),\quad t\in[0,1].$$

flow loss 为

$$\min_\theta\mathbb E\left\|\upsilon_\theta(\ell(t),x,t)-(\epsilon-\ell^{(0)})\right\|_2^2.$$

它训练的是从噪声回到 3D latent 的速度场。迁移含义是：mask 条件改变的是 denoising path 的证据输入，不是把遮挡区的像素直接复制进 latent。

### E002｜跨注意力相似度

给定 latent token $\ell$ 与 DINOv2 condition $c_{\mathrm{dino}}$：

$$q=W_q\ell,\qquad [k,v]=W_{kv}c_{\mathrm{dino}},\qquad S=\frac{qk^\top}{\sqrt D}.$$

### E003｜visibility 加权 attention

每个 patch 的可见比例为 $c_{\mathrm{vis},j}=\rho(M_{\mathrm{vis},j})$，论文把它加入 attention：

$$A=\operatorname{softmax}\!\left(S+\log c_{\mathrm{vis}}\right),\qquad
A_{ij}=\frac{c_{\mathrm{vis},j}\exp(S_{ij})}{\sum_{k=1}^{K}c_{\mathrm{vis},k}\exp(S_{ik})}.$$

当 patch 完全不可见时，权重为零；这是一种软证据门控，而非对未知区域生成固定值。

### E004｜visibility mask 生成

对目标对象 mask 与遮挡 mask，论文使用

$$M_{\mathrm{vis}}=M_{\mathrm{obj}}\odot(1-M_{\mathrm{occ}}).$$

### E005｜三态解释

论文原文只有 `M_vis` 与 `M_occ`。本读本提出的迁移扩展是

$$s(p)\in\{\mathrm{observed},\mathrm{occluded},\mathrm{background}\},\qquad
p(\text{shape}\mid x,M_{\mathrm{vis}},M_{\mathrm{occ}}).$$

E005 是我们的研究假设，不是论文已验证的公式；它把“遮挡但可能属于目标”和“目标根本不在该像素”分开。

## 7. 实验按“问题→设计→答案”解读

论文在 GSO 与 Toys4K 上训练外的对象进行评测，训练集合为 3D-FUTURE、ABO 和 HSSD，共 20,627 个合成 3D assets。单视图和四视图都比较直接把 occluded image 输入普通 3D generator、先做 pix2gestalt 再重建、以及 Amodal3R。论文报告 Amodal3R 在 FID、KID、CLIP、P-FID、COV、MMD 上优于这些基线；GSO 单视图的表格行给出 FID 30.64、P-FID 7.69、COV 39.61、MMD 3.62，四视图则给出 FID 26.27、P-FID 5.03、COV 38.74、MMD 3.61。这里的数字是论文表格事实，不等同于真实隐藏面恢复率。

消融回答了“两个 mask 模块是否互补”：只用 mask-weighted attention 改善可见纹理但几何仍可能错；只用 occlusion-aware layer 改善几何但外观不稳定；完整模型在表 3 上取得 FID 30.64、KID 0.35、COV 39.61、MMD 3.62。真实场景展示说明模型能在 Replica 和 in-the-wild 图片上生成合理资产，但这些图没有完整真实 3D ground truth，因此只能证明可用性和视觉合理性，不能证明每个背面细节正确。

## 8. 论文真正留下的结论

论文最可靠的结论是：把 visibility 与 occlusion 原因直接注入 3D latent diffusion，能比“二维补全后再三维生成”更一致地处理遮挡；多视图一致的合成遮挡比独立随机擦除更能暴露 contact occlusion 的问题。论文没有证明完全未观测背面唯一可识别，也没有证明合成遮挡与真实遮挡机制完全匹配。

## 9. 最脆弱的隐含前提

第一，训练对象主要是 furniture，论文自己承认复杂或分布外类别可能失败。第二，Segment Anything 给出的 target/occluder masks 足够准确；真实场景中 mask 错误会直接改变 attention 权重。第三，TRELLIS 的 foundation prior 覆盖目标形状；不在先验支持集内的对象会被“合理地”拉回家具原型。第四，多视图排序按 visibility 足以避免早期错误；如果高可见视角包含反光、透明或动态物体，排序反而会放大错误。

## 10. 一周内可运行的最小复现实验

用 20 个 ShapeNet/Objaverse 物体生成四个固定视角；对每个对象构造随机矩形遮挡、深度排序遮挡和 mesh random-walk 接触遮挡。保持同一小型 latent decoder 与采样步数，比较普通 image-to-3D、二维 completion+3D 和 mask-weighted+occlusion-aware 两模块。每个输入采样 8 个候选，报告 observed reprojection、hidden Chamfer、cross-view consistency、candidate coverage 和 visible violation。只在真实 hidden geometry 有 ground truth 的合成集上解释效果。

## 11. 最强反例

同一可见正面对应两个背面完全不同的椅子；所有可达视角都被一堵墙挡住；或者目标是透明/镜面物体。Amodal3R 可以生成两个 plausible samples，但若模型只输出 furniture prior 的高概率模式，视觉质量和 CLIP 仍可能很好。最强反证是 leave-one-view-out：用真实新视角检查候选是否能被淘汰，若候选分歧不随证据变化，说明采样只是随机多样性。

## 12. 非增量后续方向

把 Amodal3R 从“单次生成完整 asset”改成 evidence-aware 3D belief：latent 中显式分出 observed、inferred、unknown channels；用 ensemble candidate disagreement 指导 Fisher/EIG 主动视角；新观测只更新与射线相交的 unknown surface，并允许系统拒答。这个方向的核心不是再加一个 attention，而是让生成结果可被后续观测推翻。

## 13. 三方向迁移判断

- **二维非模态补全：**可迁移 mask semantics 与可见硬一致性，但不能把 3D decoder 的纹理生成当作二维隐藏外观真值。
- **可见三维重建：**最直接受益于 DINO condition、3D latent 和多视图排序；相机未知、动态和 mask 错误仍是主要风险。
- **三维非模态补全：**可作为强 foundation，但必须追加候选后验、三态状态、跨视角 identity 和主动观测；单个生成 mesh 只能标注为 plausible completion。

## 14. 深层诊断卡

- **表面症状：**遮挡区 geometry 断裂、纹理漂移或多视图不一致。
- **第一层根因：**二维外观补全与三维重建接口不匹配，或 mask 原因没有进入条件表示。
- **第二层根因：**未知区是不可观测多解，foundation prior 会把高概率原型误当作实例证据；合成缺失机制还可能与真实接触遮挡结构错配。
- **方法作用点：**visibility-weighted attention、occlusion-aware attention、3D latent diffusion、multi-view visibility ordering。
- **隐含前提：**目标/遮挡者 mask 正确，训练先验覆盖对象类别，合成遮挡能代表部署分布。
- **失败触发器：**分布外类别、透明/镜面、动态物体、永久遮挡、错误 segmenter。
- **迁移必要条件：**有隐藏真值或后续视角做验证，区分 observed/hidden/unknown，并报告候选 coverage 与 visible violation。
- **反证实验：**同 visible 异背面、3D-consistent contact mask、真实新视角 leave-one-view-out。
- **证据边界：**论文证明的是遮挡条件下的 3D 生成质量与跨视角一致性；真实不可见表面真实性和校准后验尚未证明。
