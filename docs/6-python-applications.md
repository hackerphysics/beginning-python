# 第6章：Python 应用实战 — 医学影像处理

## 🤖 AI 时代的思考：真正的编程能力是什么？

AI 可以帮你写代码。这是事实，而且这个能力还在快速增强。

但有一件事 AI 替代不了：**把一个模糊的现实问题，转化为可执行的代码方案**。

这个转化过程需要你：
- 理解问题的本质（不是表面需求）
- 把问题分解成可处理的子问题
- 知道用什么工具、用什么算法
- 判断结果是否正确

本章通过**医学影像处理**这个真实领域，带你走一遍完整的思路。不只是教你调 API——而是教你**怎么想**。

---

## 🩺 为什么选医学影像？

### Python 的科学计算生态

用 C++ 做图像处理，你可能会用 OpenCV：

```cpp
// C++ OpenCV
Mat img = imread("scan.png", IMREAD_GRAYSCALE);
Mat binary;
threshold(img, binary, 128, 255, THRESH_BINARY);
```

Python 做同样的事：

```python
# Python scikit-image
import skimage.io as io
from skimage.filters import threshold_otsu

img = io.imread("scan.png", as_gray=True)
thresh = threshold_otsu(img)
binary = img > thresh
```

Python 的优势不只是代码更短——而是整个生态更适合**科学计算**：
- **NumPy**：所有数据都是数组，运算直观高效
- **matplotlib**：可视化调试极其方便
- **scikit-image**：学术级图像处理算法，文档详细
- **pydicom**：专门处理医学影像格式 DICOM

### 医学影像基础概念

**CT（计算机断层扫描）**：用 X 射线从多角度扫描，重建出身体的三维切片图像。每张切片是一张灰度图，像素值表示 **HU（Hounsfield Unit，亨氏单位）**：

| 组织类型 | HU 值范围 |
|---------|----------|
| 空气    | -1000    |
| 肺部（充气）| -600 ~ -400 |
| 脂肪    | -100 ~ -50 |
| 软组织  | 40 ~ 80  |
| 骨骼    | 400 ~ 1000 |

**MRI（磁共振成像）**：利用磁场和射频脉冲成像，对软组织分辨率更好，但不用于这章的例子。

> 💡 **物理直觉**：HU 值本质是 X 射线衰减系数的归一化。水是 0，空气是 -1000，骨骼因为钙含量高所以衰减强，值很大。

---

## 🔧 环境准备

```bash
pip install numpy scipy matplotlib scikit-image pydicom
```

验证安装：

```python
import numpy as np
import scipy
import matplotlib
import skimage
import pydicom

print("All good! 🎉")
```

---

## 📂 读取医学影像数据

### DICOM 格式简介

医学影像通常用 **DICOM**（Digital Imaging and Communications in Medicine）格式存储。DICOM 文件不只是图像——它还包含大量元数据：患者信息、扫描参数、像素间距等。

```
CT_scan.dcm
├── 元数据
│   ├── PatientName: "Anonymous"
│   ├── SliceThickness: 2.5 mm
│   ├── RescaleIntercept: -1024
│   └── RescaleSlope: 1.0
└── 像素数据（PixelData）
    └── 512×512 的灰度图
```

### 用 pydicom 读取 CT 图像

```python
import pydicom
import numpy as np
import matplotlib.pyplot as plt

def load_dicom(filepath: str) -> tuple[np.ndarray, pydicom.Dataset]:
    """
    读取 DICOM 文件，返回 HU 值图像和原始数据集。
    
    为什么要转换成 HU？
    原始像素值是整数存储的，需要用 RescaleSlope 和 RescaleIntercept
    转换成真实的物理单位（HU），才有实际意义。
    """
    ds = pydicom.dcmread(filepath)
    
    # 获取原始像素数组
    pixel_array = ds.pixel_array.astype(np.float32)
    
    # 转换为 HU 值
    slope = float(getattr(ds, 'RescaleSlope', 1.0))
    intercept = float(getattr(ds, 'RescaleIntercept', 0.0))
    hu_image = pixel_array * slope + intercept
    
    return hu_image, ds


# 使用示例
hu_image, ds = load_dicom("CT_slice.dcm")
print(f"图像尺寸: {hu_image.shape}")
print(f"HU 值范围: [{hu_image.min():.0f}, {hu_image.max():.0f}]")
```

