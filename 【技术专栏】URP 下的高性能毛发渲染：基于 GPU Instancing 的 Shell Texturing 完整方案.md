
在实时渲染领域，毛发渲染（Fur/Hair Rendering）一直是个充满挑战的课题。对于移动端或对性能要求较高的中端 PC 项目而言，真实的 Strand-based 发丝渲染算力成本过高。相比之下，多层外壳技术（Shell Texturing）凭借出色的绒毛质感和可控的开销，成为了动物体表、植被地衣等材质的首选方案。

本文将为您深度拆解一套基于 **Unity URP + GPU Instancing** 优化的 Shell Texturing 毛发渲染方案，并提供从 C# 驱动到 HLSL 着色器的完整源码。

## 一、 快速上手：在 Unity 中复现效果

在深入底层逻辑之前，我们先来看看如何快速将这套系统跑起来。只需三个步骤：

1. **创建材质与 Shader**：新建一个 Shader 文件（命名为 `URP_StaticInstancedFur`），将本文末尾的 Shader 源码粘贴进去。基于该 Shader 创建一个材质，并为其赋予一张噪波图（用于生成毛发发丝）和一张长度遮罩图。
    
2. **挂载 C# 驱动脚本**：在场景中创建一个空物体（或直接使用你的目标模型），新建一个名为 `FurInstancedRenderer.cs` 的脚本并挂载。将文末的 C# 代码粘贴进去。
    
3. **配置参数**：
    
    - 在 C# 脚本面板中，将目标 `Mesh`（如狐狸、小熊的模型）和刚才创建的**毛发材质**拖入对应的槽位。
        
    - 调整 `Shell Count`（推荐 20 - 40 层）。
        
    - 开启材质的 **Enable GPU Instancing** 选项。
        
    - 运行游戏（或在 Scene 窗口中开启 `[ExecuteAlways]`），即可看到毛发效果。
        

> **核心提示**：本方案不需要目标物体有 Mesh Renderer 组件。C# 脚本会接管绘制流程，直接通过 GPU 提交渲染指令。

## 二、 核心架构：为何引入 GPU Instancing？

传统的 Shell Texturing 方案通常有两种实现方式：在 Shader 中写死数十个 Pass，或者在 DCC 软件里复制数十层拓扑结构相同的嵌套网格。这两种做法都会导致 Shader 变体爆炸或内存骤增。

本方案采用了更具现代管线思维的解法：**单 Pass Shader + GPU 实例化（GPU Instancing）**。

我们通过 C# 脚本调用 `Graphics.DrawMeshInstanced`，向 GPU 一次性提交数十次绘制指令，每次绘制复用同一个网格，仅通过 `MaterialPropertyBlock` 传入一个核心变量：**层级比率 (`_LayerRatio`)**。

- **_LayerRatio = 0**：贴合皮肤的最底层（发根）。
    
- **_LayerRatio = 1**：向外延伸的最外层（发梢）。
    

这种架构将 Draw Call 压榨到了 1 次（视 Instancing 批次限制而定），同时内存中只保留一份网格数据，达到了极致的性能优化。

## 三、 C# 脚本逻辑分析：CPU 与 GPU 的桥梁

C# 端的职责非常清晰：准备渲染数据，并按帧提交给 GPU。

- **准备矩阵数组**：毛发的每一层都处于同一个位置，因此我们创建一个长度为 `shellCount` 的 `Matrix4x4` 数组，所有元素都填充当前物体的 `localToWorldMatrix`。
    
- **分配层级比例**：我们计算出每一层的 `_LayerRatio`，将其存入 `float[]` 数组，并通过 `MaterialPropertyBlock.SetFloatArray` 传递给 Shader。
    
- **提交绘制**：利用 `Graphics.DrawMeshInstanced` 一键提交渲染。
    

## 四、 顶点着色器：形态构建与物理模拟

