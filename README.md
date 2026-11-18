# Bokeh-Rendering-
Use use AI models to detect the subjects boundary and going through different steps to make sure that the portrait applied is accurate and not bluring the subject


# 🧠 AFDC-Net & MagicBokeh: Model Architecture and Working Mechanism

## 1. Overview

**AFDC-Net (Adaptive Fusion Depth and Contour Network)** is a multimodal deep learning framework designed to generate realistic, depth-aware background blur effects while preserving the sharpness and structural details of the primary foreground subject.

The system integrates semantic understanding, monocular depth estimation, precise boundary detection, and prompt-based segmentation to produce an accurate foreground mask and a refined focal region.

The resulting predictions are passed to the **MagicBokeh** rendering pipeline, which synthesizes a natural-looking optical bokeh effect by selectively blurring background regions according to their estimated depth.

Unlike conventional portrait-blurring techniques that rely exclusively on binary foreground segmentation, this approach combines multiple visual cues to achieve more accurate subject preservation and depth-aware blur transitions.

---

## 2. Overall Pipeline Architecture

The complete processing pipeline consists of the following stages:

1. Input Image Preprocessing
2. Semantic Feature Extraction using DINOv2 with LoRA
3. Monocular Depth Estimation using Depth Anything V2
4. Foreground Boundary Extraction using BiRefNet
5. Adaptive Multimodal Feature Fusion using AFDC-Net
6. Instance-Level Mask Refinement using SAM 2.1
7. Alpha Matte Generation and Edge Refinement
8. Depth-Aware Defocus Disparity Calculation
9. Optical Blur Synthesis using MagicBokeh
10. Final Image Composition and Visualization

### Pipeline Workflow

```text
                  INPUT IMAGE
                       |
                       v
              IMAGE PREPROCESSING
                       |
          +------------+------------+
          |            |            |
          v            v            v
       DINOv2       Depth         BiRefNet
       + LoRA      Anything V2   Boundary
          |            |            |
          v            v            v
      Semantic      Depth       Boundary
      Features     Features     Features
          |            |            |
          +------------+------------+
                       |
                       v
              AFDC-Net v3
          Multimodal Feature Fusion
                       |
                       v
              Focus Prediction
                  & Mask
                       |
                       v
                  SAM 2.1
             Mask Refinement
                       |
                       v
                Alpha Matte
                 Generation
                       |
                       v
             Depth-Aware Defocus
                  Calculation
                       |
                       v
                MagicBokeh
                Blur Synthesis
                       |
                       v
              Alpha Compositing
                       |
                       v
                FINAL IMAGE
```

---

## 3. Detailed Working Mechanism

### 3.1 Input Image Preprocessing

The pipeline begins by accepting an input RGB image.

The image is prepared for the different foundation models through the required preprocessing operations, including:

- Image resizing and normalization.
- Tensor conversion.
- Model-specific input preparation.
- Spatial dimension alignment.
- Preservation of the original image for final rendering.

The preprocessed image is subsequently passed to multiple feature extraction networks.

### 3.2 Semantic Feature Extraction (DINOv2 + LoRA)

**DINOv2** is used to extract high-level semantic visual representations from the input image.

Its primary purpose is to understand the visual content and identify meaningful structures, particularly the primary subject.

The extracted semantic features help AFDC-Net distinguish the important foreground object from surrounding environmental regions.

**LoRA (Low-Rank Adaptation)** is incorporated to adapt the pretrained visual representation to the target task without requiring full backbone fine-tuning.

Key contributions:

- High-level visual understanding.
- Semantic subject representation.
- Context-aware feature extraction.
- Adaptation of pretrained representations.

The resulting semantic feature maps provide the semantic information required for multimodal fusion.

### 3.3 Monocular Depth Estimation (Depth Anything V2)

**Depth Anything V2** estimates the relative depth structure of the input image using a single RGB image.

It generates a dense depth map that represents the estimated spatial relationship between different regions.

The depth map helps the system understand:

- Which regions are closer to the camera.
- Which regions are farther away.
- How foreground and background regions are spatially distributed.
- How much defocus should be applied to different spatial regions.

The predicted depth is normalized and aligned with the other feature representations before being passed to AFDC-Net.

**Important:** Monocular depth estimation provides relative depth information rather than guaranteed metric camera-to-object distances.

### 3.4 Foreground Boundary Extraction (BiRefNet)

