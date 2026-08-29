## 1. 核心实现思路

本 Shader 采用 **Shell Texturing（多层外壳毛发技术）** 结合 **GPU Instancing（GPU 实例化）** 实现。

- **多层外壳 (Shell Texturing):** 像剥洋葱一样，将同一个模型向外扩展渲染很多层（通常 20-60 层）。底层不透明，越往外层透明度越高（被裁剪的区域越大），通过视觉堆叠欺骗眼睛，形成一根根立体的毛发。
    
- **GPU 实例化 (GPU Instancing):** 传统多层毛发需要多个 Pass 或生成庞大的网格数据。本方案通过外部 C# 脚本调用 `DrawMeshInstanced`，一次性向 GPU 提交多次绘制请求。Shader 内部只需一个 Pass，通过读取传入的 `_LayerRatio`（层级比例，0 为根部，1 为尖端）来决定当前层的位置和形态，极大节省了 Draw Call 和内存。
    

## 2. 框架逻辑解析

### 数据流与属性 (Properties & Buffer)

- **材质属性:** 定义了基础色、毛发尖端色、密度、长度、重力、梳毛方向等宏观控制参数。
    
- **实例化属性 (Props):** 核心变量 `_LayerRatio` 定义在 `UNITY_INSTANCING_BUFFER_START` 中，确保每次实例绘制都能拿到属于自己这一层的比例进度。
    

### 顶点着色器 (Vertex Shader) - 形态构建

处理每一层网格的位置偏移与物理形变。形变程度随 `_LayerRatio` 增加而增大。

1. **法线外扩:** 采样长度遮罩贴图（`_LengthMap`），沿着顶点法线方向 (`normalWS`) 将顶点向外推移。
    
2. **方向与重力 (物理模拟):** 将重力向量与自定义的“梳毛方向”结合，对顶点施加偏移。
    
3. **凌乱卷曲:** 使用世界坐标结合正弦函数 `sin()` 制造随机扰动，打破毛发绝对笔直的虚假感。
    

### 片元着色器 (Fragment Shader) - 细节裁剪与着色

决定毛发的粗细渐变和光照颜色。

1. **形状裁剪 (Alpha Clip):** 核心步骤。对高频噪声贴图 (`_NoiseTex`) 进行采样。根部保留大部分像素，尖端只保留极少部分像素。利用 `clip()` 剔除不需要的像素，形成“发丝”感。
    
2. **颜色渐变:** 颜色从根部 `_BaseColor` 根据 `ratio` 线性插值 (`lerp`) 到尖端 `_FurColor`。
    
3. **基础光照:** 计算漫反射与环境光叠加，输出最终颜色。
    

## 3. 涉及的数学与图形学知识

- **Shell Texturing (多层外壳):** 经典的毛发/草地实时渲染方案，非常适合处理短毛和动物绒毛。
    
- **GPU Instancing:** 计算机图形学中用于减少 CPU 与 GPU 之间通信开销（Draw Call）的技术。通过一次提交渲染相同网格的多个实例。
    
- **法线方向挤出 (Normal Extrusion):** 顶点沿着法线向量 `N` 移动一段距离 `d`。公式为：`P' = P + N * d`。
    
- **非线性衰减 (二次曲线物理刚度):** 代码中使用了 `ratio * ratio` 来控制重力和弯曲的影响。这在图形学物理模拟中常用来表现物体的“刚度”。根部 (ratio 接近 0) 受影响极小（坚挺），尖端 (ratio 接近 1) 受影响最大（柔软弯曲）。
    
- **Alpha Test / 裁剪 (Clip):** HLSL 中的 `clip(x)` 指令。当 `x < 0` 时，直接在管线中丢弃该片元，不进行后续的颜色混合。
    
- **兰伯特漫反射光照 (Lambertian Shading):** 代码 `dot(normalize(normal), normalize(lightDir))` 利用点积（Dot Product）计算光线与法线的夹角余弦值，从而得到基础的明暗变化。
    
- **正弦扰动 (Sine Perturbation):** 利用 `sin(posWS * frequency) * amplitude` 作为低成本的伪随机发生器，用于生成毛发的凌乱感和 UV 扰动。
    

## 4. 着色器源码

High-level shader language

```hlsl
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
        // 新增：梳毛方向（用于打破刺猬般的整齐排列）
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
            #pragma multi_compile_instancing

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
                float4 _CombDir; // 声明梳毛方向变量
            CBUFFER_END

            UNITY_INSTANCING_BUFFER_START(Props)
                UNITY_DEFINE_INSTANCED_PROP(float, _LayerRatio)
            UNITY_INSTANCING_BUFFER_END(Props)

            Varyings vert(Attributes input)
            {
                Varyings output = (Varyings)0;
                UNITY_SETUP_INSTANCE_ID(input);
                UNITY_TRANSFER_INSTANCE_ID(input, output);

                float ratio = UNITY_ACCESS_INSTANCED_PROP(Props, _LayerRatio);
                output.layerRatio = ratio;

                // 采样长度遮罩贴图
                float2 lengthUV = TRANSFORM_TEX(input.uv, _LengthMap);
                float lengthMask = SAMPLE_TEXTURE2D_LOD(_LengthMap, sampler_LengthMap, lengthUV, 0).r;
                float actualFurLength = _FurLength * lengthMask;

                float3 posWS = TransformObjectToWorld(input.positionOS.xyz);
                float3 normalWS = TransformObjectToWorldDir(input.normalOS);

                // 1. 基础外扩
                float extrusion = ratio * actualFurLength;
                posWS += normalWS * extrusion;
                
                // 2. 梳毛方向与重力融合
                float3 baseDrop = float3(0, -1, 0) * _Gravity; 
                float3 combOffset = _CombDir.xyz + baseDrop;
                float3 dirOffset = combOffset * actualFurLength * (ratio * ratio);
                posWS += dirOffset;

                // 3. 凌乱感卷曲
                float3 messOffset = sin(posWS * 30.0) * (_Messiness * actualFurLength * 0.5) * (ratio * ratio);
                posWS += messOffset;

                output.positionCS = TransformWorldToHClip(posWS);
                
                output.uv = TRANSFORM_TEX(input.uv, _NoiseTex); 
                output.normalWS = normalize(normalWS + dirOffset + messOffset);

                return output;
            }

            float4 frag(Varyings input) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(input);
                float ratio = input.layerRatio;

                // 凌乱感带来的 UV 扰动
                float2 uvOffset = sin(input.uv * 50.0) * (_Messiness * 0.02 * ratio);
                float2 noiseUV = (input.uv + uvOffset) * _NoiseTiling; 
                
                float noiseVal = SAMPLE_TEXTURE2D(_NoiseTex, sampler_NoiseTex, noiseUV).r;
                
                float shapeCurve = pow(ratio, _ThicknessCurve);
                
                // 计算透明裁剪阈值
                float rootCutoff = 1.0 - _Density;
                float cutoff = lerp(rootCutoff, _TipCutoff, shapeCurve);
                
                // 限制最大密度
                float minCutoffLimit = 1.0 - _MaxDensity;
                cutoff = max(cutoff, minCutoffLimit);
                
                if (ratio > 0.02)
                {
                    clip(noiseVal - cutoff);
                }

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