毛发的形态和动态需要在 Vertex Shader 中完成。核心逻辑是将原始模型的顶点，沿着法线方向逐层向外“挤出”，并叠加物理方向的形变。

### 1. 法线外扩 (Normal Extrusion)

最基础的壳层构建数学模型如下：

$P_{ext} = P_{os} + \vec{N} \cdot (L_{max} \cdot M_{length} \cdot Ratio)$

代码实现中，我们采样了长度遮罩 `_LengthMap`，结合全局长度 `_FurLength` 算出当前顶点的绝对长度，再乘以当前层级的 `ratio`。从最内层到最外层，网格就像气球一样层层膨胀。

### 2. 非线性刚度与重力下垂 (Non-linear Stiffness)

真实的毛发并不是直挺挺的钢针。为了让毛发受重力和梳毛方向的影响，且**发根坚硬，发梢柔软**，我们引入了非线性衰减（Non-linear Falloff）：

利用 $Ratio^2$ 构建物理刚度。发根处 $Ratio \approx 0$，几乎不发生弯曲；发梢处 $Ratio \approx 1$，偏移量达到最大。这就赋予了毛发自然服帖的重力感。

### 3. 凌乱随机扰动 (Messiness & Curl)

为了打破 GPU 完美平行计算带来的虚假感，我们使用 `sin()` 函数结合世界坐标产生不同相位的偏移，实现毛发的自然卷曲和杂乱感。

## 五、 片元着色器：Alpha Test 与裁剪造型

由于 Vertex Shader 只是将一堆气球般的网格套在一起，真正让这些外壳变成“发丝”的魔法发生在 Fragment Shader 的 **Alpha Test（透明度裁剪）** 阶段。

我们引入一张高频噪声图（Noise Map），利用 `_ThicknessCurve` 将 `_LayerRatio` 重新映射为裁剪阈值：

- **发根层**：阈值极低，仅剔除极黑区域，大部分像素保留，形成致密的底绒。
    
- **发梢层**：阈值极高，只有噪声图最白的局部像素被保留，形成尖锐的发丝。
    

通过 `clip()` 函数直接丢弃不满足条件的片元。基于多层网格的透视叠加，人眼会把这些保留下来的离散像素点脑补成一根根立体的毛发。

在光照上，我们采用最基础的 Lambertian 漫反射（$N \cdot L$）加上环境光。考虑到多层 Alpha Test 带来的极高 Overdraw（重绘率），这是为移动端性能做出的必要让步。

## 六、 完整项目源码

### 1. C# 驱动脚本 (FurInstancedRenderer.cs)

C#

```
using UnityEngine;

[ExecuteAlways]
public class FurInstancedRenderer : MonoBehaviour
{
    [Header("Render Assets")]
    public Mesh mesh;
    public Material furMaterial;

    [Header("Fur Settings")]
    [Range(1, 100)] 
    public int shellCount = 40;

    private Matrix4x4[] matrices;
    private float[] layerRatios;
    private MaterialPropertyBlock propertyBlock;

    void Update()
    {
        // 确保资源与参数有效
        if (mesh == null || furMaterial == null || shellCount <= 0) return;

        // 初始化或当层数改变时，重新分配内存
        if (matrices == null || matrices.Length != shellCount)
        {
            matrices = new Matrix4x4[shellCount];
            layerRatios = new float[shellCount];
            propertyBlock = new MaterialPropertyBlock();

            for (int i = 0; i < shellCount; i++)
            {
                // 计算当前层级的比例：0.0 (底层) -> 1.0 (最外层)
                layerRatios[i] = (float)i / (shellCount - 1);
            }
            // 将比例数组一次性传入材质属性块
            propertyBlock.SetFloatArray("_LayerRatio", layerRatios);
        }

        // 更新矩阵：所有层叠在同一位置，跟随当前物体的 Transform
        Matrix4x4 currentMatrix = transform.localToWorldMatrix;
        for (int i = 0; i < shellCount; i++)
        {
            matrices[i] = currentMatrix;
        }

        // 提交 Instancing 渲染请求 (限制: 单批次最大 1023 个实例)
        Graphics.DrawMeshInstanced(mesh, 0, furMaterial, matrices, shellCount, propertyBlock);
    }
}
```