### 窗宽窗位（Window Width/Level）

CT 图像的 HU 范围很宽（-1000 到 3000），但人眼只能分辨约 40 个灰度级。**窗宽窗位**就是把感兴趣的 HU 范围映射到显示的灰度范围：

```
显示范围 = [WL - WW/2, WL + WW/2]
```

常用窗口预设：
- **肺窗**：WL=-600, WW=1500（看肺部结构）
- **纵隔窗**：WL=40, WW=400（看软组织）
- **骨窗**：WL=400, WW=1800（看骨骼）

```python
def apply_window(hu_image: np.ndarray, wl: float, ww: float) -> np.ndarray:
    """
    应用窗宽窗位，将 HU 值映射到 [0, 255]。
    
    np.clip 把范围外的值截断，然后线性映射到 [0, 255]。
    这和 C++ 里手写一个 clamp + linear_map 是完全等价的。
    """
    lower = wl - ww / 2
    upper = wl + ww / 2
    windowed = np.clip(hu_image, lower, upper)
    # 线性归一化到 [0, 255]
    windowed = (windowed - lower) / (upper - lower) * 255.0
    return windowed.astype(np.uint8)


# 对比不同窗口
fig, axes = plt.subplots(1, 3, figsize=(15, 5))

axes[0].imshow(apply_window(hu_image, wl=-600, ww=1500), cmap='gray')
axes[0].set_title("肺窗 (WL=-600, WW=1500)")

axes[1].imshow(apply_window(hu_image, wl=40, ww=400), cmap='gray')
axes[1].set_title("纵隔窗 (WL=40, WW=400)")

axes[2].imshow(apply_window(hu_image, wl=400, ww=1800), cmap='gray')
axes[2].set_title("骨窗 (WL=400, WW=1800)")

plt.tight_layout()
plt.savefig("windows_comparison.png", dpi=150)
plt.show()
```

---

## 🫁 实战案例一：CT 图像人体轮廓分割

### 问题定义

**目标**：给定一张 CT 图像，分割出人体所在的区域（排除床板和背景空气）。

这看起来简单，但实际有几个难点：
1. 背景（空气）和人体之间 HU 差异很大，阈值容易定
2. 但 CT 床板也有较高的 HU 值，会干扰结果
3. 人体内部可能有"空腔"（肺、胃），不应该被排除在外

### 思路分析

```
原始 CT 图
    ↓ 阈值分割（HU > -300 认为是物质）
二值图（前景/背景）
    ↓ Flood Fill（从四个角填充背景）
    只保留从外部能到达的背景
    ↓ 取反 + 形态学填洞
人体轮廓 mask
    ↓ 去除小连通域（床板等噪声）
最终结果
```

> 💡 **关键洞察**：直接做阈值分割，空气是背景，但人体内的肺（HU ≈ -600）也会被误判为背景。用 Flood Fill 从图像四角开始填充，只有真正"连通到边缘"的区域才算外部背景，这样肺内部的空气就被正确保留了。

### Flood Fill 原理

类似于图像软件中的"油漆桶"工具：
1. 从种子点（四个角）开始
2. 向四个方向扩展，直到遇到前景像素
3. 所有被访问到的像素标记为"外部背景"

