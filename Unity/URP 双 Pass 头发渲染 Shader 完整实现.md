---
title: URP 双 Pass 头发渲染 Shader 完整实现
date: 2026-08-20
tags:
  - Unity
  - URP
  - Shader
  - Hair
  - Rendering
  - KajiyaKay
  - Marschner
---

# URP 双 Pass 头发渲染 Shader 完整实现

基于《基于 URP 的头发渲染技术解析》的思路，推导出的可直接落地的完整代码。

**技术栈：** 双 Pass 分层（不透明 + 半透明）· 混合法线（Sphere Normal 走 Diffuse / Bitangent 走 Specular）· Kajiya-Kay 双层天使环 + Marschner TT 透射。

**依赖：** URP 14+（Unity 2022.3）；Unity 6 的 RenderGraph 路径已在 Render Feature 中分支处理。

> [!info] 相关笔记
> [[Unity 渲染管线更改：Built-in 至 URP]] · [[六类Shader写法差异与注意点]] · [[空间转换]]

---

## 0. 整体思路回顾

插片头发的核心矛盾：**半透明需要关深度写入，但关了深度写入就没有正确遮挡**。

双 Pass 的解法是把这两件事拆开：

| | Pass 1 `Hair_Opaque` | Pass 2 `Hair_Transparent` |
|---|---|---|
| 职责 | 头发"实心"部分 | 发梢、边缘的软过渡 |
| ZWrite | On | Off |
| ZTest | LEqual | **Less** |
| Cutoff | 0.5（高阈值） | 0.01（仅剔全透明） |
| Blend | One Zero | SrcAlpha OneMinusSrcAlpha |
| 调度 | URP 自动（UniversalForward） | **需要 Render Feature 手动跑** |

`ZTest Less` 是整套方案的题眼：Pass 1 已经把实心像素的深度写进去了，Pass 2 再画到同一位置时深度**相等**，`Less` 判定失败 → 直接被拒绝。于是 Pass 2 只会落在 Pass 1 因 alpha 不足而 clip 掉的区域（背后是背景或更远的物体）。既不重复着色实心部分，又保住了软边。

> [!note] 关于锯齿
> 一开始发现头发边缘锯齿严重，第一反应去提分辨率、开 TAA/MSAA —— 方向错了。插片头发的锯齿本质是 **Alpha Test 的硬切边**，属于几何/着色层面的问题，抗锯齿只能缓解不能根治。真正的解法就是这里的 Pass 2：用一层 alpha blend 把切边"糊"开。

---

## 1. Shader 本体

`Assets/Shaders/Hair/HairDualPass.shader`

