## 1. 核心实现思路

本段 Shader 采用 **Lambert（兰伯特）漫反射光照模型**，再加上一个**简化的环境光（Ambient）近似**，是实时渲染中最基础、最常见的非 PBR 光照方案。

- **漫反射项（Diffuse）：** 表面颜色 `albedo` × 主光颜色 `mainLight.color` × 明暗因子 `NdotL`。光线越“正对”表面，`NdotL` 越大，表面越亮；越倾斜越暗。

- **环境光项（Ambient）：** 用一个固定颜色 `float3(0.2, 0.25, 0.3)` 乘上 `albedo`。它是对真实世界中**天空光 / 间接光 / 全局光照（GI）**的廉价近似——不计算任何反弹光，只用一个常数模拟“没有光直接照射的地方也不是纯黑”。

- **合成：** `finalColor = 漫反射 + 环境光`。环境光保证了背光面不至于死黑，漫反射负责主要的明暗立体感。

- **特点：** 不包含高光（Specular）、不包含菲涅尔（Fresnel）、不遵循能量守恒，只追求“看起来对”。适合作为理解光照入门的第一个模型，也常作为移动端低配的兜底方案。


## 2. 框架逻辑解析

这段代码位于**片元着色器（Fragment Shader）**中，逐像素计算最终颜色。

### 数据流（从顶点到片元）

1. **顶点着色器**负责把法线从物体空间转换到世界空间（`normalWS`），随插值传入片元。
2. **片元着色器**拿到插值后的法线，配合主光源方向做点积，得到明暗。

### 关键步骤逐行解析

1. **获取主光源：**
   `Light mainLight = GetMainLight();`
   URP 提供的接口，返回一个封装了主平行光（方向、颜色、衰减等）的结构体。这里只用到 `direction`（方向）和 `color`（颜色）。

2. **计算 NdotL：**
   `float NdotL = saturate(dot(normalize(input.normalWS), normalize(mainLight.direction)));`
   - `normalize()`：把法线和光方向都归一化成单位向量。
   - `dot()`：两者点积，得到夹角的余弦值（见第三节）。
   - `saturate()`：把结果截断到 `[0, 1]`。当法线背对光源时点积为负，截成 0，表示背面接不到光。

3. **环境光近似：**
   `float3 ambient = float3(0.2, 0.25, 0.3) * albedo;`
   - `(0.2, 0.25, 0.3)` 三个通道**不相等**，蓝色分量略高，模拟天空偏冷色调的天光（Ambient）。
   - 乘以 `albedo` 让环境光也带上物体本身的颜色。

4. **合成最终颜色：**
   `float3 finalColor = albedo * mainLight.color * NdotL + ambient;`
   - 漫反射 = `albedo × lightColor × NdotL`
   - 加上环境光 = `ambient`
   - 直接相加，即 `最终颜色 = 漫反射 + 环境光`。

5. **输出：**
   `return float4(finalColor, 1.0);`
   - 颜色转成 `float4`，Alpha 通道填 `1.0`，表示完全不透明。


## 3. 涉及的数学与图形学知识

- **点积（Dot Product）：** 两个向量 `a · b = |a| |b| cosθ`。当两个向量都是单位向量时，点积恰好等于夹角的余弦值 `cosθ`。这是计算“两个方向有多接近”的核心工具。

- **兰伯特余弦定律（Lambert's Cosine Law）：** 一个理想漫反射表面接受到的光照强度，与光线和表面法线夹角的余弦成正比。光线垂直照射（θ=0，cos=1）最亮；掠射（θ→90°，cos→0）趋近于 0。这就是 `NdotL` 的物理来源。

- **NdotL（法线点光向）：** 计算机图形学里对 `dot(N, L)` 的惯用叫法。它是漫反射的明暗因子，数值在 `[0, 1]` 之间，0 表示背面，1 表示正对光源。

- **saturate()：** HLSL 内置函数，等价于 `clamp(x, 0, 1)`。用来处理“背面”这种情况——法线背对光源时点积为负，必须截成 0，否则颜色会被减成负数（错误）。

- **归一化 normalize()：** 把向量缩放到长度为 1。只有单位向量点积才等于 `cosθ`，所以做点积前必须先归一化，否则结果会受向量长度干扰。

- **环境光（Ambient）近似：** 真实世界没有纯黑阴影，是因为有天空光、地面反弹等间接光。实时渲染里精确模拟这些代价极高，因此用一个常数颜色近似，代价几乎为零。它是“全局光照（GI）最廉价的替代品”。

- **Albedo（反照率 / 基础色）：** 物体在纯白光照下表现出的固有颜色，本质上是“反射了多少光、吸收了多少光”的比例，不含光照信息。

- **线性空间与能量守恒（延伸）：** Lambert + Ambient 这种“直接相加”的写法不遵循能量守恒（漫反射 + 环境光可能超过入射光），是传统非 PBR 的做法。现代 PBR 会引入 BRDF、间接光漫反射积分等概念，让结果更物理正确。


## 4. 着色器源码

一个最小可运行的 URP 漫反射 Shader，完整复现上述光照逻辑（关键行已注释）：

```hlsl
Shader "Custom/URP_LambertDiffuse"
{
    Properties
    {
        _BaseColor ("Base Color (Albedo)", Color) = (1, 1, 1, 1)
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" "Queue"="Geometry" }

        Pass
        {
            Name "UniversalForward"
            Tags { "LightMode"="UniversalForward" }

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
            };

            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float3 normalWS   : NORMAL;
            };

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseColor;
            CBUFFER_END

            Varyings vert(Attributes input)
            {
                Varyings output = (Varyings)0;
                // 顶点：模型空间 -> 裁剪空间
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                // 法线：模型空间 -> 世界空间
                output.normalWS   = TransformObjectToWorldNormal(input.normalOS);
                return output;
            }

            float4 frag(Varyings input) : SV_Target
            {
                float3 albedo = _BaseColor.rgb;

                // 1. 获取 URP 主光源
                Light mainLight = GetMainLight();
                // 2. 计算漫反射明暗因子 NdotL（背面截断为 0）
                float NdotL = saturate(dot(normalize(input.normalWS), normalize(mainLight.direction)));

                // 3. 环境光近似（蓝色分量略高，模拟冷色天光）
                float3 ambient = float3(0.2, 0.25, 0.3) * albedo;
                // 4. 最终颜色 = 漫反射 + 环境光
                float3 finalColor = albedo * mainLight.color * NdotL + ambient;

                return float4(finalColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```
