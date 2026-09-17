---
日期: 2026-07-17T16:16:00
tags:
---
# 一、 涉及到的效果

本 Shader 主要用于实现类似“科幻全息投影”、“能量护盾”或“物体扫描”的视觉特效，包含以下三个核心表现：

- **边缘发光（Rim Light / Fresnel）**：模型正对视角时偏透明/暗淡，视线与表面越平行（边缘），发光越强烈。

- **动态扫描纹理（Flow Texture）**：在模型表面叠加一层不断移动的图案（如扫描线或能量波纹），产生动态效果。

- **半透明混合（Transparent Blending）**：利用 `Blend SrcAlpha One`（叠加混合模式）和 `ZWrite Off`（关闭深度写入），使物体呈现非实体的光效堆叠质感。

- 菲涅尔效应 : 随着距离变化而产生的反射效果/清晰度的变化
		    在反射效果中，离你近的反射得更模糊，离你远的反射得更清晰
  https://zhuanlan.zhihu.com/p/357190332



## 二、 核心数学原理

图形学的本质是用数学公式模拟视觉现象。本代码涉及以下关键数学知识：

### 1. 空间变换（矩阵乘法）

- **原理**：将顶点从模型本地坐标系搬移到游戏世界坐标系。

- **公式**：

$$P_{world} = M_{object \to world} \times P_{local}$$

- **代码**：`mul(unity_ObjectToWorld, v.vertex)`。通过内置变换矩阵 `unity_ObjectToWorld` 与顶点坐标相乘实现。


### 2. 向量的归一化与方向计算

- **标准化 (Normalize)**：将任何向量的长度缩放为 $1$，保持方向不变。这是为了消除距离对光照角度计算的干扰。

- **视线向量计算**：目标点减去起点。

    - 公式：
$$\vec{V} = P_{camera} - P_{world\_pos}$$

	- 代码：`_WorldSpaceCameraPos.xyz - i.pos_world`


### 3. 边缘光照的核心：点乘 (Dot Product)

- **原理**：两个单位向量的点乘结果等于它们夹角的余弦值。

- **公式**：

$$\vec{N} \cdot \vec{V} = \cos(\theta)$$

- **应用**：`dot(normal_world, view_world)`。夹角越小（看中心），结果越接近 $1$；夹角越大（看边缘），结果越接近 $0$。

- **反转逻辑**：为了实现边缘亮、中间暗，使用 `1.0 - NdotV` 将结果反转，这就是**菲涅尔效应 (Fresnel Effect)** 的基础模拟。


### 4. 动画原理：线性位移

- **原理**：UV 坐标随时间推移而不断增加，导致采样贴图时位置发生偏移。

- **公式**：

$$S = S_0 + v \times t$$

- **代码**：`uv_flow = uv_flow + _Time.y * _FlowSpeed.xy`（当前坐标 = 初始坐标 + 速度 $\times$ 时间）。


## 三、 逻辑架构

整个 Shader 遵循基础的渲染管线流程，分为三个主要区块：

### 1. Properties (属性面板)

定义了暴露给 Unity 材质面板的参数，作为输入源：

- 主贴图、流动贴图。

- 边缘光的颜色、强度、控制边缘宽度的阈值（`_RimMin`, `_RimMax`）。

- 流动效果的缩放（Tiling）和速度（Speed）。


### 2. 顶点着色器 (Vertex Shader)

顶点着色器的主要任务是**把片元着色器需要用到的各种“世界空间”数据计算好并传递下去**。在这个特效中，世界空间坐标尤为重要。

1. **`o.pos`**: 常规的 MVP 矩阵变换，将顶点从模型空间转换到裁剪空间，用于最终屏幕上的位置显示。

2. **`normal_world` & `pos_world`**: 计算世界空间下的法线方向和顶点位置。这是后续计算“视线与法线夹角（边缘光）”的必需数据。

3. **`o.pivot_world`**: 这是一个很巧妙的操作。通过将本地坐标的原点 `float4(0,0,0,1)` 转换到世界空间，求出了**当前物体中心点（Pivot）的世界坐标**。这将在片元阶段用来生成不易受物体位移影响的流光 UV。

4. **`o.uv`**: 传递模型自带的基础 UV，用于读取遮罩/发光贴图。

### 3. 片元着色器 (Fragment Shader)
<br>

##### 1. 边缘光效层（Rim Light / Fresnel）

``` hlsl
half3 normal_world = normalize(i.normal_world);
half3 view_world = normalize(_WorldSpaceCameraPos.xyz - i.pos_world);
half NdotV = saturate(dot(normal_world,view_world));
half fresnel = 1.0 - NdotV;
fresnel = smoothstep(_RimMin,_RimMax,fresnel);
```