```python
from scipy import ndimage
from skimage import morphology, measure
import numpy as np


def flood_fill_background(binary_mask: np.ndarray) -> np.ndarray:
    """
    从四个角 Flood Fill，标记外部背景。
    
    scipy.ndimage.label 会找连通域，我们利用它来做 flood fill：
    先在图像四周加一圈 0（确保四角连通），然后找到包含角点的连通域。
    
    为什么不用 skimage.segmentation.flood_fill？
    那个是单点出发，我们需要同时从四个角出发，这种方法更简洁。
    """
    # 在四周添加一圈 0（背景），确保四角一定是背景
    padded = np.pad(binary_mask, pad_width=1, mode='constant', constant_values=0)
    
    # 对取反后的 mask 做连通域标记（0=背景，找连通的背景）
    inverted = ~padded.astype(bool)
    labeled, num_features = ndimage.label(inverted)
    
    # 角点 (0,0) 所在的连通域就是"外部背景"
    background_label = labeled[0, 0]
    
    # 外部背景 mask（去掉 padding）
    external_bg = (labeled == background_label)[1:-1, 1:-1]
    
    return external_bg


def segment_body(hu_image: np.ndarray) -> np.ndarray:
    """
    从 CT 图像中分割人体轮廓。
    返回 bool 类型的 mask，True 表示人体区域。
    
    Parameters
    ----------
    hu_image : np.ndarray
        HU 值图像，shape (H, W)
    
    Returns
    -------
    body_mask : np.ndarray
        人体轮廓 mask，shape (H, W)，dtype bool
    """
    # Step 1: 阈值分割
    # HU > -300：空气约 -1000，人体组织最低约 -600（肺），
    # 取 -300 是个保守阈值，能保留大部分软组织
    binary = hu_image > -300
    
    # Step 2: 初步形态学操作——闭运算
    # 闭运算 = 先膨胀后腐蚀，作用：填充小孔洞、连接断裂区域
    # disk(5) 表示半径为 5 像素的圆形结构元素
    selem = morphology.disk(5)
    binary = morphology.binary_closing(binary, selem)
    
    # Step 3: Flood Fill 找外部背景
    external_bg = flood_fill_background(binary)
    
    # Step 4: 人体 = 不是外部背景的区域
    body_rough = ~external_bg
    
    # Step 5: 填洞——人体内部的空腔（肺、胃等）应该包含在人体内
    body_filled = ndimage.binary_fill_holes(body_rough)
    
    # Step 6: 去除小连通域（床板碎片、噪声）
    # remove_small_objects 会删除像素数少于 min_size 的连通域
    body_clean = morphology.remove_small_objects(body_filled, min_size=10000)
    
    return body_clean
```

### 完整可视化代码

```python
def visualize_segmentation(hu_image: np.ndarray, body_mask: np.ndarray,
                           save_path: str = None):
    """可视化分割结果。"""
    fig, axes = plt.subplots(1, 3, figsize=(18, 6))
    
    # 原始图像（肺窗显示）
    display_img = apply_window(hu_image, wl=-600, ww=1500)
    axes[0].imshow(display_img, cmap='gray')
    axes[0].set_title("原始 CT 图像（肺窗）")
    axes[0].axis('off')
    
    # 分割 mask
    axes[1].imshow(body_mask, cmap='gray')
    axes[1].set_title("人体轮廓 Mask")
    axes[1].axis('off')
    
    # 叠加显示
    axes[2].imshow(display_img, cmap='gray')
    # 用红色轮廓线标出人体边界
    contours = measure.find_contours(body_mask.astype(float), 0.5)
    for contour in contours:
        axes[2].plot(contour[:, 1], contour[:, 0], 'r-', linewidth=2)
    axes[2].set_title("分割结果叠加")
    axes[2].axis('off')
    
    plt.tight_layout()
    if save_path:
        plt.savefig(save_path, dpi=150, bbox_inches='tight')
    plt.show()


# ── 主流程 ──────────────────────────────────────────────
if __name__ == "__main__":
    # 加载数据
    hu_image, ds = load_dicom("CT_slice.dcm")
    
    # 人体轮廓分割
    body_mask = segment_body(hu_image)
    
    # 可视化
    visualize_segmentation(hu_image, body_mask, save_path="body_segmentation.png")
    
    # 统计
    body_pixels = body_mask.sum()
    total_pixels = body_mask.size
    print(f"人体区域占比: {body_pixels / total_pixels * 100:.1f}%")
```

> ⚠️ **调试技巧**：如果结果不对，中间加一步 `plt.imshow(binary, cmap='gray'); plt.show()` 看看每一步的输出，这比 C++ 里 `cv::imshow` 方便得多——不需要等待窗口，直接在 Jupyter 里看。

---

## 🫁 实战案例二：CT 图像肺部轮廓分割

### 从人体轮廓到肺部

