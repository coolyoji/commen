# Nature Reader｜Gate3DGen: A Pre-processing Model that Selectively Purifies Real-World Images for 3D Generation

> source_format: `pdf-text`；中文为主、保留关键英文术语。以下是基于 13 页 PDF 的章节式源文读本，不冒充逐段全文翻译。

## 论文定位

【原文事实】Gate3DGen 将真实照片的退化处理放到冻结的 image-to-3D generator 之前，由 router 决定 KEEP/PURIFY，再由 one-step purifier 生成输入。

【证据边界】PDF 可提取文本完整；本次没有逐页视觉裁图，也没有运行官方代码或下载权重，图表数值以正文/补充表格文本为准。

## 公式索引

- [E001](#E001) 路由概率
- [E002](#E002) flow path
- [E003](#E003) one-step endpoint
- [E004](#E004) region/edge loss
- [E005](#E005) total objective

<a id="E001"></a>
**Source:** p.4 E001 · Eq. (3)

$$
p=\sigma(o_{\rm purify}-o_{\rm keep})
$$

**中文说明：** 两个路由 token 的 logit 差转换为 PURIFY 概率。

<a id="E002"></a>
**Source:** p.4 E002 · Eq. (5)

$$
z_t=(1-t)z_0+t\epsilon
$$

**中文说明：** clean latent 与高斯噪声之间的 flow-matching 路径。

<a id="E003"></a>
**Source:** p.4 E003 · Eq. (6)

$$
\hat z_0=\epsilon-v_\theta(\epsilon,E(x_o),h,1),\qquad \hat x_c=D(\hat z_0)
$$

**中文说明：** 在 $t=1$ 处一次速度预测得到 clean latent，再经冻结 decoder 解码。

<a id="E004"></a>
**Source:** p.5 E004 · Eq. (8)

$$
L_{\rm region}=\ell(\delta_i,m_i)+\ell(\delta_i,r_i),\qquad
L_{\rm edge}=\frac12\left(\ell(D_u\delta_i,m_i^u)+\ell(D_v\delta_i,m_i^v)\right)
$$

**中文说明：** 主体 mask、退化差分图和相邻主体边缘共同限制净化结果。

<a id="E005"></a>
**Source:** p.5 E005 · Eq. (9)

$$
L=\lambda_{\rm lat}L_{\rm lat}+\lambda_1L_1+\lambda_{\rm lpips}L_{\rm lpips}
\lambda_{\rm region}L_{\rm region}+\lambda_{\rm edge}L_{\rm edge}
$$

**中文说明：** latent、像素、感知、区域和边缘项的加权和；权重来自补充材料。


## 章节对照


<a id="S001"></a>
### p.1｜摘要与主张

**Source:** p.1 S001

**Original (excerpt):** Gate3DGen: A Pre-processing Model that Selectively Purifies Real-World Images for 3D Generation Xiaolin Li1,2* Zihao Huang2* Fanpeng Meng2 Chen Yang2 Guanjun Wu3,2 Kai Zhang1 Jiemin Fang2‡ Qi Tian2 1Tsinghua University 2Huawei Inc. 3Huazhong University of Science and Technology *Equal contribution ‡Project lead Corresponding authors Abstract Single-image 3D generation has achieved high fidelity on curated benchmarks, yet degrades sharply on real- world images with uneven illumination, blur, and occlusions. Purifying such inputs requires deciding when to intervene and preserving the visual cues needed for 3D generation, while leaving clean inputs unchanged. We introduceGate3DGen, a selective pre-processing model that gates image inputs to existing 3D generators without retraining them. A degradation-aware router bypasses inputs classified as clean and extracts features to guide purification of the remaining inputs. Conditioned on these features, a 3D-oriented one-step purifier purifies degraded inputs with spatial losses that encourage preservation of object identity, contours, and textural cues. To provide paired supervision, we construct a synthetic data engine that renders clean views of 3D assets and synthesises composite degradations. On the synthetic benchmark, the purifier improves image fidelity and downstream 3D geometry in only 2.3 s on average, reducing TRELLIS.2 mesh distance by 61.5% relative to de- graded inputs. VLM evaluation and human preferences on real photographs further support identity preservation and degradation correction. More demos and code are available atsupra-3d.github.io/gate3dgen. Observed Image 3D Generation Gated 3D Generation Observed Image Observed Image Observed Image 𝒢 𝒢 𝒢 𝒢 𝒢 𝒢 𝒢 𝒢 3D Generation 3D Generation 3D Generation Purified …

**中文对照：** 摘要说明真实图像中的 uneven illumination、blur、occlusion 会破坏三维生成；方法用 degradation-aware router 和 one-step purifier，合成 benchmark 上报告 2.3 秒、foreground LPIPS 降低 39.1%、TRELLIS.2 mesh distance 相对 degraded input 降低 61.5%。


<a id="S002"></a>
### p.2｜问题、相关工作与贡献

**Source:** p.2 S002

**Original (excerpt):** Gate3DGen 1 Introduction Single-image 3D generative models have achieved remarkable progress on curated benchmarks (Hong et al., 2024; Tochilkin et al., 2024; Xu et al., 2024; Xiang et al., 2026; Li et al., 2026; Yang et al., 2026), yet degrade sharply on real-world images. This vulnerability stems from restrictive assumptions about input images. Contemporary models assume centered, sharp, and fully visible objects under uniform studio lighting (Xu et al., 2024; Xiang et al., 2025; 2026). In real-world scenarios, however, unconstrained photographs routinely exhibit blur, uneven illumination, or occlusions (Cho et al., 2025; Qu et al., 2026), corrupting visual cues and inducing severe artifacts in 3D outputs. Open-source image-to-3D pipelines commonly use background removal and recentering (Tochilkin et al., 2024; Xu et al., 2024; Xiang et al., 2025; Zhao et al., 2025), which do not directly address photometric corruptions or foreground occlusions. Specialized preprocessing also targets individual factors such as illumination through delighting (Li et al., 2025a; Zhao et al., 2025). When multiple degradations coexist, correcting one factor can leave other sources of ambiguity in the conditioning image. Our comparisons show that generic 2D image editing can alter object appearance or contours, with adverse effects on downstream geometry (Section 4.4). The objective is therefore to recover a suitable observation of the same object, retaining the visible structural evidence that the 3D generator uses to infer shape and appearance. This motivates selective input purification that preserves visual cues for 3D generation while avoiding unnecessary changes to already suitable clean inputs. We therefore introduce Gate3DGen, a selective pre-processing model that gates real-world …

**中文对照：** 引言把问题归因于 curated benchmark 的输入假设，并强调普通 2D editing 可能改变 identity 与 contour；贡献是 selective purification、router+purifier、以及带 3D ground truth 的合成数据引擎。


<a id="S003"></a>
### p.3｜数据引擎与总览

**Source:** p.3 S003

**Original (excerpt):** Gate3DGen 3D AssetsMulti-view Rendering “Generate a realistic photo with this subject…” Anchor View SelectionClean ReferenceDegraded Image (A) Data Engine Joint Multimodal Self AttentionJoint Multimodal Self Attention (B) Gate3DGen Window/FullSelf AttentionM Window/FullSelf Attention NCausal GQASelf AttentionM Causal GQASelf Attention Degradation-Aware RouterM NObserved Image 3D ObjectMulti-view RGB Render ℛ!𝑥" $𝑦 𝑥"𝑥#𝑦 N 𝒞! 𝒢ℰ 𝒟 Purified Image$𝑥# 𝑥! ℎ #𝛾"𝛾=1 “Select the viewpoint with the most geometric visibility”“Degrade this photo with reflections/ occlusions / …” One-Step Purifier Figure 2. Overview of Gate3DGen. (A) Data Engine.We render 3D assets from multi-view, select informative anchor views, and apply composite degradations to construct aligned observation-reference pairs (xo, xc).(B) Gate3DGen.Router Rϕ bypasses inputs classified as clean (bγ= 0). Otherwise (bγ= 1), its features h guide the one-step DiT purifier Cθ. bxc conditions the frozen 3D generatorG. Clean references and spatial losses are used only in training. of occluded geometry through volumetric reconstruction or multi-view synthesis (Cho et al., 2025; Qu et al., 2026; Shi et al., 2025). Our work focuses on selective input purification before frozen 3D generators, addressing composite degradations involving uneven illumination, blur, and occlusions. 2D Image Editing and Degradation Routing.Recent advances in instruction-guided editing (Black Forest Labs et al., 2025; Wu et al., 2025; 2026a; Liu et al., 2025; Zhang et al., 2026; Qiao et al., 2026) and all-in-one restoration (Potlapalli et al., 2023; Conde et al., 2024; Lin et al., 2023; Yu et al., 2024; Guo et al., 2024) synthesize visually compelling images. Their objectives do not explicitly measure downstream 3D fidelity. In our comparisons, ge…

**中文对照：** 数据引擎从 3D assets 渲染多视图，选 informative anchors，再合成复合退化，得到 paired observation/reference。图 2 的关键是净化结果再送入冻结的 3D generator。


<a id="S004"></a>
### p.4｜路由与单步 purifier

**Source:** p.4 S004

**Original (excerpt):** Gate3DGen Thepre-processed image bxc is either apurified imagefrom Cθ (PURIFY) or the unchanged input (KEEP). To supervise this objective where matched pairs (xo, xc, γ) are unavailable for real-world images, we construct a synthetic data engine from 3D assets (Section 3.2) to supply training data. 3.2 Synthetic Data Engine Learning Cθ requires paired images (xo i , xc i ) that share identical 3D geometry and viewpoints while differing only in capture quality, whereas learning Rϕ requires examples labeled by whether purification is necessary. Because real photographs provide neither matched clean counterparts nor controlled routing labels, we use high-fidelity 3D assets to construct this supervision (Figure 2A). Anchor view selection.From about 4k high-fidelity 3D assets in TexVerse (Zhang et al., 2025), we render eight views under uniform lighting. A vision-language model (VLM) ranks these views by geometric visibility and self-occlusion. The four most informative views per asset serve as multi-view anchors and matched clean referencesx c i. Degradation synthesis.For each anchor, instruction-guided editing produces a matched observation xo i with composite corruptions, including uneven illumination, blur, and occlusions. Each pair is audited by Qwen3-VL (Qwen Team, 2025) for structural consistency, and RMBG-2.0 extracts a soft foreground mask mi from the clean reference xc i. This pipeline produces roughly 44k image pairs Dtrain ={(x o i , xc i , mi)} for Cθ and 100k routing pairs Droute ={(x j, γj)} for Rϕ, whereγ j = 1denotes PURIFYandγ j = 0denotes KEEP. Training and evaluation assets are disjoint. 3.3 Degradation-Aware Router Purifying clean inputs can introduce unwanted changes. To guide purification of images with spatially varying degradations, we use features e…

**中文对照：** 路由器输出 Purify/Keep logits 和连续 hidden features；被选中的图像在 latent 空间做一次 flow-matching endpoint 预测。单步设计换取速度，但不是完整迭代采样。


<a id="S005"></a>
### p.5｜空间损失与训练目标

**Source:** p.5 S005

**Original (excerpt):** Gate3DGen 3D-oriented spatial constraints.For a training triple (xo i , xc i , mi)∈ D train, let z0,i =E(x c i ), and letbz0,i andbxc i be obtained from Equation 6. Standard pixel reconstruction losses treat all regions uniformly. Generic image editing methods overwrite uncorrupted regions and blur high-frequency shading cues that 3D generators rely on. The region- weighted reconstruction and edge losses guide purification by supervising the subject, emphasizing degradation-affected regions, and encouraging preservation of edges. We obtain a soft degradation map ri by smoothing the pairwise residual between xo i and xc i. For an error tensor a and a nonnegative spatial weight w broadcast to the shape of a, we define the normalized weighted mean ℓ(a, w) = P j w(j)|a(j)| max P j w(j),1  ,(7) where the parenthesized subscript (j) indexes channel and spatial entries within each tensor after broadcasting, while i indexes training samples. Withδ i =bxc i −x c i, the implemented losses are Lregion = ℓ(δi, mi) + ℓ(δi, ri),L edge = 1 2  ℓ(Duδi, mu i ) + ℓ(Dvδi, mv i )  ,(8) The two terms in Lregion are normalized independently: the first supervises the full subject through mi, while the second focuses on degradation-affected regions through ri. The edge term matches finite differences along the horizontal (u) and vertical (v) image axes, weighted by adjacent subject-mask values. These weights guide purification, and selectivity refers to image-level routing. Section A details degradation-map pooling and adjacent-mask edge weights. Total training objective.Combining spatial constraints with the backbone objectives gives the full purifier loss: L=λ latLlat +λ 1L1 +λ lpipsLlpips +λ regionLregion +λ edgeLedge,(9) where Llat = MSE(bz0,i, z0,i) supervises latent reconstruction at …

**中文对照：** region loss 同时强调主体和退化差分区域，edge loss 约束有限差分；latent、L1、LPIPS、region、edge 共同优化，但监督仍来自二维 clean reference。


<a id="S006"></a>
### p.6｜实验设置与主要结果

**Source:** p.6 S006

**Original (excerpt):** Gate3DGen Table 1. Degradation-aware routing on synthetic and real images (%).Recall is for PURIFY. Accuracy is overall (PURIFY and KEEP).H = 2 Recall×Accuracy/(Recall + Accuracy). Benchmark In-the-wild Method Recall Accuracy H↑ Recall Accuracy H↑ mPLUG-Owl2 100.00 54.12 70.23 0.00 41.67 0.00 DepictQA-Wild 99.54 53.42 69.53 100.00 58.3373.68 LIQE-MIX 51.66 65.48 57.75 17.86 48.96 26.17 NIQE 98.52 53.32 69.19 75.00 54.17 62.91 Qwen2.5-VL 83.64 74.21 78.64 14.29 43.75 21.54 MANIQA 79.57 89.04 84.04 12.50 45.83 19.64 GPT-5 72.27 77.58 74.83 – – – Ours 88.45 90.77 89.59 89.29 58.33 70.56 Table 2. Synthetic 2D purification.Fidelity metrics use clean references. Input is unedited, Grad. is Sobel gradient MAE, and IoU is foreground mask overlap. ∗ denotes the commercial closed-source model. Method Time (s)↓ LPIPS↓PSNR↑SSIM↑Grad.↓IoU↑ Degraded (Input) – 0.2262 20.254 0.8323 0.0584 0.9537 FLUX.1 30.2 0.2577 17.992 0.7907 0.0688 0.9373 OmniGen2 84.9 0.4166 13.491 0.6478 0.1123 0.6238 Step1X 100.8 0.2426 18.828 0.7761 0.0781 0.8971 Qwen-Edit 81.5 0.2389 18.710 0.7812 0.0727 0.9537 Mage-Flow 15.6 0.2418 19.175 0.7755 0.0835 0.9530 GPT Image-2.5 Flare∗ – 0.2710 16.571 0.7020 0.1319 0.9189 FireRed 83.2 0.2258 15.868 0.7677 0.0814 0.9604 Ours 2.3 0.1378 22.514 0.8708 0.0488 0.9758 outputting PURIFYfor all inputs, resulting in H values of only 69.19–70.23%. MANIQA attains 89.04% accuracy but only 79.57% recall. Our router improves both, indicating that its higher H reflects better detection of inputs needing purification alongside more accurate routing overall. Zero-shot Qwen2.5-VL, which shares the backbone of our router, reaches H = 78.64% . Training on Droute raises recall by 4.81 percentage points and accuracy by 16.56 points, improving H by 10.95 points.On the in-the-wild benchmar…

**中文对照：** 实验把 2D purification、下游 mesh、渲染、路由混淆和推理时间放在同一协议中；结果支持合成数据上的改善，不等于真实隐藏几何被观测。


<a id="S007"></a>
### p.7｜路由/真实图/局限

**Source:** p.7 S007

**Original (excerpt):** Gate3DGen Table 3. Downstream 3D metrics on TRELLIS.2.Input is degraded, and Oracle uses clean references. MD, CD, and F all 1 are ×10−3, and F outer 1 is ×10−1. PSNR-N evaluates normals. LPIPS compares four-view shaded and albedo renderings. ∗ denotes a commercial closed-source model. Method MD↓F all 1 ↑CD↓F outer 1 ↑PSNR-N↑ LPIPS↓Albedo LPIPS↓ Clean reference (Oracle) 0.517 12.46 0.517 1.1586 16.36 0.1548 0.1575 Degraded (Input) 1.984 11.52 1.985 1.0531 15.26 0.1857 0.1916 FLUX.1 1.528 11.06 1.528 1.0238 15.35 0.1864 0.1935 OmniGen2 3.485 6.88 3.485 0.6407 13.11 0.2362 0.2411 Step1X 1.228 10.48 1.229 0.9637 15.25 0.1803 0.1861 Qwen-Image-Edit 1.047 10.81 1.047 1.0123 15.69 0.1780 0.1859 Mage-Flow 1.386 11.42 1.386 1.0422 15.39 0.1797 0.1855 GPT Image-2.5 Flare∗ 1.179 10.39 1.179 0.9720 15.29 0.1792 0.1859 FireRed 0.835 10.58 0.835 0.9889 15.82 0.1701 0.1781 Ours 0.764 11.96 0.765 1.1205 15.98 0.1654 0.1714 Input GT (3D) Original (3D) FireRed (2D/3D) GPT Image 2.5 (2D/3D) Ours (2D/3D) No 3D GT (In-the-wild) No 3D GT (In-the-wild) Figure 3. Comparison of images (2D) and Pixal3D outputs (3D).Original (3D) uses unedited inputs. Ours (2D) uses purified images. Boxes mark structure and appearance changes. methods reduce MD yet all seven lower both F1 scores below the unedited input. Large geometric artifacts, such as an occluder fused into the object (Figure 3), can increase MD, whereas F1 at a small distance threshold is sensitive to local geometric deviations. Generic image editing can therefore reduce large artifacts while introducing local geometric deviations. Only Gate3DGen improves MD and both F1 scores among the compared editors, consistent with the spatial constraints in Equation 8. Albedo LPIPS also improves, consistent with removing input illumination effects bef…

**中文对照：** 路由消融显示视觉 LoRA 改变 recall/accuracy 的权衡；真实照片使用 VLM 与 human preference，缺少真实三维真值。


<a id="S008"></a>
### p.8｜结论与证据边界

**Source:** p.8 S008

**Original (excerpt):** Gate3DGen Table 4. In-the-wild comparisons against 2D editing baselines.FD-DINOv2 is calculated on multi-view renders per method. VLM scores span 0–10. Purification measures degradation removal in both VLM evaluation and the user study. Pref% reports preferences for Gate3DGen over each baseline. ∗ marks commercial closed-source models. FD-DINOv2↓VLM Scores↑User Study (Pref%)↑ Method Pixal3D TRELLIS.2 Identity Purification Quality Identity Purification Quality FLUX.1 0.795 0.790 7.31 6.22 6.24 77.5% 81.0% 72.1% OmniGen2 0.893 0.902 5.43 4.51 4.72 86.0% 93.2% 92.3% Step1X 0.788 0.789 7.91 7.09 5.98 81.8% 83.3% 86.8% Qwen-Image-Edit 0.792 0.810 8.09 6.99 6.60 91.7% 94.6% 82.1% Mage-Flow 0.739 0.769 8.50 7.55 6.22 84.2% 88.1% 80.0% GPT Image-2.5 Flare∗ 0.733 0.722 6.95 6.316.92 58.5% 57.1% 57.5% FireRed 0.771 0.796 8.00 7.086.92 66.7% 63.6% 63.6% Ours 0.716 0.757 8.62 8.00 6.58 – – – Table 5. Purifier ablations on our synthetic benchmark.We report 2D image fidelity and downstream 3D metrics from TRELLIS.2. Checkmarks indicate the use of router features (h) and spatial losses (region-weighted reconstruction and edge). MD, CD, and F all 1 are ×10−3, and F outer 1 is ×10−1. Shaded and albedo LPIPS use the corresponding four-view renderings. Method h Spatial losses 2D LPIPS↓ 2D PSNR↑ MD↓F all 1 ↑CD↓F outer 1 ↑ Shaded LPIPS↓ Albedo LPIPS↓ A 0.135122.292 0.82012.060.820 1.1012 0.1794 0.1781 B ✓ 0.1363 22.572 0.75511.96 0.755 1.12100.1658 0.1716 Full ✓ ✓ 0.1378 22.514 0.764 11.96 0.765 1.1205 0.1654 0.1714 For FD-DINOv2, we fit Gaussians per generator to L2-normalized DINOv2 ViT-L/14-reg CLS features from the 96 photographs and 384 shaded renders (four per asset). Images are resized to 224×224 and ImageNet-normalized. Rendered views are separate samples without pooling. In Table 4…

**中文对照：** 结论是选择性净化可以改善冻结 generator 的输入条件；最脆弱前提是 synthetic degradation 与真实退化相似。


<a id="S009"></a>
### p.12｜补充方法与训练细节

**Source:** p.12 S009

**Original (excerpt):** Gate3DGen Gate3DGen: A Pre-processing Model that Selectively Purifies Real-World Images for 3D Generation Supplementary Materials A Method Details Under a fixed prompt, the degradation-aware router is trained to output the PURIFY/KEEPdecision described in Section 3.3. These decisions correspond to the implementation tokens Purify and Keep, respectively. It updates language-transformer layers 24–27 and the final language normalization. The language-model head and visual backbone are frozen except for rank-16 LoRA adapters in vision blocks 28–31. Training uses token-level supervision with higher weights on the decision tokens and the end-of-sequence token, and with higher sample weights on rare degradation families and on subtle degradations. The discrete routing indicatorbγi follows Equation 4, with the threshold τ frozen on the development set. The language-only router variant omits the vision LoRA adapters. For the purifier Cθ, image-transformer blocks 1–28 remain frozen and blocks 29–49 are updated together with the cross-attention projection layers receiving h. Training uses the flow-matching endpoint at t= 1 in Equation 5, estimates bz0, and reconstructsbxc =D(bz0) by Equation 6. The subject mask mi is the 8-bit soft alpha predicted by RMBG-2.0 from xc i. It is normalized to [0,1] and receives the same resize and random crop as the paired images. The degradation map ri averages |xo i −x c i |/2 over RGB channels and applies 7×7 average pooling with stride 1 and padding 3. In Equation 7, the single-channel weight is broadcast over RGB channels, and each weighted sum is divided by its own broadcast weight sum, clamped below at 1. For the edge term, Du and Dv are one-pixel forward differences, with mu i and mv i given by the products of the two subject-mask values adja…

**中文对照：** 补充给出训练层冻结、mask/差分图构造、loss 权重、15,000 updates/32 GPUs 等细节，并报告 router ablation。


<a id="S010"></a>
### p.13｜补充结果

**Source:** p.13 S010

**Original (excerpt):** Gate3DGen -1.5 -1 -0.5 0 0.5 1 Ours FireRed GPT-Image-2.5 Flare MAGE Observed Clean (oracle) Shaded LPIPS LPIPS PSNR SSIM Grad MAE IoU MD Mesh F1 CD Outer Shell F1 Albedo LPIPS Normal PSNR Figure 4. Oracle-normalized comparison of 2D purification and downstream 3D generation.The radar plot summarizes percentage progress from the observed degraded input (0) toward the matched clean-reference oracle (1) using the metrics reported in Tables 2 and 3. Negative values indicate deterioration relative to the observed input. Blue labels denote 2D image metrics, green labels denote 3D geometry and normal metrics, and orange labels denote rendered-appearance metrics. Lower-is-better metrics are direction-adjusted so that farther outward is always better. Table 7. Downstream 3D metrics on Pixal3D.Input and Oracle use degraded and clean-reference images, respectively. MD and CD are ×10−3. F all 1 is all-surface F1@10−8 (×10−3). F outer 1 is outer-surface F1@10−6 (×10−1). PSNR-N evaluates rendered normals. LPIPS and albedo LPIPS use four-view shaded and albedo renderings, respectively.Boldand underline mark the best and second-best editing results, excluding Input and Oracle. ∗ denotes a commercial closed-source model. Method MD↓F all 1 ↑CD↓F outer 1 ↑PSNR-N↑ LPIPS↓Albedo LPIPS↓ Degraded (Input) 1.221 9.37 1.221 0.8821 10.31 0.2824 0.2805 Clean reference (Oracle) 0.753 10.75 0.753 1.0086 10.42 0.2760 0.2713 FLUX.1 1.214 9.06 1.214 0.8582 10.26 0.2916 0.2906 Mage-Flow 1.259 8.88 1.259 0.8428 10.30 0.2883 0.2864 OmniGen2 3.204 6.00 3.204 0.5700 10.03 0.3051 0.3047 Step1X 1.411 8.79 1.412 0.8289 10.20 0.2911 0.2897 GPT Image-2.5 Flare∗ 1.467 8.29 1.468 0.7864 10.53 0.2825 0.2836 Qwen-Image-Edit 1.150 9.29 1.150 0.8757 10.31 0.2889 0.2883 FireRed 1.148 9.12 1.149 0.863910.77 0.27590.2775…

**中文对照：** Pixal3D 结果显示几何指标改善并非所有 rendering metrics 都最优，提醒不能用单一指标概括“恢复真实”。


## 术语表

| English term | 中文 | 在本文中的作用 | 最易失败处 |
|---|---|---|---|
| degradation-aware router | 退化感知路由器 | 选择 KEEP/PURIFY 并提供条件特征 | 轻微但关键退化被判 KEEP |
| one-step flow matching | 单步流匹配 | 用一次速度估计近似 clean latent | endpoint 近似损失细节 |
| region-weighted loss | 区域加权损失 | 强调主体和退化区域 | 伪 mask/伪差分会强化错误 |
| edge loss | 边缘损失 | 保护轮廓与梯度 | 把真实细杆与噪声边缘混淆 |
| frozen downstream generator | 冻结下游生成器 | 隔离预处理贡献 | 预处理适配某个 generator |


## 读者结论

【论文原文事实】Gate3DGen 在合成退化 benchmark 上提高 2D/3D 指标，并在真实照片上获得 VLM/人评支持。

【基于证据的解释】它最值得迁移的机制是“先判断信息是否可靠，再决定是否调用生成先验”。

【我们的迁移推断】amodal/三维补全应把 visible evidence、unknown region 和 generated hypothesis 分开存储，并允许新视角撤销旧补全；这不是本文已验证的结果。
