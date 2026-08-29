

> [!info] 核心概念
> 
> 在渲染管线的最后阶段（逐片元操作阶段），片元着色器 (Fragment Shader) 输出的颜色需要与颜色缓冲区 (Color Buffer) 中已存在的颜色进行合并。这个过程被称为**混合 (Blending)**。混合的核心在于**混合方程式**。

## 1. 核心混合方程式 (The Blend Equation)

绝大多数混合操作都基于以下这个基础数学公式：

$$C_{result} = (C_{src} \times F_{src}) \oplus (C_{dst} \times F_{dst})$$

- $C_{result}$: 最终写入缓冲区的目标颜色 (Result Color)
    
- $C_{src}$: 片元着色器当前计算输出的源颜色 (Source Color)
    
- $C_{dst}$: 帧缓冲区/Render Target 中原本已经存在的目标颜色 (Destination Color)
    
- **$F_{src}$**: **源混合因子 (Source Blend Factor)**
    
- **$F_{dst}$**: **目标混合因子 (Destination Blend Factor)**
    
- **$\oplus$**: **混合操作符 (Blend Operation)**，默认为相加 (Add)
    

## 2. 混合因子字典 (Blend Factors)

在 Shader 编写中，我们通过指定 $F_{src}$ 和 $F_{dst}$ 来决定图层叠加的最终效果。常见的因子如下：

|**Shader 指令**|**数学意义**|**说明**|
|---|---|---|
|`One`|$1$|完全保留该部分的颜色|
|`Zero`|$0$|完全丢弃该部分的颜色|
|`SrcColor`|$C_{src}$|乘以源颜色的 RGB 值|
|`SrcAlpha`|$A_{src}$|乘以源颜色的 Alpha 值（常用）|
|`DstColor`|$C_{dst}$|乘以目标颜色的 RGB 值|
|`DstAlpha`|$A_{dst}$|乘以目标颜色的 Alpha 值|
|`OneMinusSrcColor`|$1 - C_{src}$|乘以源颜色反转后的 RGB 值|
|`OneMinusSrcAlpha`|$1 - A_{src}$|乘以源颜色反转后的 Alpha 值（常用）|
|`OneMinusDstColor`|$1 - C_{dst}$|乘以目标颜色反转后的 RGB 值|
|`OneMinusDstAlpha`|$1 - A_{dst}$|乘以目标颜色反转后的 Alpha 值|

> [!tip] 记忆口诀
> 
> `Blend [SrcFactor] [DstFactor]`
> 
> 前面控制自己（当前片元）留多少，后面控制底色（背景）留多少。

## 3. 经典混合模式速查表 (Cheat Sheet)

### 不透明 (Opaque)

完全覆盖背景，通常用于实体模型。为了性能，通常会直接通过渲染队列 (Opaque Queue) 关闭混合，但其等效指令如下：

- **指令**: `Blend One Zero`
    
- **公式**: $C = C_{src} \times 1 + C_{dst} \times 0$
    

### 传统半透明 (Traditional Alpha Blending)

最常用的透明度混合，适用于玻璃、水体、UI等。自身颜色乘以Alpha，背景颜色乘以 (1-Alpha)。

- **指令**: `Blend SrcAlpha OneMinusSrcAlpha`
    
- **公式**: $C = C_{src} \times A_{src} + C_{dst} \times (1 - A_{src})$
    

### 预乘 Alpha (Premultiplied Alpha)

> [!note] TA 必会概念
> 
> 如果源颜色在进入混合阶段之前（如在图像编辑软件导出或在 Shader 内部计算时），已经提前将 RGB 乘上了 Alpha (即 $C_{src} = RGB_{raw} \times A_{src}$)，则混合因子需要改变。这能解决传统半透明在插值和泛光时的黑边问题。

- **指令**: `Blend One OneMinusSrcAlpha`
    
- **公式**: $C = C_{src} \times 1 + C_{dst} \times (1 - A_{src})$
    

### 正片叠底 (Multiply)

类似 Photoshop 中的正片叠底。源颜色去乘以目标颜色，结果总是变得更暗。常用于贴花 (Decals)、环境光遮蔽 (AO) 叠加、阴影投射。

- **指令**: `Blend DstColor Zero` 或 `Blend Zero SrcColor`
    
- **公式**: $C = C_{src} \times C_{dst} + 0$
    

### 滤色 (Screen)

类似 Photoshop 中的滤色，双重反相相乘再反相，结果总是变得更亮。常用于光晕、全息投影。

- **指令**: `Blend OneMinusDstColor One`
    
- **公式**: $C = C_{src} \times (1 - C_{dst}) + C_{dst} \times 1$
    

### 线性减淡 / 叠加发光 (Additive)

纯粹的光能叠加，将源颜色直接加上去。没有深度感，多层叠加会迅速爆曝（超过 1.0）。极度适合粒子系统、特效 (VFX)、火焰、魔法光效。

- **指令**: `Blend One One` (无视透明度) 或 `Blend SrcAlpha One` (受透明度控制)
    
- **公式**: $C = C_{src} \times A_{src} + C_{dst} \times 1$
    

## 4. 混合操作符 (BlendOp) 进阶

默认的 $\oplus$ 是相加 (Add)，但在特殊的图形效果（如体积雾、自定义深度剔除、特殊的后期处理）中，我们需要修改操作符。

- **`BlendOp Add`**: (默认) $C = (Src \times F_{src}) + (Dst \times F_{dst})$
    
- **`BlendOp Sub`**: $C = (Src \times F_{src}) - (Dst \times F_{dst})$ （常用于热成像、反色特效）
    
- **`BlendOp RevSub`**: $C = (Dst \times F_{dst}) - (Src \times F_{src})$
    
- **`BlendOp Min`**: $C = \min(Src, Dst)$ （忽略混合因子，直接取两者较小值）
    
- **`BlendOp Max`**: $C = \max(Src, Dst)$ （忽略混合因子，直接取两者较大值。在计算距离场或高度图混合时极为有用）
    

## 5. 分离混合 (Separate Blend)

在将半透明物体渲染到 Render Texture (RT) 时，为了保证 RT 输出后的 Alpha 通道数值正确（用于后续 UI 叠加或后处理计算），需要将 RGB 通道和 Alpha 通道的混合逻辑分离开来。

- **语法**: `Blend [RGB源因子] [RGB目标因子], [Alpha源因子] [Alpha目标因子]`
    
- **最佳实践 (渲染半透明物体至 RT)**:
    
    代码段
    
    ```
    // RGB 做普通透明混合，Alpha 通道做预乘叠加防止衰减出错
    Blend SrcAlpha OneMinusSrcAlpha, One OneMinusSrcAlpha
    ```
    

## 6. 颜色通道遮罩 (ColorMask)

除了混合，管线还提供通道级别的写入控制。虽然它不改变数学混合的结果，但决定了最终哪些通道允许被写入 Render Target。

- `ColorMask RGB`: 默认写入颜色，不写入 Alpha
    
- `ColorMask RGBA`: 写入所有通道
    
- `ColorMask 0`: 完全不写入颜色。（**常用技巧**：配合 `ZWrite On` 可以实现只写入深度缓冲 (Depth Buffer) 的不可见遮挡物 (Occluder)，用于实现 X-Ray 透视遮挡或者早期深度剔除 Pre-Z 阶段）。