**BiRefNet** is responsible for extracting precise foreground boundaries and fine structural details.

Accurate boundary information is particularly important around:

- Hair strands.
- Shoulders and clothing.
- Complex object silhouettes.
- Thin structures.
- Irregular foreground boundaries.

The generated boundary representation provides additional spatial information that helps AFDC-Net distinguish the foreground subject from nearby background regions.

This improves the quality of the predicted focus mask and reduces unwanted background inclusion.

### 3.5 Adaptive Multimodal Feature Fusion (AFDC-Net v3)

AFDC-Net v3 is the central fusion component of the proposed architecture.

It combines three complementary visual representations:

1. **Semantic Features:** Extracted from DINOv2 with LoRA.
2. **Depth Features:** Generated by Depth Anything V2.
3. **Boundary Features:** Extracted using BiRefNet.

These representations are spatially aligned and projected into compatible feature dimensions before fusion.

The fused representation combines semantic understanding, spatial depth information, and precise boundary characteristics.

The network uses this combined information to predict a focus mask and determine the primary in-focus region.

The major advantage of multimodal fusion is that the system does not depend on a single visual representation when identifying the subject.

For example:

- Semantic information helps recognize the primary subject.
- Depth information helps distinguish foreground and background planes.
- Boundary information helps preserve fine structural details.

Together, these cues provide a more informative representation for foreground and focus prediction.

### 3.6 Instance-Level Mask Refinement (SAM 2.1)

The initial focus and foreground predictions are passed to **SAM 2.1 (Segment Anything Model 2.1)** for further refinement.

SAM 2.1 uses automatically generated prompt points to identify and refine the target subject.

Positive prompt points are selected from regions where the predicted foreground representations provide sufficient agreement.

The model generates candidate segmentation masks, and the selected proposal is used to obtain a refined subject mask.

This stage helps:

- Remove unwanted foreground noise.
- Improve subject boundary consistency.
- Preserve internal negative spaces.
- Refine complex object structures.
- Improve foreground-background separation.

The refined mask provides a cleaner representation for subsequent alpha matte generation.

### 3.7 Alpha Matte Generation and Edge Refinement

A binary segmentation mask is not always sufficient for realistic photographic compositing.

Sharp binary boundaries can create visible cutout effects, particularly around hair and fine structures.

To address this problem, the pipeline generates a refined alpha matte.

The process includes:

**Morphological Erosion**

The mask boundary is slightly contracted to reduce background color contamination around the foreground subject.

**Gaussian Feathering**

The mask boundary is softened to create a continuous alpha transition between foreground and background.

The resulting alpha matte contains values between 0 and 1:

- `0.0` represents fully transparent or background regions.
- `1.0` represents fully opaque or foreground regions.
- Intermediate values represent partially blended boundary regions.

This allows the final rendering process to produce smoother and more natural transitions.

### 3.8 Depth-Aware Defocus Disparity Calculation

After obtaining the refined subject mask and depth map, the system calculates the defocus disparity for each pixel.

The calibrated focal depth is estimated using the depth values within the predicted foreground mask.

A representative formulation is:

\[
d_{\text{focus}} =
\operatorname{median}
\left(
D(x,y)\mid M(x,y)>0.5
\right)
\]

Where:

- \(D(x,y)\) represents the estimated depth map.
- \(M(x,y)\) represents the predicted foreground mask.
- \(d_{\text{focus}}\) represents the calibrated focal depth.

The depth difference from the focal plane determines the defocus strength.

A simplified disparity formulation is:

\[
\operatorname{Disp}(x,y)
=
|D(x,y)-d_{\text{focus}}|
\]

The resulting disparity map provides a spatially varying blur signal.

Regions closer to the calibrated focal plane receive less blur, while regions farther from it receive progressively stronger blur according to the rendering configuration.

### 3.9 Optical Blur Synthesis (MagicBokeh)

The MagicBokeh rendering stage converts the predicted defocus information into a photographic background-blur effect.

Instead of applying the same blur strength to the entire background, the system uses depth-aware blur information to control the defocus process.

A configurable aperture blur kernel is used to simulate the appearance of an optical camera lens.

The rendering process aims to reproduce:

- Natural background defocus.
- Progressive blur intensity.
- Subject sharpness preservation.
- Smooth foreground-background transitions.
- A realistic photographic bokeh appearance.