有了人体 mask，下一步提取肺部。

**核心思路**：在人体区域内，HU 值很低（-600 ~ -400）的连通区域就是肺部（充满空气）。

```python
def segment_lungs(hu_image: np.ndarray, body_mask: np.ndarray) -> np.ndarray:
    """
    在人体轮廓内分割肺部区域。
    
    思路：
    1. 用 HU 阈值找出"低密度"区域（空气 + 肺组织）
    2. 只保留人体内部的低密度区域（排除体外的空气）
    3. 排除气管等细小结构，保留真正的肺叶
    4. 形态学后处理，让边界更光滑
    """
    # Step 1: 低密度阈值（肺部 HU 范围）
    # 空气: -1000, 肺实质: -600~-400
    # 取 -400 作为上限，避免把血管等高密度结构包进来
    low_density = hu_image < -400
    
    # Step 2: 只取人体内部的低密度区域
    # 体外空气虽然 HU 很低，但不在 body_mask 里
    inside_body = low_density & body_mask
    
    # Step 3: 连通域分析，找出大的低密度区域
    # 肺部是最大的两个连通域（左肺、右肺）
    labeled, num_features = ndimage.label(inside_body)
    
    if num_features == 0:
        print("警告：未找到肺部区域")
        return np.zeros_like(hu_image, dtype=bool)
    
    # 统计每个连通域的大小
    region_sizes = ndimage.sum(inside_body, labeled,
                               range(1, num_features + 1))
    
    # 只保留较大的连通域（面积 > 总人体面积的 1%）
    min_size = body_mask.sum() * 0.01
    lung_mask = np.zeros_like(inside_body)
    for label_idx, size in enumerate(region_sizes, start=1):
        if size > min_size:
            lung_mask |= (labeled == label_idx)
    
    # Step 4: 形态学后处理
    # 闭运算：填充肺内血管导致的小孔洞
    selem = morphology.disk(3)
    lung_mask = morphology.binary_closing(lung_mask, selem)
    
    # 填洞：确保肺内部完全填充
    lung_mask = ndimage.binary_fill_holes(lung_mask)
    
    return lung_mask


def visualize_lungs(hu_image: np.ndarray,
                    body_mask: np.ndarray,
                    lung_mask: np.ndarray,
                    save_path: str = None):
    """可视化肺部分割结果。"""
    fig, axes = plt.subplots(1, 2, figsize=(12, 6))
    
    display_img = apply_window(hu_image, wl=-600, ww=1500)
    
    # 左图：原始肺窗
    axes[0].imshow(display_img, cmap='gray')
    axes[0].set_title("CT 图像（肺窗）")
    axes[0].axis('off')
    
    # 右图：分割结果叠加（半透明）
    axes[1].imshow(display_img, cmap='gray')
    
    # 人体轮廓（绿色）
    body_contours = measure.find_contours(body_mask.astype(float), 0.5)
    for c in body_contours:
        axes[1].plot(c[:, 1], c[:, 0], 'g-', linewidth=1.5, label='人体轮廓')
    
    # 肺部区域（青色填充）
    lung_overlay = np.zeros((*hu_image.shape, 4))  # RGBA
    lung_overlay[lung_mask, :] = [0, 0.8, 0.8, 0.4]  # 青色半透明
    axes[1].imshow(lung_overlay)
    
    axes[1].set_title("分割结果（绿=人体，青=肺部）")
    axes[1].axis('off')
    
    plt.tight_layout()
    if save_path:
        plt.savefig(save_path, dpi=150, bbox_inches='tight')
    plt.show()


# ── 完整流程 ────────────────────────────────────────────
if __name__ == "__main__":
    hu_image, ds = load_dicom("CT_slice.dcm")
    
    # 先分割人体，再分割肺部
    body_mask = segment_body(hu_image)
    lung_mask = segment_lungs(hu_image, body_mask)
    
    visualize_lungs(hu_image, body_mask, lung_mask,
                    save_path="lung_segmentation.png")
    
    print(f"左右肺总面积: {lung_mask.sum()} 像素")
    if hasattr(ds, 'PixelSpacing'):
        spacing = float(ds.PixelSpacing[0])  # mm/pixel
        area_cm2 = lung_mask.sum() * (spacing / 10) ** 2
        print(f"换算面积: {area_cm2:.1f} cm²")
```