```hlsl
Shader "Custom/Hair/HairDualPass"
{
    Properties
    {
        [Header(Base)][Space(4)]
        _BaseMap            ("Base Map (RGB) Opacity(A)", 2D) = "white" {}
        _BaseColor          ("Base Color", Color) = (1,1,1,1)
        _AlphaMask          ("Opacity Mask (R) 半透黑白遮罩", 2D) = "white" {}
        _AlphaCutoff        ("Opaque Cutoff (Pass1)", Range(0,1)) = 0.5
        _TransparentCutoff  ("Transparent Cutoff (Pass2)", Range(0,1)) = 0.01
        _AlphaScale         ("Alpha Scale", Range(0,2)) = 1.0
        _AOMap              ("AO / Root-Tip Map (R)", 2D) = "white" {}
        _AOStrength         ("AO Strength", Range(0,1)) = 1.0

        [Header(Sphere Normal   Diffuse)][Space(4)]
        [Toggle(_SPHERENORMAL_RADIAL)] _UseRadialSphere ("使用径向球法线(稳定版)", Float) = 0
        _SphereCenterOS     ("Sphere Center (Object Space)", Vector) = (0,0,0,0)
        _ThicknessScale     ("Thickness Scale (球的胖瘦)", Range(0,2)) = 1.0
        _ThicknessBias      ("Thickness Bias (朝上分量)", Range(-1,2)) = 0.3
        _NormalBlend        ("Geometry -> Sphere 混合", Range(0,1)) = 1.0
        _DiffuseWrap        ("Diffuse Wrap (半兰伯特)", Range(0,1)) = 0.3

        [Header(Specular Shift)][Space(4)]
        _ShiftTex           ("Shift Noise (R)", 2D) = "gray" {}
        _PrimaryShift       ("Primary Shift", Range(-1,1)) = 0.05
        _SecondaryShift     ("Secondary Shift", Range(-1,1)) = -0.06
        [Toggle(_STRAND_USE_TANGENT)] _StrandUseTangent ("发丝流向用 Tangent (默认 Bitangent)", Float) = 0

        [Header(Spec1   Kajiya Kay   R Lobe)][Space(4)]
        _PrimaryColor       ("Primary Color", Color) = (1,1,1,1)
        _PrimaryPower       ("Primary Power", Range(1,512)) = 96
        _PrimaryScale       ("Primary Scale", Range(0,4)) = 1.0

        [Header(Spec2   Kajiya Kay   TRT Lobe)][Space(4)]
        _SecondaryColor     ("Secondary Color", Color) = (1,0.75,0.45,1)
        _SecondaryPower     ("Secondary Power", Range(1,512)) = 20
        _SecondaryScale     ("Secondary Scale", Range(0,4)) = 0.6
        _SecondaryTint      ("Spec2 染发色权重", Range(0,1)) = 0.6

        [Header(Marschner   TT Lobe   透射)][Space(4)]
        _MarschnerColor     ("Marschner Color", Color) = (1,0.42,0.22,1)
        _MarschnerPower     ("Marschner Power", Range(1,128)) = 12
        _MarschnerScale     ("Marschner Scale", Range(0,4)) = 0.5
        _MarschnerWrap      ("Back Light Wrap", Range(0.01,1)) = 0.5
        _MarschnerShadow    ("透射受阴影影响程度", Range(0,1)) = 0.3

        [Header(Ambient)][Space(4)]
        _EnvStrength        ("SH / Ambient Strength", Range(0,2)) = 1.0

        [Header(Render State)][Space(4)]
        [Enum(UnityEngine.Rendering.CullMode)] _Cull ("Cull", Float) = 2  // 2 = Back
    }

    SubShader
    {
        // 队列放在 AlphaTest：Pass1 跟着不透明物体一起渲染并写深度
        Tags
        {
            "RenderType"        = "TransparentCutout"
            "Queue"             = "AlphaTest"
            "RenderPipeline"    = "UniversalPipeline"
            "IgnoreProjector"   = "True"
        }

        // ============================================================
        //  公共代码块：结构体 / 属性 / 光照模型 / ShadeHair
        // ============================================================
        HLSLINCLUDE
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Shadows.hlsl"

        CBUFFER_START(UnityPerMaterial)
            float4 _BaseMap_ST;
            float4 _ShiftTex_ST;
            float4 _SphereCenterOS;
            half4  _BaseColor;
            half4  _PrimaryColor;
            half4  _SecondaryColor;
            half4  _MarschnerColor;
            half   _AlphaCutoff;
            half   _TransparentCutoff;
            half   _AlphaScale;
            half   _AOStrength;
            half   _ThicknessScale;
            half   _ThicknessBias;
            half   _NormalBlend;
            half   _DiffuseWrap;
            half   _PrimaryShift;
            half   _SecondaryShift;
            half   _PrimaryPower;
            half   _PrimaryScale;
            half   _SecondaryPower;
            half   _SecondaryScale;
            half   _SecondaryTint;
            half   _MarschnerPower;
            half   _MarschnerScale;
            half   _MarschnerWrap;
            half   _MarschnerShadow;
            half   _EnvStrength;
            half   _Cull;
        CBUFFER_END

        TEXTURE2D(_BaseMap);    SAMPLER(sampler_BaseMap);
        TEXTURE2D(_AlphaMask);  SAMPLER(sampler_AlphaMask);
        TEXTURE2D(_AOMap);      SAMPLER(sampler_AOMap);
        TEXTURE2D(_ShiftTex);   SAMPLER(sampler_ShiftTex);

        #define HAIR_TWO_PI 6.28318530718

        struct Attributes
        {
            float4 positionOS : POSITION;
            float3 normalOS   : NORMAL;
            float4 tangentOS  : TANGENT;
            float2 uv         : TEXCOORD0;
            UNITY_VERTEX_INPUT_INSTANCE_ID
        };

        struct Varyings
        {
            float4 positionCS     : SV_POSITION;
            float4 uv             : TEXCOORD0;   // xy = base, zw = shift
            float3 positionWS     : TEXCOORD1;
            half3  normalWS       : TEXCOORD2;   // 几何法线：用于高光偏移 / 背光判定
            half3  tangentWS      : TEXCOORD3;
            half3  bitangentWS    : TEXCOORD4;   // 默认发丝流向
            half3  sphereNormalWS : TEXCOORD5;   // 球形法线：只喂给 Diffuse
            half   fogFactor      : TEXCOORD6;
        #if defined(REQUIRES_VERTEX_SHADOW_COORD_INTERPOLATOR)
            float4 shadowCoord    : TEXCOORD7;
        #endif
            UNITY_VERTEX_INPUT_INSTANCE_ID
            UNITY_VERTEX_OUTPUT_STEREO
        };

        struct HairFragmentOutput
        {
            half3 color;
            half  alpha;
        };

        // ------------------------------------------------------------
        //  球形法线：让漫反射像照在一个光滑球体上，抹掉发丝级噪点
        // ------------------------------------------------------------
        half3 ComputeSphereNormal(float3 positionWS, float3 positionOS, float2 uv, half3 normalWS)
        {
        #if defined(_SPHERENORMAL_RADIAL)
            // 稳定版：以头部中心为球心的径向法线，角色移动/旋转时不会漂
            float3 dir = positionOS - _SphereCenterOS.xyz;
            dir.xz *= _ThicknessScale;
            dir.y  += _ThicknessBias;
            half3 sphereNorm = TransformObjectToWorldNormal(normalize(dir));
        #else
            // 原文版：用世界坐标 + UV 构造径向向量
            float3 n = float3(
                sin(positionWS.x + uv.x * HAIR_TWO_PI) * _ThicknessScale,
                _ThicknessBias + 0.2,                        // 略微向上，模拟头顶隆起
                cos(positionWS.z + uv.y * HAIR_TWO_PI) * _ThicknessScale
            );
            half3 sphereNorm = normalize(n);
        #endif
            // 保留一点几何法线，避免完全丢失造型信息
            return normalize(lerp(normalWS, sphereNorm, _NormalBlend));
        }

        // ------------------------------------------------------------
        //  高光偏移：把切线沿法线推一点，两层错开形成双光环
        // ------------------------------------------------------------
        half3 ShiftTangent(half3 T, half3 N, half shift)
        {
            return normalize(T + shift * N);
        }

        // ------------------------------------------------------------
        //  Kajiya-Kay：T·H 越接近 0（垂直）高光越强 -> 一圈天使环
        // ------------------------------------------------------------
        half KajiyaKay(half3 T, half3 V, half3 L, half power)
        {
            half3 H     = SafeNormalize(L + V);
            half  dotTH = dot(T, H);
            half  sinTH = sqrt(max(0.0h, 1.0h - dotTH * dotTH));
            // dirAtten：抑制光源在发丝背侧时的假高光（原文省略，建议保留）
            half  dirAtten = smoothstep(-1.0h, 0.0h, dotTH);
            return dirAtten * pow(max(0.0h, sinTH), power);
        }

        // ------------------------------------------------------------
        //  Marschner TT（简化）：光穿透发丝射出，逆光时的通透感
        //  与 KK 的区别：半程向量用 (V - L)，且叠一个背光可见性
        // ------------------------------------------------------------
        half MarschnerTT(half3 T, half3 V, half3 L, half3 N, half power)
        {
            half3 H     = SafeNormalize(V - L);              // TT 波瓣在光的另一侧
            half  dotTH = dot(T, H);
            half  sinTH = sqrt(max(0.0h, 1.0h - dotTH * dotTH));
            half  spec  = pow(max(0.0h, sinTH), power * 0.8h) * 0.5h;

            // 只有逆光（N·L < 0）时透射才明显
            half backLit = saturate((-dot(N, L) + _MarschnerWrap) / (1.0h + _MarschnerWrap));
            return spec * backLit;
        }

        // ------------------------------------------------------------
        //  单光源着色
        // ------------------------------------------------------------
        half3 ShadeHairSingleLight(Light light, half3 albedo, half aoFactor,
                                   half3 diffuseNormal, half3 geometryNormal,
                                   half3 t1, half3 t2, half3 V)
        {
            half3 L      = light.direction;
            half  shadow = light.shadowAttenuation;
            half  atten  = light.distanceAttenuation * shadow;

            // ---- Diffuse：球形法线 + 可选 wrap ----
            half NdotL = saturate((dot(diffuseNormal, L) + _DiffuseWrap) / (1.0h + _DiffuseWrap));
            half3 diffuseTotal = albedo * NdotL * aoFactor;

            // ---- Specular ----
            half spec1 = KajiyaKay(t1, V, L, _PrimaryPower)   * _PrimaryScale;
            // 次级高光落在受光面才合理，乘 NdotL 抑制暗面浮光
            half spec2 = KajiyaKay(t2, V, L, _SecondaryPower) * _SecondaryScale * NdotL;
            half mars  = MarschnerTT(t2, V, L, geometryNormal, _MarschnerPower) * _MarschnerScale;

            half3 spec2Col = lerp(_SecondaryColor.rgb, _SecondaryColor.rgb * albedo, _SecondaryTint);
            half3 specTotal = (spec1 * _PrimaryColor.rgb)
                            + (spec2 * spec2Col)
                            + (mars  * _MarschnerColor.rgb * albedo);

            specTotal *= aoFactor;

            // 透射项不该被阴影完全掐死（光本来就是穿过去的）
            half3 marsPart  = (mars * _MarschnerColor.rgb * albedo * aoFactor);
            half3 marsAtten = light.distanceAttenuation * lerp(1.0h, shadow, _MarschnerShadow);

            half3 baseLit = (diffuseTotal + specTotal - marsPart) * atten;
            return (baseLit + marsPart * marsAtten) * light.color;
        }

        // ------------------------------------------------------------
        //  ShadeHair：两个 Pass 共用的完整着色
        // ------------------------------------------------------------
        HairFragmentOutput ShadeHair(Varyings input, half facing)
        {
            HairFragmentOutput o = (HairFragmentOutput)0;

            half4 baseTex  = SAMPLE_TEXTURE2D(_BaseMap,   sampler_BaseMap,   input.uv.xy);
            half  maskTex  = SAMPLE_TEXTURE2D(_AlphaMask, sampler_AlphaMask, input.uv.xy).r;
            half  aoTex    = SAMPLE_TEXTURE2D(_AOMap,     sampler_AOMap,     input.uv.xy).r;
            half  shiftTex = SAMPLE_TEXTURE2D(_ShiftTex,  sampler_ShiftTex,  input.uv.zw).r;

            half3 albedo   = baseTex.rgb * _BaseColor.rgb;
            half  alpha    = saturate(baseTex.a * maskTex * _BaseColor.a * _AlphaScale);
            half  aoFactor = lerp(1.0h, aoTex, _AOStrength);
            half  shiftVal = shiftTex - 0.5h;                 // 噪声映射到 [-0.5, 0.5]

            // 双面片：背面翻转法线
            half3 geometryNormal = normalize(input.normalWS) * facing;
            half3 diffuseNormal  = normalize(input.sphereNormalWS) * facing;

        #if defined(_STRAND_USE_TANGENT)
            half3 strandDir = normalize(input.tangentWS);
        #else
            half3 strandDir = normalize(input.bitangentWS);   // 发丝流向（V 沿发丝时用它）
        #endif

            half3 V = normalize(GetWorldSpaceViewDir(input.positionWS));

            // 两层错开的切线
            half3 t1 = ShiftTangent(strandDir, geometryNormal, shiftVal + _PrimaryShift);
            half3 t2 = ShiftTangent(strandDir, geometryNormal, shiftVal + _SecondaryShift);

            // ---- 主光 ----
        #if defined(REQUIRES_VERTEX_SHADOW_COORD_INTERPOLATOR)
            float4 shadowCoord = input.shadowCoord;
        #elif defined(MAIN_LIGHT_CALCULATE_SHADOWS)
            float4 shadowCoord = TransformWorldToShadowCoord(input.positionWS);
        #else
            float4 shadowCoord = float4(0, 0, 0, 0);
        #endif

            Light mainLight = GetMainLight(shadowCoord, input.positionWS, half4(1, 1, 1, 1));
            half3 color = ShadeHairSingleLight(mainLight, albedo, aoFactor,
                                               diffuseNormal, geometryNormal, t1, t2, V);

            // ---- 附加光 ----
        #if defined(_ADDITIONAL_LIGHTS)
            InputData inputData          = (InputData)0;
            inputData.positionWS         = input.positionWS;
            inputData.normalWS           = diffuseNormal;
            inputData.viewDirectionWS    = V;
            inputData.shadowCoord        = shadowCoord;
            inputData.normalizedScreenSpaceUV = GetNormalizedScreenSpaceUV(input.positionCS);

            uint pixelLightCount = GetAdditionalLightsCount();
            LIGHT_LOOP_BEGIN(pixelLightCount)
                Light addLight = GetAdditionalLight(lightIndex, input.positionWS, half4(1, 1, 1, 1));
                color += ShadeHairSingleLight(addLight, albedo, aoFactor,
                                              diffuseNormal, geometryNormal, t1, t2, V);
            LIGHT_LOOP_END
        #endif

            // ---- 环境光：同样走球形法线，保持柔和 ----
            color += SampleSH(diffuseNormal) * albedo * aoFactor * _EnvStrength;

            o.color = color;
            o.alpha = alpha;
            return o;
        }

        // ------------------------------------------------------------
        //  共用顶点着色器
        // ------------------------------------------------------------
        Varyings HairVertex(Attributes input)
        {
            Varyings output = (Varyings)0;
            UNITY_SETUP_INSTANCE_ID(input);
            UNITY_TRANSFER_INSTANCE_ID(input, output);
            UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(output);

            VertexPositionInputs posInputs  = GetVertexPositionInputs(input.positionOS.xyz);
            VertexNormalInputs   normInputs = GetVertexNormalInputs(input.normalOS, input.tangentOS);

            output.positionCS  = posInputs.positionCS;
            output.positionWS  = posInputs.positionWS;
            output.uv.xy       = TRANSFORM_TEX(input.uv, _BaseMap);
            output.uv.zw       = TRANSFORM_TEX(input.uv, _ShiftTex);

            output.normalWS    = normInputs.normalWS;
            output.tangentWS   = normInputs.tangentWS;
            output.bitangentWS = normInputs.bitangentWS;   // 发丝流向

            // 球形法线在顶点阶段算，插值后天然平滑
            output.sphereNormalWS = ComputeSphereNormal(posInputs.positionWS,
                                                        input.positionOS.xyz,
                                                        input.uv,
                                                        normInputs.normalWS);

            output.fogFactor = ComputeFogFactor(posInputs.positionCS.z);
        #if defined(REQUIRES_VERTEX_SHADOW_COORD_INTERPOLATOR)
            output.shadowCoord = GetShadowCoord(posInputs);
        #endif
            return output;
        }
        ENDHLSL

        // ============================================================
        //  Pass 1 — 不透明层：写深度，建立正确的遮挡关系
        // ============================================================
        Pass
        {
            Name "Hair_Opaque"
            Tags { "LightMode" = "UniversalForward" }

            Cull  [_Cull]
            ZWrite On
            ZTest  LEqual
            Blend  One Zero

            HLSLPROGRAM
            #pragma target 3.0
            #pragma vertex   HairVertex
            #pragma fragment HairFragOpaque

            #pragma shader_feature_local          _SPHERENORMAL_RADIAL
            #pragma shader_feature_local_fragment _STRAND_USE_TANGENT

            #pragma multi_compile _ _MAIN_LIGHT_SHADOWS _MAIN_LIGHT_SHADOWS_CASCADE _MAIN_LIGHT_SHADOWS_SCREEN
            #pragma multi_compile _ _ADDITIONAL_LIGHTS_VERTEX _ADDITIONAL_LIGHTS
            #pragma multi_compile_fragment _ _ADDITIONAL_LIGHT_SHADOWS
            #pragma multi_compile_fragment _ _SHADOWS_SOFT
            #pragma multi_compile_fragment _ _SCREEN_SPACE_OCCLUSION
            #pragma multi_compile_fog
            // URP 14/15/16 用这行；URP 17(Unity 6) 换成 _CLUSTER_LIGHT_LOOP
            #pragma multi_compile_fragment _ _FORWARD_PLUS
            #pragma multi_compile_instancing

            half4 HairFragOpaque(Varyings input, FRONT_FACE_TYPE cullFace : FRONT_FACE_SEMANTIC) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(input);
                half facing = IS_FRONT_VFACE(cullFace, 1.0h, -1.0h);

                HairFragmentOutput o = ShadeHair(input, facing);
                clip(o.alpha - _AlphaCutoff);        // 剔除半透明，只留实心

                half3 color = MixFog(o.color, input.fogFactor);
                return half4(color, 1.0h);
            }
            ENDHLSL
        }

        // ============================================================
        //  Pass 2 — 半透明层：发梢 / 边缘
        //  ZTest Less 是关键：Pass1 已写深度的像素深度相等 -> 直接被拒绝，
        //  只有比深度缓冲更近的（背景上的发梢）才会画出来。
        //  注意：这个 Pass 必须靠 Render Feature 单独调度（见第 2 节）
        // ============================================================
        Pass
        {
            Name "Hair_Transparent"
            Tags { "LightMode" = "HairTransparent" }

            Cull  [_Cull]
            ZWrite Off
            ZTest  Less
            Blend  SrcAlpha OneMinusSrcAlpha, One OneMinusSrcAlpha

            HLSLPROGRAM
            #pragma target 3.0
            #pragma vertex   HairVertex
            #pragma fragment HairFragTransparent

            #pragma shader_feature_local          _SPHERENORMAL_RADIAL
            #pragma shader_feature_local_fragment _STRAND_USE_TANGENT

            #pragma multi_compile _ _MAIN_LIGHT_SHADOWS _MAIN_LIGHT_SHADOWS_CASCADE _MAIN_LIGHT_SHADOWS_SCREEN
            #pragma multi_compile _ _ADDITIONAL_LIGHTS_VERTEX _ADDITIONAL_LIGHTS
            #pragma multi_compile_fragment _ _ADDITIONAL_LIGHT_SHADOWS
            #pragma multi_compile_fragment _ _SHADOWS_SOFT
            #pragma multi_compile_fog
            #pragma multi_compile_fragment _ _FORWARD_PLUS
            #pragma multi_compile_instancing

            half4 HairFragTransparent(Varyings input, FRONT_FACE_TYPE cullFace : FRONT_FACE_SEMANTIC) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(input);
                half facing = IS_FRONT_VFACE(cullFace, 1.0h, -1.0h);

                HairFragmentOutput o = ShadeHair(input, facing);
                clip(o.alpha - _TransparentCutoff);   // 仅剔除完全透明
                // 实心部分不需要在这里再算一遍：ZTest Less 会因深度相等而拒绝它

                half3 color = MixFog(o.color, input.fogFactor);
                return half4(color, o.alpha);
            }
            ENDHLSL
        }

        // ============================================================
        //  ShadowCaster / DepthOnly
        //  一律用 Pass1 的 _AlphaCutoff，保证影子与深度和实心层一致
        // ============================================================
        Pass
        {
            Name "ShadowCaster"
            Tags { "LightMode" = "ShadowCaster" }

            ZWrite On
            ZTest  LEqual
            ColorMask 0
            Cull [_Cull]

            HLSLPROGRAM
            #pragma target 3.0
            #pragma vertex   ShadowVert
            #pragma fragment ShadowFrag
            #pragma multi_compile_vertex _ _CASTING_PUNCTUAL_LIGHT_SHADOW
            #pragma multi_compile_instancing

            float3 _LightDirection;
            float3 _LightPosition;

            struct SAttributes { float4 positionOS : POSITION; float3 normalOS : NORMAL; float2 uv : TEXCOORD0; UNITY_VERTEX_INPUT_INSTANCE_ID };
            struct SVaryings   { float4 positionCS : SV_POSITION; float2 uv : TEXCOORD0; UNITY_VERTEX_INPUT_INSTANCE_ID };

            SVaryings ShadowVert(SAttributes input)
            {
                SVaryings output = (SVaryings)0;
                UNITY_SETUP_INSTANCE_ID(input);
                UNITY_TRANSFER_INSTANCE_ID(input, output);

                float3 positionWS = TransformObjectToWorld(input.positionOS.xyz);
                float3 normalWS   = TransformObjectToWorldNormal(input.normalOS);
            #if _CASTING_PUNCTUAL_LIGHT_SHADOW
                float3 lightDirWS = normalize(_LightPosition - positionWS);
            #else
                float3 lightDirWS = _LightDirection;
            #endif
                float4 positionCS = TransformWorldToHClip(ApplyShadowBias(positionWS, normalWS, lightDirWS));
            #if UNITY_REVERSED_Z
                positionCS.z = min(positionCS.z, UNITY_NEAR_CLIP_VALUE);
            #else
                positionCS.z = max(positionCS.z, UNITY_NEAR_CLIP_VALUE);
            #endif
                output.positionCS = positionCS;
                output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
                return output;
            }

            half4 ShadowFrag(SVaryings input) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(input);
                half a = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv).a
                       * SAMPLE_TEXTURE2D(_AlphaMask, sampler_AlphaMask, input.uv).r
                       * _BaseColor.a * _AlphaScale;
                clip(a - _AlphaCutoff);
                return 0;
            }
            ENDHLSL
        }

        Pass
        {
            Name "DepthOnly"
            Tags { "LightMode" = "DepthOnly" }

            ZWrite On
            ColorMask R
            Cull [_Cull]

            HLSLPROGRAM
            #pragma target 3.0
            #pragma vertex   DepthVert
            #pragma fragment DepthFrag
            #pragma multi_compile_instancing

            struct DAttributes { float4 positionOS : POSITION; float2 uv : TEXCOORD0; UNITY_VERTEX_INPUT_INSTANCE_ID };
            struct DVaryings   { float4 positionCS : SV_POSITION; float2 uv : TEXCOORD0; UNITY_VERTEX_INPUT_INSTANCE_ID };

            DVaryings DepthVert(DAttributes input)
            {
                DVaryings output = (DVaryings)0;
                UNITY_SETUP_INSTANCE_ID(input);
                UNITY_TRANSFER_INSTANCE_ID(input, output);
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
                return output;
            }

            half4 DepthFrag(DVaryings input) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(input);
                half a = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv).a
                       * SAMPLE_TEXTURE2D(_AlphaMask, sampler_AlphaMask, input.uv).r
                       * _BaseColor.a * _AlphaScale;
                clip(a - _AlphaCutoff);
                return 0;
            }
            ENDHLSL
        }
    }

    FallBack "Universal Render Pipeline/Lit"
}
```