### 2. URP 毛发着色器 (URP_StaticInstancedFur.shader)

High-level shader language

```
Shader "Custom/URP_StaticInstancedFur"
{
    Properties
    {
        [Header(Color Settings)]
        _BaseColor ("Base Color (Root)", Color) = (0.1, 0.05, 0.02, 1)
        _FurColor ("Fur Tip Color", Color) = (0.8, 0.4, 0.1, 1)
        
        [Header(Shape and Density)]
        _NoiseTex ("Fur Noise Mask", 2D) = "white" {}
        _NoiseTiling ("Noise Tiling (Global Multiplier)", Float) = 30.0
        _MaxDensity ("Global Max Density", Range(0.0, 1.0)) = 1.0 
        _Density ("Root Density (Base)", Range(0.0, 1.0)) = 1.0
        _TipCutoff ("Tip Thinness (Cutoff)", Range(0.0, 1.0)) = 0.9
        _ThicknessCurve ("Taper Curve (Root to Tip)", Range(0.1, 5.0)) = 1.5
        
        [Header(Fur Length)]
        _FurLength ("Max Fur Length (Global)", Float) = 0.2
        _LengthMap ("Length Mask (R Channel)", 2D) = "white" {} 
        
        [Header(Physics and Natural)]
        _Gravity ("Gravity / Droop", Range(0.0, 1.0)) = 0.2 
        _Messiness ("Messiness (Tangle/Curl)", Range(0.0, 1.0)) = 0.3
        _CombDir ("Comb Direction (X, Y, Z)", Vector) = (0.0, -0.5, 0.5, 0.0) 
    }
    SubShader
    {
        Tags { "RenderType"="TransparentCutout" "RenderPipeline"="UniversalPipeline" "Queue"="AlphaTest" }
        Cull Off 

        Pass
        {
            Name "UniversalForward"
            Tags { "LightMode"="UniversalForward" }

            HLSLPROGRAM
            #pragma target 4.5
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing // 必须开启以支持 GPU Instancing

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            struct Attributes
            {
                float4 positionOS   : POSITION;
                float3 normalOS     : NORMAL;
                float2 uv           : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Varyings
            {
                float4 positionCS   : SV_POSITION;
                float2 uv           : TEXCOORD0;
                float3 normalWS     : NORMAL;
                float layerRatio    : TEXCOORD1;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            TEXTURE2D(_NoiseTex); SAMPLER(sampler_NoiseTex);
            TEXTURE2D(_LengthMap); SAMPLER(sampler_LengthMap);

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseColor;
                float4 _FurColor;
                float4 _NoiseTex_ST; 
                float4 _LengthMap_ST; 
                float _NoiseTiling;
                float _MaxDensity;    
                float _Density;
                float _TipCutoff;   
                float _ThicknessCurve;
                float _FurLength;
                float _Gravity;
                float _Messiness;
                float4 _CombDir; 
            CBUFFER_END

            // 声明实例化属性池
            UNITY_INSTANCING_BUFFER_START(Props)
                UNITY_DEFINE_INSTANCED_PROP(float, _LayerRatio)
            UNITY_INSTANCING_BUFFER_END(Props)

            Varyings vert(Attributes input)
            {
                Varyings output = (Varyings)0;
                UNITY_SETUP_INSTANCE_ID(input);
                UNITY_TRANSFER_INSTANCE_ID(input, output);

                // 读取当前层的比例
                float ratio = UNITY_ACCESS_INSTANCED_PROP(Props, _LayerRatio);
                output.layerRatio = ratio;

                // 遮罩与长度计算
                float2 lengthUV = TRANSFORM_TEX(input.uv, _LengthMap);
                float lengthMask = SAMPLE_TEXTURE2D_LOD(_LengthMap, sampler_LengthMap, lengthUV, 0).r;
                float actualFurLength = _FurLength * lengthMask;

                float3 posWS = TransformObjectToWorld(input.positionOS.xyz);
                float3 normalWS = TransformObjectToWorldDir(input.normalOS);

                // 1. 法线向外推移
                float extrusion = ratio * actualFurLength;
                posWS += normalWS * extrusion;
                
                // 2. 物理与梳毛方向 (二次曲线实现非线性刚度)
                float3 baseDrop = float3(0, -1, 0) * _Gravity; 
                float3 combOffset = _CombDir.xyz + baseDrop;
                float3 dirOffset = combOffset * actualFurLength * (ratio * ratio);
                posWS += dirOffset;

                // 3. 凌乱与卷曲扰动
                float3 messOffset = sin(posWS * 30.0) * (_Messiness * actualFurLength * 0.5) * (ratio * ratio);
                posWS += messOffset;

                output.positionCS = TransformWorldToHClip(posWS);
                output.uv = TRANSFORM_TEX(input.uv, _NoiseTex); 
                // 混合重力与卷曲后的新法线
                output.normalWS = normalize(normalWS + dirOffset + messOffset);

                return output;
            }

            float4 frag(Varyings input) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(input);
                float ratio = input.layerRatio;

                // 配合网格形变的 UV 扰动增强乱发感
                float2 uvOffset = sin(input.uv * 50.0) * (_Messiness * 0.02 * ratio);
                float2 noiseUV = (input.uv + uvOffset) * _NoiseTiling; 
                float noiseVal = SAMPLE_TEXTURE2D(_NoiseTex, sampler_NoiseTex, noiseUV).r;
                
                float shapeCurve = pow(ratio, _ThicknessCurve);
                
                // 计算 Alpha Test 的阈值
                float rootCutoff = 1.0 - _Density;
                float cutoff = lerp(rootCutoff, _TipCutoff, shapeCurve);
                
                // 防夹断保护
                float minCutoffLimit = 1.0 - _MaxDensity;
                cutoff = max(cutoff, minCutoffLimit);
                
                // 强制保留最底层，防止走光
                if (ratio > 0.02)
                {
                    clip(noiseVal - cutoff); // 多层裁剪核心
                }

                // 光照计算
                float3 albedo = lerp(_BaseColor.rgb, _FurColor.rgb, ratio);
                Light mainLight = GetMainLight();
                float NdotL = saturate(dot(normalize(input.normalWS), normalize(mainLight.direction)));
                
                float3 ambient = float3(0.2, 0.25, 0.3) * albedo; 
                float3 finalColor = albedo * mainLight.color * NdotL + ambient;

                return float4(finalColor, 1.0);
            }
            ENDHLSL
        }
    }
}
```

## 七、 进阶与性能优化建议

使用多层 Shell Texturing 时，极高的 Overdraw（重绘率）是你必须面对的敌人。在投入实际商业项目前，建议搭配以下策略：

1. **动态 LOD 控制**：在 C# 脚本中检测相机与物体的距离。距离越远，向 `DrawMeshInstanced` 提交的 `shellCount` 越少，远景甚至可以降至 3 - 5 层，这能挽救移动端发热降频的问题。
    
2. **Depth Pre-pass**：如果你的场景出现了严重的毛发交叠，可以考虑在 Shader 中额外增加一个只写入深度但不输出颜色的 Depth Only Pass，利用硬件的 Early-Z 技术提前剔除掉被遮挡的片元。
    
3. **风力流场接入**：可以将简单的常量 `_CombDir` 升级为采样全局风力噪声纹理，就能轻松实现微风拂过大片绒毛（或草地）的波浪效果。