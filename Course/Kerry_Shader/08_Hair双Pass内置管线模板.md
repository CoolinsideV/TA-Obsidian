---
日期: 2026-08-20
tags:
  - Shader
  - 头发渲染
  - Built-in
---

> 完整可用的 Unity 内置管线双 Pass 头发渲染模板。原理见 [[07_Hair双Pass头发渲染]]。

## 一、 模板整体结构

- 共享代码放 `CGINCLUDE`，两个 Pass 复用同一套顶点着色器与工具函数。
- Pass 1（背面）`Cull Front` + `ZWrite Off`：输出透光。
- Pass 2（正面）`Cull Back` + `ZWrite On`：输出双层各向异性高光。
- 两个 Pass 均做 `Alpha Test` 裁剪 + `SrcAlpha OneMinusSrcAlpha` 混合。

## 二、 完整 Shader 代码

```hlsl
Shader "Hair/DualPass_Anisotropic_BuiltIn"
{
    Properties
    {
        _MainTex      ("Albedo (RGB) Alpha (A)", 2D) = "white" {}
        _Color        ("Tint Color", Color) = (0.5, 0.3, 0.2, 1)

        _SpecColor1   ("Primary Spec Color", Color)   = (1, 1, 1, 1)
        _SpecColor2   ("Secondary Spec Color", Color) = (0.6, 0.5, 0.4, 1)
        _Shininess1   ("Primary Shininess", Range(8, 256))   = 64
        _Shininess2   ("Secondary Shininess", Range(8, 256)) = 24
        _Shift1       ("Primary Highlight Shift", Range(-1, 1))   = 0.0
        _Shift2       ("Secondary Highlight Shift", Range(-1, 1)) = 0.5

        _BackColor    ("Back Transmission Color", Color) = (0.3, 0.1, 0.05, 1)
        _BackStrength ("Back Transmission Strength", Range(0, 1)) = 0.5

        _Ambient      ("Ambient Strength", Range(0, 1)) = 0.2
        _Cutoff       ("Alpha Cutoff", Range(0, 1))     = 0.1
    }

    SubShader
    {
        Tags { "Queue"="AlphaTest" "RenderType"="TransparentCutout" "IgnoreProjector"="True" }

        // ================================================================
        // 共享代码：顶点着色器 + 各向异性高光工具函数
        // ================================================================
        CGINCLUDE
        #include "UnityCG.cginc"
        #include "Lighting.cginc"

        struct appdata
        {
            float4 vertex : POSITION;
            float3 normal : NORMAL;
            float4 tangent : TANGENT;
            float2 uv : TEXCOORD0;
        };

        struct v2f
        {
            float4 pos       : SV_POSITION;
            float2 uv        : TEXCOORD0;
            float3 worldPos  : TEXCOORD1;
            float3 worldN    : TEXCOORD2;
            float3 worldT    : TEXCOORD3;
        };

        sampler2D _MainTex;
        float4 _MainTex_ST;
        fixed4 _Color;
        fixed4 _SpecColor1;
        fixed4 _SpecColor2;
        half   _Shininess1;
        half   _Shininess2;
        half   _Shift1;
        half   _Shift2;
        fixed4 _BackColor;
        half   _BackStrength;
        half   _Ambient;
        half   _Cutoff;

        v2f vert(appdata v)
        {
            v2f o;
            o.pos      = UnityObjectToClipPos(v.vertex);
            o.uv       = TRANSFORM_TEX(v.uv, _MainTex);
            o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
            o.worldN   = UnityObjectToWorldNormal(v.normal);
            // 发丝切线：直接用 mesh 切线。若高光混乱，需打直 UV / FlowMap 统一方向
            o.worldT   = UnityObjectToWorldDir(v.tangent.xyz);
            return o;
        }

        // 切线偏移：沿法线方向偏移切线，让高光上下移动
        float3 ShiftTangent(float3 T, float3 N, float shift)
        {
            return normalize(T + N * shift);
        }

        // Kajiya-Kay 各向异性高光
        half StrandSpecular(float3 T, float3 V, float3 L, half exponent)
        {
            float3 H = normalize(L + V);
            half dotTH = dot(T, H);
            half sinTH = sqrt(1.0 - dotTH * dotTH);   // 用 sin 产生条状高光
            half dirAtten = smoothstep(-1.0, 0.0, dotTH);
            return dirAtten * pow(sinTH, exponent);
        }
        ENDCG

        // ================================================================
        // Pass 1：背面（透光 / 次表面散射），先渲染，不写深度
        // ================================================================
        Pass
        {
            Name "Back"
            Tags { "LightMode"="ForwardBase" }

            Cull Front
            ZWrite Off
            Blend SrcAlpha OneMinusSrcAlpha

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment fragBack
            #pragma target 3.0

            fixed4 fragBack(v2f i) : SV_Target
            {
                fixed4 albedo = tex2D(_MainTex, i.uv) * _Color;
                clip(albedo.a - _Cutoff);

                float3 N = normalize(i.worldN);
                float3 V = normalize(_WorldSpaceCameraPos - i.worldPos);
                float3 L = normalize(_WorldSpaceLightPos0.xyz);

                // 背面透光：背面法线与光线反向时，光从发丝背后穿透过来
                half transmission = saturate(dot(-N, L));
                transmission = pow(transmission, 2.0) * _BackStrength;

                // 边缘透光：背面轮廓越靠边越亮（菲涅尔近似）
                half rim = 1.0 - saturate(dot(-N, V));
                transmission = max(transmission, rim * _BackStrength);

                fixed3 col = albedo.rgb * _BackColor.rgb * transmission * _LightColor0.rgb;
                return fixed4(col, albedo.a * transmission);
            }
            ENDCG
        }

        // ================================================================
        // Pass 2：正面（主高光 + 次高光），后渲染，写深度
        // ================================================================
        Pass
        {
            Name "Front"
            Tags { "LightMode"="ForwardBase" }

            Cull Back
            ZWrite On
            Blend SrcAlpha OneMinusSrcAlpha

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment fragFront
            #pragma target 3.0

            fixed4 fragFront(v2f i) : SV_Target
            {
                fixed4 albedo = tex2D(_MainTex, i.uv) * _Color;
                clip(albedo.a - _Cutoff);

                float3 N = normalize(i.worldN);
                float3 T = normalize(i.worldT);
                float3 V = normalize(_WorldSpaceCameraPos - i.worldPos);
                float3 L = normalize(_WorldSpaceLightPos0.xyz);
                fixed3 lightColor = _LightColor0.rgb;

                // 环境光
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.rgb * _Ambient;

                // 漫反射（半兰伯特，避免头发暗部死黑）
                half NdotL = dot(N, L) * 0.5 + 0.5;

                // 双层各向异性高光（主高光锐利靠发梢，次高光柔和带色偏靠发根）
                float3 T1 = ShiftTangent(T, N, _Shift1);
                float3 T2 = ShiftTangent(T, N, _Shift2);

                half spec1 = StrandSpecular(T1, V, L, _Shininess1);
                half spec2 = StrandSpecular(T2, V, L, _Shininess2);

                fixed3 specular = _SpecColor1.rgb * spec1 + _SpecColor2.rgb * spec2;

                fixed3 col = albedo.rgb * lightColor * NdotL + ambient + specular * lightColor;
                return fixed4(col, albedo.a);
            }
            ENDCG
        }
    }

    FallBack "Diffuse"
}
```

## 三、 使用说明

1. 贴图 `_MainTex` 的 **Alpha 通道**存头发插片的透明轮廓（发丝形状），不透明区域 A=1，边缘渐变到 0。
2. `_Cutoff` 做硬边裁剪，去掉边缘抖动；`Blend` 再补上平滑的半透明边缘。
3. 高光若出现混乱的锯齿条状，问题几乎都出在**切线方向不一致**——优先打直 UV，或改用 FlowMap / 虚拟球心法统一切线。
4. 背面透光颜色 `_BackColor` 建议偏棕红/暖色，模拟发丝背后透光。

## 四、 与 URP 的差异

- 本模板是 Built-in 的 `ForwardBase` 单平行光写法（`_WorldSpaceLightPos0` / `_LightColor0`）。
- URP 下这些内置变量失效，需改用 `GetMainLight()` 等 API；且 URP 一个 SubShader 最多两个 Pass，正好符合本双 Pass 结构，但更多 Pass 需 RenderFeature 扩展。
