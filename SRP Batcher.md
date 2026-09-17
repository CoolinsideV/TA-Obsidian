---
tags:
  - Unity
  - URP
  - Shader
  - 性能优化
  - SRPBatcher
date: 2026-08-30
aliases:
  - SRP Batcher 合批条件与写法
---
SRP Batcher（可编程渲染管线批处理）是 URP 中大幅降低 CPU SetPass Call 开销的核心优化技术。它的精髓在于：**允许不同材质（只要 Shader 变体相同）进行合批渲染**，通过在 GPU 端利用常量缓冲区（CBUFFER）缓存材质数据来实现。

---

## 一、 SRP Batcher 触发的核心前提条件


### 1. 全局开关已开启
- 必须在当前使用的 **URP Asset** 设置中，勾选 `SRP Batcher` 选项。（通常在 *Quality* 或 *Advanced* 标签下）。

### 2. Shader 与变体（Variant）必须完全一致
- 参与合批的物体必须使用**同一个 Shader**。
- 参与合批的物体必须处于**同一个 Shader 变体**（Shader Variant）。如果材质球 A 开启了 `_ALPHATEST_ON` 宏，而材质球 B 没有开启，它们将无法被合并在同一个 SRP Batch 中。
- **重点：** 材质球（Material）可以是不同的，材质参数（颜色、贴图、浮点数等）也可以不同，这也是 SRP Batcher 最大的优势。

### 3. 支持的渲染组件
- 目前主要支持 `MeshRenderer` 和 `SkinnedMeshRenderer`。
- 不支持 `ParticleSystem`（粒子系统需要走动态批处理或 GPU Instancing）。

### 4. 绝对不能使用 MaterialPropertyBlock (MPB)
- 如果你在 C# 脚本中使用 `MaterialPropertyBlock` 来动态修改某个独立物体的材质属性，该物体将会**打破 SRP Batcher 合批**。
- **原因**：SRP Batcher 依赖材质自身的统一 CBUFFER 内存偏移，MPB 会破坏这种布局。如果大量物体需要逐对象修改属性，请关闭 SRP Batcher 并改用 **GPU Instancing**。

---

## 二、 Shader 编写规范（兼容 SRP Batcher）

为了让自定义 Shader 在 Inspector 面板显示为 **SRP Batcher: compatible**，必须严格遵循内存布局规则：

### 1. 材质属性必须封装在 `UnityPerMaterial` 中
所有在 `Properties` 块中声明的**非纹理/非 Buffer** 变量（`float`, `half`, `float4`, `Color` 等），都必须包裹在名为 `UnityPerMaterial` 的 CBUFFER 块中，且**变量名必须严格一致**。

```hlsl
// 正确做法
CBUFFER_START(UnityPerMaterial)
    float4 _BaseColor;
    float4 _BaseMap_ST; // Tiling 和 Offset 属于 float4，也必须放进来
    float _Metallic;
CBUFFER_END
```

### 2. 纹理和采样器必须放在 CBUFFER 之外
纹理不属于常量缓冲区数据，如果放进 CBUFFER 会直接报错或导致不兼容。

```hlsl
TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);
// ... 然后在下面写 CBUFFER_START(UnityPerMaterial)
```

### 3. 引用 URP Core.hlsl 引入引擎内置 CBUFFER
引擎级别的数据（如 `unity_ObjectToWorld`）由 URP 管理，存放在 `UnityPerDraw` CBUFFER 中。只要引入核心库并使用 URP 提供的宏，就无需手动处理。

```hlsl
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
// 在顶点着色器中使用宏进行变换
// OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
```

---

## 三、 优先级与冲突排查

1. **优先级关系**：
   - 静态批处理 (Static Batching) > SRP Batcher > GPU Instancing > 动态批处理 (Dynamic Batching)。
2. **如何验证合批是否成功？**
   - **检查 Shader**：点击 Shader 资产，看 Inspector 面板底部是否显示 `SRP Batcher: compatible`。
   - **检查 运行状态**：打开 `Window -> Analysis -> Frame Debugger`，点击一个 Draw Call，查看左侧树状图中是否显示为 **SRP Batch**。如果合批失败，右面板通常会给出原因说明（如 "Node has different shader variant"）。

---

## 附：URP 兼容模板 (最简 Unlit)

```hlsl
Shader "Custom/URP_SRPBatcher_Unlit"
{
    Properties
    {
        _BaseMap ("Texture", 2D) = "white" {}
        _BaseColor ("Color", Color) = (1,1,1,1)
        _Cutoff ("Alpha Cutoff", Range(0.0, 1.0)) = 0.5
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            // 1. 纹理和采样器在外部
            TEXTURE2D(_BaseMap);
            SAMPLER(sampler_BaseMap);

            // 2. 标量和向量包裹在 UnityPerMaterial 中
            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                float4 _BaseColor;
                float _Cutoff;
            CBUFFER_END

            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                half4 texColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);
                return texColor * _BaseColor;
            }
            ENDHLSL
        }
    }
}
```