The blur kernel and aperture configuration control the visual characteristics of the resulting bokeh effect.

### 3.10 Final Image Composition

The final stage combines the original sharp image, the generated blurred image, and the refined alpha matte.

The compositing operation can be expressed as:

\[
I_{\text{final}}(x,y)
=
\alpha(x,y)I_{\text{sharp}}(x,y)
+
(1-\alpha(x,y))I_{\text{blur}}(x,y)
\]

Where:

- \(I_{\text{final}}\) represents the final output image.
- \(I_{\text{sharp}}\) represents the original sharp image.
- \(I_{\text{blur}}\) represents the generated defocused image.
- \(\alpha\) represents the refined foreground alpha matte.

This operation preserves the foreground subject while blending the blurred background into the final composition.

The output is a depth-aware portrait image with a sharp primary subject and a progressively defocused background.

---

## 4. Role of Each Model in the Architecture

| Model / Component | Primary Responsibility |
|:---|:---|
| **DINOv2 + LoRA** | Semantic visual feature extraction and task adaptation. |
| **Depth Anything V2** | Monocular depth estimation and spatial understanding. |
| **BiRefNet** | Fine-grained foreground boundary extraction. |
| **AFDC-Net v3** | Multimodal feature fusion and focus prediction. |
| **SAM 2.1** | Prompt-based subject mask refinement. |
| **Alpha Matte Processing** | Smooth foreground boundaries and transparency transitions. |
| **MagicBokeh** | Depth-aware optical blur synthesis and final image rendering. |

---

## 5. Explainability and Model Interpretability

To understand how the proposed architecture makes its predictions, the pipeline incorporates explainability techniques.

### 5.1 SHAP (SHapley Additive exPlanations)

SHAP is used to analyze the contribution of different input modalities to the model's predictions.

It helps investigate the relative influence of:

- Semantic visual features.
- Monocular depth information.
- Foreground boundary representations.

By analyzing feature contributions, SHAP provides insights into how different modalities influence the fusion network's output.

### 5.2 LIME (Local Interpretable Model-agnostic Explanations)

LIME provides local explanations of model predictions by perturbing image regions and observing how the predictions change.

The image is divided into superpixels, and their influence on the model's output is analyzed.

This helps identify which image regions contribute positively or negatively to the predicted focus region.

Together, SHAP and LIME provide complementary perspectives on model behavior and interpretability.

---

## 6. Key Advantages of the Proposed Approach

The combination of semantic understanding, depth estimation, boundary extraction, and instance-level refinement provides several advantages:

- **Multimodal Understanding:** Combines semantic, depth, and boundary information rather than relying on a single segmentation signal.
- **Improved Subject Preservation:** Uses multiple visual cues to identify the primary foreground subject.
- **Fine-Grained Boundary Refinement:** Preserves complex silhouettes and detailed structures.
- **Negative-Space Preservation:** Helps retain internal gaps and spaces within the foreground subject.
- **Depth-Aware Blur:** Produces spatially varying defocus instead of uniform background blurring.
- **Natural Alpha Transitions:** Reduces harsh cutout boundaries through alpha matte refinement.
- **Explainable Predictions:** Incorporates SHAP and LIME to investigate model behavior.

---

## 7. Complete Processing Summary

The proposed AFDC-Net and MagicBokeh framework follows a multimodal, multi-stage processing strategy.

First, the input image is analyzed independently by semantic feature extraction, monocular depth estimation, and boundary extraction networks.

Next, AFDC-Net fuses these complementary representations to predict the primary focus region.

SAM 2.1 subsequently refines the predicted subject mask, and alpha matte processing improves the quality of foreground boundaries.

The refined mask and depth information are then used to calculate depth-aware defocus disparity.

Finally, MagicBokeh synthesizes the optical background blur and combines it with the original image through alpha compositing.

The overall objective is to produce realistic depth-aware bokeh effects while maintaining subject sharpness, structural consistency, and natural foreground-background transitions.

---

## 8. Overall Architecture Objective

The central objective of this framework is to move beyond conventional foreground-background segmentation and develop a more comprehensive depth-aware image rendering pipeline.

By combining pretrained visual foundation models with an adaptive multimodal fusion network and optical rendering techniques, AFDC-Net and MagicBokeh aim to bridge the gap between deep-learning-based scene understanding and realistic photographic depth-of-field simulation.