- **思路**：利用菲涅尔效应（Fresnel）制作边缘发光。通过点乘法线（`normal_world`）和视线方向（`view_world`）得到 `NdotV`。视线垂直于表面的地方值为1，平行（边缘）的地方值为0。
    
- `1.0 - NdotV` 将其反转，使得**物体边缘处值最大（最亮）**。
    
- `smoothstep` 用于平滑并控制边缘光的宽窄和硬度。
<br>
##### 2. 局部高亮层（Emissive Mask）

```hlsl
half4 emiss = tex2D(_MainTex,i.uv).r;
emiss = pow(emiss,5.0);
half final_fresnel = saturate(fresnel + emiss);
```

- **思路**：读取主贴图（`_MainTex`）的 R 通道作为额外的发光遮罩。
    
- 使用 `pow(emiss, 5.0)` 是一个经典的图形学技巧，用来**大幅度增加对比度**，把灰暗的区域压黑，只保留贴图中最亮的点。
    
- 最后将“贴图发光”和“边缘发光”相加（`saturate` 防止溢出），整合成最终的发光强度 `final_fresnel`。
<br>
##### 3. 基础颜色与边缘光混合

```hlsl
half3 final_rim_color = lerp(_InnerColor.xyz , _RimColor.xyz * _RimIntensity,final_fresnel);
half final_rim_alpha = final_fresnel;
```

- **思路**：使用上面计算出的发光强度作为插值系数（`lerp`）。
    
- 物体中心部位（发光弱的地方）显示为 `_InnerColor`（内部基础色）。
    
- 物体边缘或贴图高亮部位显示为带有强度的边缘光颜色 `_RimColor * _RimIntensity`。
    
- 透明度（Alpha）也直接由发光强度决定，意味着不发光的地方会变成透明的。
<br>
##### 4. 世界空间流动特效层（Flow Map）

```hlsl
half2 uv_flow = (i.pos_world.xy - i.pivot_world.xy) * _FlowTilling.xy;
uv_flow = uv_flow + _Time.y * _FlowSpeed.xy;
float4 flow_rgba = tex2D(_FlowTex,uv_flow) * _FlowIntensity;
```

- **思路**：制作一个在物体表面移动的流光图案。
    
- 这里**没有使用模型自带的 UV**，而是使用了 `(pos_world.xy - pivot_world.xy)`。这是一种基于世界空间 XY 平面的平面投影映射（Planar Mapping）。
    
- 减去 `pivot_world` 的作用是：确保流光纹理是**锚定在物体身上的**。如果物体在世界空间中移动，流光贴图不会跟着世界坐标滑动产生“穿帮”，而是相对物体原点保持一致。
    
- `+ _Time.y * _FlowSpeed.xy` 让贴图随着时间滚动起来。
<br>
##### 5. 最终输出合成

```hlsl
float3 final_col = final_rim_color + flow_rgba.xyz;
float final_alpha = saturate(final_rim_alpha + flow_rgba.a + _InnerAlpha);
return float4(final_col , final_alpha);
```

- **思路**：将底层的边缘光/基础色（`final_rim_color`）与表层的流光（`flow_rgba.xyz`）进行**加法混合（Additive Blending）**。
    
- 透明度也同样进行累加，并加上一个基础保底透明度 `_InnerAlpha`，最后截断到 0~1 输出。


## 四、 代码实现难点

1. **`saturate` 与 `smoothstep` 的运用**

    - **难点**：不理解为何要频繁限制数值范围。
    
    - **解析**：点乘结果可能出现负数（看背面时），颜色不能为负，必须用 `saturate` 截断在 $0 \sim 1$ 之间。`smoothstep(_RimMin, _RimMax, fresnel)` 则是难点中的难点，它起到了“对比度调节”的作用，把线性过渡的渐变光晕，挤压成具有硬度或特定宽度的科幻光边。
    
2. **基于模型中心的 UV 计算**
    
    - **难点**：常规贴图动画直接使用顶点的 `uv`，而这里使用了 `(i.pos_world.xy - i.pivot_world.xy)`。
    
    - **解析**：由于特效通常要适应不同形状的物体，直接使用模型的世界坐标减去模型的中心点坐标，可以生成一套基于物体真实物理尺寸的正交投影 UV。这样流动纹理就不会因为模型原本展开的 UV 扭曲而变形。
    
3. **Alpha 预乘与加法混合的逻辑**
    
    - **难点**：为何最后是 `final_col + flow_rgba.xyz`（加法），而混合模式又开了 `Blend SrcAlpha One`。
    
    - **解析**：护盾是“发光体”，发光体在物理世界中是能量的叠加。使用加法可以将底层颜色和表层流动特效的高光部分完美融合，不会出现由于直接替换颜色导致的突兀感。关闭 `ZWrite`（深度写入）则是为了防止透明物体遮挡背后的其他特效。