---

## 2. Render Feature（URP 必需）

> [!warning] 为什么必须写这个
> Built-in 管线会按顺序把 Shader 里的 Pass 全跑一遍；**URP 是单次遍历（Single-Pass）管线**，只会执行与当前渲染阶段 `LightMode` 匹配的那个 Pass。
> `LightMode = "HairTransparent"` 不是 URP 认识的内置标签 → **不会被自动执行**。必须写一个 Render Feature，在透明队列结束后专门跑一趟。

`Assets/Scripts/Rendering/HairTransparentFeature.cs`

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;
#if UNITY_6000_0_OR_NEWER
using UnityEngine.Rendering.RenderGraphModule;
#endif

/// <summary>
/// 在透明队列渲染结束后，单独跑一趟 LightMode = "HairTransparent" 的 Pass。
/// 依赖不透明阶段已写入的深度缓冲，配合 Shader 里的 ZTest Less 完成分层。
/// </summary>
[DisallowMultipleRendererFeature("Hair Transparent Pass")]
public class HairTransparentFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class Settings
    {
        public RenderPassEvent renderPassEvent = RenderPassEvent.AfterRenderingTransparents;
        public LayerMask layerMask = -1;
        [Tooltip("与 Shader 里 Tags{ LightMode = ... } 一致")]
        public string lightModeTag = "HairTransparent";
    }

    public Settings settings = new Settings();
    private HairTransparentPass m_Pass;

    public override void Create()
    {
        m_Pass = new HairTransparentPass(settings);
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        var camType = renderingData.cameraData.cameraType;
        if (camType == CameraType.Preview || camType == CameraType.Reflection)
            return;

        renderer.EnqueuePass(m_Pass);
    }

    // ----------------------------------------------------------------
    class HairTransparentPass : ScriptableRenderPass
    {
        const string k_ProfilerTag = "Hair Transparent";
        static readonly ProfilingSampler s_Sampler = new ProfilingSampler(k_ProfilerTag);

        readonly List<ShaderTagId> m_ShaderTags;
        FilteringSettings m_FilteringSettings;

        public HairTransparentPass(Settings settings)
        {
            renderPassEvent = settings.renderPassEvent;
            m_ShaderTags = new List<ShaderTagId> { new ShaderTagId(settings.lightModeTag) };

            // 材质队列在 AlphaTest(2450)，属于 opaque range，
            // 所以这里必须用 all，否则一个都筛不出来。
            m_FilteringSettings = new FilteringSettings(RenderQueueRange.all, settings.layerMask);
        }

#if UNITY_6000_0_OR_NEWER
        private class PassData
        {
            public RendererListHandle rendererList;
        }

        public override void RecordRenderGraph(RenderGraph renderGraph, ContextContainer frameData)
        {
            var renderingData = frameData.Get<UniversalRenderingData>();
            var cameraData    = frameData.Get<UniversalCameraData>();
            var lightData     = frameData.Get<UniversalLightData>();
            var resourceData  = frameData.Get<UniversalResourceData>();

            using (var builder = renderGraph.AddRasterRenderPass<PassData>(k_ProfilerTag, out var passData, s_Sampler))
            {
                var drawSettings = RenderingUtils.CreateDrawingSettings(
                    m_ShaderTags, renderingData, cameraData, lightData,
                    SortingCriteria.CommonTransparent);   // 由远及近

                var param = new RendererListParams(renderingData.cullResults, drawSettings, m_FilteringSettings);
                passData.rendererList = renderGraph.CreateRendererList(param);
                builder.UseRendererList(passData.rendererList);

                builder.SetRenderAttachment(resourceData.activeColorTexture, 0);
                // ZWrite Off，深度只读；ZTest Less 需要读到不透明阶段写入的深度
                builder.SetRenderAttachmentDepth(resourceData.activeDepthTexture, AccessFlags.Read);
                builder.AllowPassCulling(false);

                builder.SetRenderFunc((PassData data, RasterGraphContext ctx) =>
                {
                    ctx.cmd.DrawRendererList(data.rendererList);
                });
            }
        }
#endif

        // Compatibility Mode / URP 14~16
#pragma warning disable 618, 672
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            CommandBuffer cmd = CommandBufferPool.Get();
            using (new ProfilingScope(cmd, s_Sampler))
            {
                context.ExecuteCommandBuffer(cmd);
                cmd.Clear();

                var drawSettings = CreateDrawingSettings(m_ShaderTags, ref renderingData, SortingCriteria.CommonTransparent);
                context.DrawRenderers(renderingData.cullResults, ref drawSettings, ref m_FilteringSettings);
            }
            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
#pragma warning restore 618, 672
    }
}
```

---

## 3. 接线步骤

1. 两个文件丢进工程 → 在 **Universal Renderer Data** 上 `Add Renderer Feature` → **Hair Transparent Pass**，保持 `AfterRenderingTransparents`。
2. 材质用 `Custom/Hair/HairDualPass`，Queue 保持 `AlphaTest`（2450）。
3. 头发模型的 **UV 必须沿发丝方向排布**。一般 V 沿发丝走 → 用默认的 Bitangent；如果 UV 是 U 沿发丝，勾上 `发丝流向用 Tangent`。
4. `_ShiftTex` 给一张沿发丝方向拉伸的灰度噪声（中值 0.5），控制天使环的破碎感。
5. **调参顺序建议：**
   1. 先 `_AlphaCutoff` 定实心边界；
   2. `_ThicknessScale` / `_ThicknessBias` 把漫反射调干净；
   3. `_PrimaryShift` / `_SecondaryShift` 拉开两层光环；
   4. 最后加 Marschner 透射。

---

## 4. 混合法线：为什么要拆

| | 用什么法线 | 原因 |
|---|---|---|
| Diffuse | **Sphere Normal**（球形法线） | 每根发丝各自的法线会让漫反射出现"脏"的噪点。用一个包裹头部的平滑球法线，光照像照在光滑球体上，柔和整洁。 |
| Specular | **Bitangent**（发丝流向，不用法线） | 高光基于发丝流向计算。无论球法线怎么平滑，高光依然准确落在每根发丝上，形成天使环。 |

球形法线在**顶点阶段**计算，插值后天然平滑，比逐像素算更便宜也更干净。

---

## 5. 高光模型：KK + Marschner 的分工

头发在微观上是无数细圆柱体。光照在圆柱上的反射不是一个**点**，而是一个**圆锥**——这是 Kajiya-Kay 的物理直觉来源。

但 KK 本质上还是把头发当成不透明细管（像金属），**不能模拟透射**。所以补一个简化 Marschner TT 项来模拟光穿过发丝的次表面散射。完整 Marschner（R / TT / TRT 三波瓣 + 方位角积分）开销太大，实时里一般不上。

| 项 | 模型 | 作用 |
|---|---|---|
| `spec1` | Kajiya-Kay | 表面白色反光，发丝光泽度，第一层光环 |
| `spec2` | Kajiya-Kay | 带颜色的次级反光（TRT 简化版），让头发颜色更丰富 |
| `mars` | Marschner TT | 逆光下的通透感 |

**高光偏移**把两层错开：

```hlsl
float shiftVal = shiftTex.r - 0.5;
float3 t1 = normalize(strandDir + (shiftVal + _PrimaryShift)   * geometryNormal);
float3 t2 = normalize(strandDir + (shiftVal + _SecondaryShift) * geometryNormal);
```

---

## 6. ⚠️ 对原文的几处修正

推导过程中发现原文公式有几处需要补强，这几条是实际接进工程时会直接影响效果的：

| 位置 | 原文写法 | 问题 / 处理 |
|---|---|---|
| **Kajiya-Kay** | `pow(sinTH, power) * scale` | 缺 `dirAtten`。光源在发丝背侧时 `sinTH` 依然接近 1，会在暗面浮出假高光。加 `smoothstep(-1, 0, dotTH)` 压掉。 |
| **Marschner** | 注释掉的 `dot(T,H)`，H 沿用 KK 的 `L+V` | 那样算出来只是"更钝的 KK"，不是透射。TT 波瓣的半程向量应是 `V - L`，再乘一个 `-N·L` 的背光可见性，才有真的逆光通透感。 |
| **球形法线** | `sin(worldPos.x + ...)` | 用世界坐标当相位，**角色一走动球形法线就会滑**（远景尤其明显）。加了 `_SPHERENORMAL_RADIAL` 开关切成"以头部中心为球心的径向法线"，稳定得多。原文写法保留为默认。 |
| **透射 × 阴影** | `* lightAttenuation` 一刀切 | 透射的物理意义就是光穿过头发，被自身阴影完全掐掉不合理。用 `_MarschnerShadow` 控制透射项受阴影影响的比例。 |
| **Feature 过滤** | 原文未提 | 材质在 `AlphaTest`(2450) 队列属于 **opaque range**，Feature 里若写 `RenderQueueRange.transparent` 会**一个物体都筛不到**。必须 `all`。这是最容易踩的坑。 |

> [!caution] 方案天花板
> Pass 2 内部（发片与发片之间）仍然是 `ZWrite Off` 的乱序混合，只靠 `SortingCriteria.CommonTransparent` 做**逐物体**排序。同一网格内多层发片重叠时依然会有排序错误 —— 这是双 Pass 方案的上限。要再进一步就得上 **OIT（Order-Independent Transparency）/ Depth Peeling**，代价陡增。