---

## 🧠 从实战中学到的编程思维

### 1. 把模糊需求拆解成具体步骤

**模糊需求**：「分割人体」

**具体步骤**：
1. 先定义什么是人体（HU 值范围）
2. 处理干扰因素（床板、体外空气）
3. 处理边界情况（人体内部空腔）
4. 评估结果（视觉检查 + 定量统计）

这个拆解过程是你的核心价值。AI 可以帮你实现每一步，但只有你才能把问题定义清楚。

### 2. 算法选择的逻辑

为什么选 Flood Fill 而不是直接阈值？

| 方案 | 优点 | 问题 |
|------|------|------|
| 纯阈值 | 简单快速 | 肺内空气会被当成背景 |
| Flood Fill | 只标记真正连通到边缘的背景 | 稍复杂 |
| 深度学习分割 | 精度高 | 需要训练数据，overkill |

选算法的标准：**够用就好，不过度复杂**。

### 3. 如何验证结果的正确性

```python
def validate_segmentation(body_mask: np.ndarray, 
                           lung_mask: np.ndarray,
                           hu_image: np.ndarray) -> dict:
    """
    定量验证分割结果的合理性。
    
    好的分割应该满足：
    - 人体占图像 20%~60%（过多或过少都不对）
    - 肺部在人体内部（不会超出人体范围）
    - 肺部 HU 均值在合理范围内
    """
    total = body_mask.size
    body_ratio = body_mask.sum() / total
    lung_ratio = lung_mask.sum() / total
    
    # 肺部应该完全在人体内
    lung_outside_body = lung_mask & ~body_mask
    
    # 肺部区域的 HU 统计
    lung_hu = hu_image[lung_mask]
    
    results = {
        "body_ratio": f"{body_ratio:.1%}",
        "lung_ratio": f"{lung_ratio:.1%}",
        "lung_outside_body_pixels": lung_outside_body.sum(),
        "lung_hu_mean": f"{lung_hu.mean():.0f} HU",
        "lung_hu_std": f"{lung_hu.std():.0f} HU",
    }
    
    # 简单的合理性检查
    assert 0.1 < body_ratio < 0.7, f"人体占比异常: {body_ratio:.1%}"
    assert lung_outside_body.sum() == 0, "肺部超出人体范围！"
    assert -700 < lung_hu.mean() < -200, f"肺部 HU 均值异常: {lung_hu.mean():.0f}"
    
    return results
```

> 💡 用 `assert` 做快速验证是个好习惯。结果不对时立刻报错，比悄悄出错然后很难调试要好得多。

### 4. AI 替代不了的能力

| 能力 | AI 能做吗？ |
|------|-----------|
| 根据需求写代码 | ✅ 很擅长 |
| 调用正确的 API | ✅ 很擅长 |
| 定义问题本身 | ❌ 你来做 |
| 判断哪个边界情况重要 | ❌ 你来做 |
| 解释结果是否合理 | ❌ 你来做 |
| 在多种方案中做权衡 | ⚠️ 需要你引导 |

**结论**：学 Python 不是为了记住 API，而是为了训练这种「拆解问题 → 选择工具 → 验证结果」的思维方式。这套思维，在 AI 时代反而更值钱。

---

## 📚 延伸阅读

- [pydicom 官方文档](https://pydicom.github.io/)
- [scikit-image 用户指南](https://scikit-image.org/docs/stable/user_guide/)
- [Radiopaedia](https://radiopaedia.org/)：医学影像学习资源，可以看真实 CT 案例
- [The Lung Image Database Consortium (LIDC-IDRI)](https://wiki.cancerimagingarchive.net/display/Public/LIDC-IDRI)：公开的肺部 CT 数据集，可以用来练手

> 🎯 **下一步**：如果你对医学影像深度学习感兴趣，可以看看 [nnU-Net](https://github.com/MIC-DKFZ/nnUNet)——一个用 Python 写的自动医学影像分割框架，代码质量很高，是学习工程实践的好案例。
