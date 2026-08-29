---
tags:
  - Unity
  - Shader
  - 渲染管线
  - SRP
created: 2026-08-19
---

# Unity 六类 Shader 写法差异与注意点总结

> [!abstract] 架构总览
>
> Unity 的渲染生态已由黑盒化的**内置管线（Built-in Render Pipeline）**全面转向数据驱动的**可编程渲染管线（SRP，即 URP / HDRP）**。着色器的编写不仅是语法的变迁（从 Cg 到 HLSL），更是**数据调度方式**（SRP Batcher 的 CBUFFER 内存布局）、**光照求交逻辑**（光栅化 vs 硬件 Ray Tracing）与**变体管理策略**（Shader Variant Collection / Stripping）的底层重构。
>
> 本文从**写法差异**、**注意点**、**基于渲染管线的差异**三个维度，横向对比六类 Shader 资产。

## 0. 总览对比表

| 维度 | Standard Surface | Unlit | Image Effect | Compute | Ray Tracing | Shader Variant Collection |
|---|---|---|---|---|---|---|
| **文件扩展名** | `.shader` | `.shader` | `.shader` | `.compute` | `.raytrace`（HLSL） | `.shadervariants` |
| **本质** | 代码生成宏（Code Generator） | 顶点/片元着色器 | 全屏片元着色器 | GPGPU 通用计算 | 硬件光线求交 | 变体预热资产 |
| **管线支持** | 仅 Built-in | 全管线 | 全管线（接口不同） | 管线无关 | 仅 HDRP | 全管线 |
| **光照** | 自动（Standard） | 无 | 无 | 无 | 自定义 | — |
| **核心关键字** | `#pragma surface` | `#pragma vertex/fragment` | `vert_img` / `Blit` | `#pragma kernel` | `#pragma raytracing` | 无（资产） |
| **是否挂材质** | 是 | 是 | 是（后处理） | 否（C# Dispatch） | 否（C# Dispatch） | 否 |

---

## 1. Standard Surface Shader（标准表面着色器）

**定义与本质**

Surface Shader 并不是一种图形学 API 概念，而是 Unity 早期开发的一套**代码生成器宏语言**。开发者只需在 `surf()` 函数中声明表面的物理属性（Albedo、Normal、Metallic、Smoothness），编译器便会在底层自动展开生成 ForwardBase、ForwardAdd、ShadowCaster、Deferred 等多个 Pass，以求解近似的渲染方程：

$$
L_o = L_e + \int_{\Omega} f_r(p, \omega_i, \omega_o) L_i(\omega_i) (\omega_i \cdot \mathbf{n}) \, d\omega_i
$$

**写法结构**

```hlsl
Shader "Custom/StandardSurface"
{
    Properties
    {
        _MainTex   ("Albedo (RGB)", 2D)      = "white" {}
        _BumpMap   ("Normal Map",    2D)      = "bump"  {}
        _Glossiness("Smoothness",    Range(0,1)) = 0.5
        _Metallic  ("Metallic",      Range(0,1)) = 0.0
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "Queue"="Geometry" }
        LOD 200

        // ⚠️ 表面着色器必须使用 CGPROGRAM（旧 CG 语法），不是 HLSLPROGRAM
        CGPROGRAM
        #pragma surface surf Standard fullforwardshadows
        #pragma target 3.0

        sampler2D _MainTex;
        sampler2D _BumpMap;

        half _Glossiness;
        half _Metallic;

        // Input 结构体字段名有约定：uv_<贴图名> 会被自动填充对应 UV
        struct Input
        {
            float2 uv_MainTex;
            float2 uv_BumpMap;
        };

        // 只需输出表面属性，光照由 Standard 模型自动完成
        void surf (Input IN, inout SurfaceOutputStandard o)
        {
            fixed4 c = tex2D(_MainTex, IN.uv_MainTex);
            o.Albedo      = c.rgb;
            o.Metallic    = _Metallic;
            o.Smoothness  = _Glossiness;
            o.Normal      = UnpackNormal(tex2D(_BumpMap, IN.uv_BumpMap));
            o.Alpha       = c.a;
        }
        ENDCG
    }
    FallBack "Diffuse"
}
```

**自定义光照模型（进阶）**

若需绕过 Standard，可自定义 `Lighting` 函数，并**需手动声明 `SurfaceOutput` 结构体**：

```hlsl
CGPROGRAM
#pragma surface surf MyLambert

// 自定义时结构体不再自动提供，必须自行定义
struct SurfaceOutput
{
    fixed3 Albedo;
    fixed3 Normal;
    fixed3 Emission;
    half   Specular;
    fixed  Gloss;
    fixed  Alpha;
};

// 函数命名规范：Lighting + 光照模型名
half4 LightingMyLambert (SurfaceOutput s, half3 lightDir, half atten)
{
    half NdotL = saturate(dot(s.Normal, lightDir));
    half4 c;
    c.rgb = s.Albedo * _LightColor0.rgb * (NdotL * atten);
    c.a   = s.Alpha;
    return c;
}

void surf (Input IN, inout SurfaceOutput o)
{
    o.Albedo = fixed3(1, 1, 1);
}
ENDCG
```

> [!warning] 注意点
>
> - **只能挂在 Built-in 管线**。在 URP / HDRP 中会直接渲染为品红色（Magenta / Pink）报错。
> - 必须用 `CGPROGRAM ... ENDCG`，且多数变量类型习惯用 `half` / `fixed` 以匹配移动端精度。
> - `Input` 结构体的字段名是**约定俗成的魔法名**，非任意命名：`uv_MainTex`（UV）、`worldPos`、`worldNormal`、`viewDir`、`screenPos`、`color` 等；用了法线贴图再取 `worldRefl` 时需追加 `INTERNAL_DATA`。
> - `#pragma surface` 的可选修饰符很关键：`addshadow`（让自定义顶点修改也投影）、`noshadow` / `nolightmap` / `noforwardadd`、`alpha:blend/fade/premul`、`vertex:`、`finalcolor:` 等。
> - **变体生成不可控**：一个 surface 会隐式生成大量 Pass 组合，是变体膨胀的来源之一，无法做细粒度剔除。

**基于管线的差异**

- **Built-in**：`#pragma surface surf Standard` 直接驱动底层光照。
- **URP / HDRP**：

  > [!danger] 完全废弃
  >
  > SRP 的设计哲学是**管线状态绝对可控**，因此拒绝了 surface 的黑盒代码生成。
  >
  > - 替代方案一：手写纯 HLSL 的多 Pass 结构，调用库内 BRDF（如 URP 的 `UniversalFragmentPBR`）。
  > - 替代方案二（工程主流）：用 Shader Graph / Amplify Shader Editor 连线，直接编译出符合 SRP 规范的多 Pass HLSL。

---

## 2. Unlit Shader（无光照着色器）

**定义与本质**

Unlit 是**最基础的可编程管线映射**：不引入环境光、方向光或阴影，仅在顶点阶段完成空间变换、在片元阶段输出颜色。核心即 MVP 齐次裁剪变换：

$$
v_{clip} = M_{proj} \cdot M_{view} \cdot M_{model} \cdot v_{local}
$$

它也是理解 Built-in 与 SRP 语法差异的最小样本。**Built-in 写法**：

```hlsl
Shader "Custom/Unlit"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _Color   ("Color", Color) = (1,1,1,1)
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "Queue"="Geometry" }
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            struct appdata { float4 vertex : POSITION; float2 uv : TEXCOORD0; };
            struct v2f     { float2 uv : TEXCOORD0; float4 vertex : SV_POSITION; };

            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed4 _Color;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv     = TRANSFORM_TEX(v.uv, _MainTex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                return tex2D(_MainTex, i.uv) * _Color;
            }
            ENDCG
        }
    }
}
```

**URP 版本（含 SRP Batcher 规范）**

```hlsl
Shader "Custom/URPUnlit"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _Color   ("Color", Color) = (1,1,1,1)
    }
    SubShader
    {
        // ⚠️ 必须声明所属管线，否则 SRP 无法识别
        Tags { "RenderPipeline"="UniversalPipeline" "RenderType"="Opaque" "Queue"="Geometry" }
        Pass
        {
            Name "ForwardLit"
            Tags { "LightMode"="UniversalForward" }

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            // ⚠️ SRP Batcher 要求：所有材质属性必须包进 CBUFFER
            CBUFFER_START(UnityPerMaterial)
                float4 _MainTex_ST;
                float4 _Color;
            CBUFFER_END

            // 用宏声明纹理与采样器，替代 sampler2D
            TEXTURE2D(_MainTex);
            SAMPLER(sampler_MainTex);

            struct Attributes { float4 positionOS : POSITION; float2 uv : TEXCOORD0; };
            struct Varyings   { float4 positionHCS : SV_POSITION; float2 uv : TEXCOORD0; };

            Varyings vert (Attributes IN)
            {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv          = TRANSFORM_TEX(IN.uv, _MainTex);
                return OUT;
            }

            half4 frag (Varyings IN) : SV_Target
            {
                return SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, IN.uv) * _Color;
            }
            ENDHLSL
        }
    }
}
```

**Built-in → URP 的 Diff 视角**

```diff
  Shader "Custom/Unlit"
  {
      SubShader
      {
-         Tags { "RenderType"="Opaque" "Queue"="Geometry" }
+         Tags { "RenderPipeline"="UniversalPipeline" "RenderType"="Opaque" "Queue"="Geometry" }
          Pass
          {
-             CGPROGRAM
+             HLSLPROGRAM
              #pragma vertex vert
              #pragma fragment frag
-             #include "UnityCG.cginc"
+             #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

-             struct appdata { float4 vertex : POSITION; float2 uv : TEXCOORD0; };
-             struct v2f     { float2 uv : TEXCOORD0; float4 vertex : SV_POSITION; };
+             CBUFFER_START(UnityPerMaterial)
+                 float4 _MainTex_ST;
+                 float4 _Color;
+             CBUFFER_END
+
+             TEXTURE2D(_MainTex);
+             SAMPLER(sampler_MainTex);
+
+             struct Attributes { float4 positionOS : POSITION; float2 uv : TEXCOORD0; };
+             struct Varyings   { float4 positionHCS : SV_POSITION; float2 uv : TEXCOORD0; };

-             v2f vert(appdata v) { v2f o;
-                 o.vertex = UnityObjectToClipPos(v.vertex);
+             Varyings vert(Attributes IN) { Varyings OUT;
+                 OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
-                 o.uv = TRANSFORM_TEX(v.uv, _MainTex); return o; }
+                 OUT.uv = TRANSFORM_TEX(IN.uv, _MainTex); return OUT; }

-             fixed4 frag(v2f i) : SV_Target {
-                 return tex2D(_MainTex, i.uv) * _Color; }
+             half4 frag(Varyings IN) : SV_Target {
+                 return SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, IN.uv) * _Color; }
-             ENDCG
+             ENDHLSL
          }
      }
  }
```

> [!warning] 注意点
>
> - **SRP Batcher 兼容性**：URP/HDRP 中所有 `Properties` 声明的变量**必须**包进 `CBUFFER_START(UnityPerMaterial) ... CBUFFER_END`；违反会导致该材质无法批处理（退回 SRP Batcher 前的逐材质 SetPass）。
> - 纹理访问改用 `TEXTURE2D` + `SAMPLER` + `SAMPLE_TEXTURE2D` 宏，以兼容多平台（含 Metal / Vulkan 的采样器分离规范）。
> - 命名约定：URP 用 `positionOS`（Object Space）/ `positionHCS`（Homogeneous Clip Space），替代旧的 `vertex` 语义，避免误解。
> - `TRANSFORM_TEX` 依赖 `_MainTex_ST`（Tiling/Offset），遗漏会报错。
> - HDRP 的 Unlit 更重：通常基于 HDRP 内置 Lit/Unlit 模板或 Shader Graph，需处理 `SurfaceType`（Opaque/Transparent）、`BlendMode` 等宏，不建议从零手写。

---

## 3. Image Effect Shader（屏幕后处理着色器）

**定义与本质**

后处理本质上是一个**不做模型变换的全屏 Unlit**：在屏幕空间渲染一个覆盖视口的几何体（现代管线用**覆盖全屏的大三角形**而非 Quad，以避免对角线处的像素 Overdraw），对当前 Render Target 采样并滤波（Bloom、景深、泛光、色调映射等）。

**Built-in 写法（OnRenderImage + Graphics.Blit）**

C# 端：

```csharp
[RequireComponent(typeof(Camera))]
public class MyImageEffect : MonoBehaviour
{
    public Material effectMaterial;

    void OnRenderImage(RenderTexture src, RenderTexture dst)
    {
        Graphics.Blit(src, dst, effectMaterial);
    }
}
```

Shader 端（Blit Shader）：

```hlsl
Shader "Hidden/MyImageEffect"
{
    Properties { _MainTex ("Texture", 2D) = "white" {} }
    SubShader
    {
        // ⚠️ 后处理三件套：关闭深度写入、始终通过深度测试、关闭背面剔除
        Cull Off
        ZWrite Off
        ZTest Always

        Pass
        {
            CGPROGRAM
            #pragma vertex vert_img
            #pragma fragment frag
            #include "UnityCG.cginc"

            sampler2D _MainTex;

            // vert_img / v2f_img 由 UnityCG 提供，直接复用
            fixed4 frag (v2f_img i) : SV_Target
            {
                fixed4 col = tex2D(_MainTex, i.uv);
                return col;
            }
            ENDCG
        }
    }
}
```

**URP 写法（ScriptableRendererFeature + ScriptableRenderPass + 全屏三角形）**

Shader 端：

```hlsl
Shader "Hidden/URPImageEffect"
{
    SubShader
    {
        Tags { "RenderPipeline"="UniversalPipeline" }
        Pass
        {
            ZWrite Off ZTest Always Cull Off
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            TEXTURE2D(_BlitTexture);
            SAMPLER(sampler_BlitTexture);

            struct Attributes { uint vertexID : SV_VertexID; };
            struct Varyings   { float4 positionCS : SV_POSITION; float2 uv : TEXCOORD0; };

            Varyings vert (Attributes IN)
            {
                Varyings OUT;
                // 用顶点 ID 直接构造全屏大三角形，避免上传 Quad 顶点
                OUT.positionCS = GetFullScreenTriangleVertexPosition(IN.vertexID);
                OUT.uv         = GetFullScreenTriangleTexCoord(IN.vertexID);
                return OUT;
            }

            half4 frag (Varyings IN) : SV_Target
            {
                return SAMPLE_TEXTURE2D(_BlitTexture, sampler_BlitTexture, IN.uv);
            }
            ENDHLSL
        }
    }
}
```

C# 端（Feature + Pass 骨架）：

```csharp
public class MyBlitFeature : ScriptableRendererFeature
{
    public Material blitMaterial;
    public RenderPassEvent renderPassEvent = RenderPassEvent.AfterRenderingTransparents;
    MyBlitPass pass;

    public override void Create() => pass = new MyBlitPass(blitMaterial, renderPassEvent);
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
        => renderer.EnqueuePass(pass);
}

public class MyBlitPass : ScriptableRenderPass
{
    Material mat; RenderTargetIdentifier src;
    public MyBlitPass(Material m, RenderPassEvent e) { mat = m; renderPassEvent = e; }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        CommandBuffer cmd = CommandBufferPool.Get();
        src = renderingData.cameraData.renderer.cameraColorTarget;
        // 新版 URP 用 Blitter 替代过时的 CommandBuffer.Blit
        Blitter.BlitCameraTexture(cmd, src, src, mat, 0);
        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }
}
```

> [!warning] 注意点
>
> - **后处理三件套不可省**：`ZWrite Off`（不写深度）、`ZTest Always`（无视深度直接通过）、`Cull Off`（全屏三角形双面）。
> - **不要用 Quad**：全屏大三角形（`GetFullScreenTriangleVertexPosition`）避免对角线 Overdraw 与顶点上传。
> - 命名习惯放 `Hidden/` 下，避免出现在材质选择器里。
> - **HDRP 用 Custom Pass**：挂载到 Custom Pass Volume，配合高精度浮点颜色缓冲（16-bit Float / R11G11B10）工作；后处理链需留意色调映射（Tonemapping）的注入时机。

**基于管线的差异**

- **Built-in**：强耦合 `Camera.OnRenderImage` + `Graphics.Blit`。
- **URP**：`ScriptableRendererFeature` + `ScriptableRenderPass` + `Blitter`；可精确插入到 `RenderPassEvent` 队列（如 `AfterRenderingTransparents`）。Unity 2022.2+ 也提供无需写代码的 **Full Screen Pass Renderer Feature**。
- **HDRP**：**Custom Pass + Volume System**，插在 HDRP 渲染帧的特定注入点，需处理颜色空间与高精度缓冲。

---

## 4. Compute Shader（计算着色器）

**定义与本质**

Compute Shader 是利用 GPU 大规模并行能力处理**非图形渲染任务（GPGPU）**的程序，完全脱离顶点/片元光栅化管线。基于 DirectX 11 DirectCompute 规范，通过线程组布局 `[numthreads(x,y,z)]` 实现极高并发读写，常用于视锥剔除（Frustum Culling）、GPU 粒子、流体模拟、积分计算等。

**Shader 端（`.compute` 文件，纯 HLSL，无 ShaderLab）**

```hlsl
// GPUParticles.compute
#pragma kernel CSMain

RWStructuredBuffer<float4> _Particles;  // 可读写结构化缓冲
StructuredBuffer<float4>   _Forces;     // 只读结构化缓冲
float _DeltaTime;

// 每个线程组 64 个线程，沿一维展开
[numthreads(64, 1, 1)]
void CSMain (uint3 id : SV_DispatchThreadID)   // 全局线程 ID
{
    float3 pos = _Particles[id.x].xyz;
    float3 force = _Forces[id.x].xyz;
    pos += force * _DeltaTime;
    _Particles[id.x] = float4(pos, 1.0);
}
```

**C# 调度端**

```csharp
public class GPUParticles : MonoBehaviour
{
    public ComputeShader compute;
    public int particleCount = 65536;
    ComputeBuffer particleBuffer, forceBuffer;

    void Start()
    {
        particleBuffer = new ComputeBuffer(particleCount, sizeof(float) * 4);
        forceBuffer    = new ComputeBuffer(particleCount, sizeof(float) * 4);

        int kernel = compute.FindKernel("CSMain");
        compute.SetBuffer(kernel, "_Particles", particleBuffer);
        compute.SetBuffer(kernel, "_Forces",    forceBuffer);
    }

    void Update()
    {
        int kernel = compute.FindKernel("CSMain");
        compute.SetFloat("_DeltaTime", Time.deltaTime);
        // Dispatch 参数是「线程组数」= 线程数 / numthreads
        compute.Dispatch(kernel, particleCount / 64, 1, 1);
    }
}
```

> [!tip] 关键换算
>
> **线程组数 × `numthreads` = 总线程数**。上例 64 线程/组，65536 个粒子需要 `65536 / 64 = 1024` 个线程组。`SV_DispatchThreadID` 是全局线程索引，`SV_GroupID` / `SV_GroupThreadID` 分别代表组 ID 与组内线程 ID。

**异步回读（AsyncGPUReadback，避免 CPU 阻塞等待）**

```csharp
void Readback()
{
    AsyncGPUReadback.Request(particleBuffer, request =>
    {
        if (request.hasError) return;
        var data = request.GetData<float4>();   // 数据已在回调时回到 CPU
    });
}
```

> [!warning] 注意点
>
> - **`.compute` 与 `.shader` 语法互通但不直接互嵌**：可以共享 `.hlsl` / `.cginc` 公共头，但 `#pragma kernel` 只存在于 `.compute`；`.compute` 也**不能**直接挂到材质上。
> - **管线无关**：`.compute` 语法在 Built-in / URP / HDRP **完全一致**，唯一差异在 C# 调度方式——SRP 环境更推荐 `CommandBuffer.DispatchCompute()` 以与渲染队列安全同步（而非 `ComputeShader.Dispatch()` 直调）。
> - 读写资源用 `RWStructuredBuffer` / `RWTexture2D`，只读用 `StructuredBuffer` / `Texture2D`；GPU 内存由 `ComputeBuffer` 或 `RenderTexture.enableRandomWrite` 承载，用完必须 `Release()`。
> - 回读数据到 CPU 有显著延迟，优先用 `AsyncGPUReadback`。
> - 线程组尺寸尽量是 GPU warp/wavefront（NVIDIA 32 / AMD 64）的整数倍以提升占用率；注意共享内存（`groupshared`）与内存屏障（`GroupMemoryBarrierWithGroupSync`）的同步。
> - 平台兼容：`#pragma target` 与双精度（double）在移动端/部分 GPU 不支持。

---

## 5. Ray Tracing Shader（光线追踪着色器）

**定义与本质**

基于 **DXR（DirectX Raytracing）/ Vulkan RT / Metal RT** 的现代着色器阶段，彻底抛弃光栅化，采用物理空间求交。含多个专用阶段：

| 阶段 | HLSL 属性 | 职责 |
|---|---|---|
| Ray Generation | `[shader("raygeneration")]` | 生成主射线，调度 `TraceRay` |
| Miss | `[shader("miss")]` | 射线未命中时的着色（天空盒等） |
| Closest Hit | `[shader("closesthit")]` | 最近命中点的光照计算 |
| Any Hit | `[shader("anyhit")]` | 命中即触发（用于 alpha 测试阴影） |
| Intersection | `[shader("intersection")]` | 自定义图元求交（程序化几何） |

**写法（`.raytrace` 文件，纯 HLSL，HDRP 专属）**

```hlsl
#pragma raytracing RayGen

#include "Packages/com.unity.render-pipelines.high-definition/Runtime/RenderPipeline/Raytracing/Shaders/RaytracingMacros.hlsl"
#include "Packages/com.unity.render-pipelines.high-definition/Runtime/RenderPipeline/Raytracing/Shaders/ShaderVariablesRaytracing.hlsl"

RWTexture2D<float4> _Output;
RaytracingAccelerationStructure _AccelerationStructure;
float4x4 _CameraInvViewProj;

// 自定义 payload 承载各阶段间传递的数据
struct RayPayload
{
    float4 color;
};

[shader("raygeneration")]
void RayGen()
{
    uint2 dispatchIndex = DispatchRaysIndex().xy;
    uint2 dims          = DispatchRaysDimensions().xy;
    float2 uv           = (dispatchIndex + 0.5f) / dims;

    RayDesc ray;
    ray.Origin    = ...;
    ray.Direction = ...;
    ray.TMin      = 0.001f;
    ray.TMax      = 1e20f;

    RayPayload payload = (RayPayload)0;
    TraceRay(_AccelerationStructure, RAY_FLAG_NONE, 0xFF, 0, 1, 0, ray, payload);
    _Output[dispatchIndex] = payload.color;
}

[shader("miss")]
void Miss(inout RayPayload payload)
{
    payload.color = float4(0, 0, 0, 1);   // 未命中 → 黑色（或采样天空）
}

[shader("closesthit")]
void ClosestHit(inout RayPayload payload, AttributeData attributes)
{
    payload.color = float4(1, 1, 1, 1);
}
```

**C# 调度端**

```csharp
using UnityEngine.Rendering;

public class RayTraceRenderer : MonoBehaviour
{
    [SerializeField] RayTracingShader shader;

    public void Render(RenderTexture output, RayTracingAccelerationStructure accel, Camera cam)
    {
        shader.SetShaderPass("RayGen");
        shader.SetTexture("_Output", output);
        shader.SetAccelerationStructure("_AccelerationStructure", accel);
        shader.SetMatrix("_CameraInvViewProj", cam.projectionMatrix.inverse * cam.worldToCameraMatrix.inverse);
        shader.Dispatch("RayGen", output.width, output.height, 1);
    }
}
```

> [!danger] 注意点
>
> - **仅 HDRP 支持**，且要求支持 DXR/Vulkan RT/Metal RT 的硬件（NVIDIA RTX、AMD RDNA2+、Apple Silicon 等）。Built-in 不支持；URP 仅实验性/有限支持。
> - 资源类型是专用的 `RayTracingShader` / `RayTracingAccelerationStructure`，不是普通 `Shader` / `ComputeShader`。
> - `TraceRay` 有递归深度限制（HDRP 默认约 31 层），`RayPayload` 结构体过大（>32 字节）会显著影响性能。
> - **降噪（Denoising）** 是刚需：单次采样噪声极大，需要 HDRP 内置的时空降噪器或累积帧（Temporal Accumulation）。
> - 加速结构（BLAS/TLAS）构建有开销，动态物体需增量重建；剔除光追对象能大幅降本。
> - 与光栅化混合渲染时，需协调好 G-Buffer 与光追结果在合成（Composite）阶段的融合。

---

## 6. Shader Variant Collection（着色器变体集合）

**定义与本质**

SVC **不是代码文件，而是一种资产（Asset）**。Shader 中的 `#pragma multi_compile` / `#pragma shader_feature` 会产生多个编译排列（Permutations）。若某 Keyword 组合的变体在运行时才被首次使用，GPU 会**即时编译**，导致严重的帧率毛刺（Hitch）。SVC 的作用就是把工程实际用到的 Keyword 组合打包，在加载阶段（Loading）提前交给 GPU 编译预热。

**资产用法**

- 菜单 `Create > Shader > Shader Variant Collection` 创建 `.shadervariants` 资产；
- 在 Project Settings > Graphics 中挂入 **Preloaded Shaders**，构建时随包预编译；
- 或运行时手动 `WarmUp()`。

```csharp
// 运行时手动预热
ShaderVariantCollection svc = new ShaderVariantCollection();
svc.Add(new ShaderVariantCollection.ShaderVariant(
    shader,
    PassType.Normal,
    new[] { "_NORMALMAP", "_METALLICGLOSSMAP" }   // 该变体启用的 Keyword 组合
));
svc.WarmUp();   // 立即触发 GPU 编译，避免首帧卡顿
```

**关键字声明方式对比（决定变体是否进包）**

```hlsl
// multi_compile：所有组合都会编译并进包（适合运行时可随意切换）
#pragma multi_compile _ _NORMALMAP

// shader_feature：仅被材质实际用到的组合才进包（默认推荐，控制包体）
#pragma shader_feature _ _EMISSION

// 局部关键字（Unity 2021.1+）：不占用全局关键字位，用 per-material 存储
#pragma shader_feature_local _ _DETAIL

// 全局关键字仍用 Shader.EnableKeyword / 材质 Keyword 开关
#pragma multi_compile _ _GLOBAL_ON
```

**变体剥离（Shader Stripping，控制组合爆炸）**

```csharp
// 实现 IPreprocessShaders，在构建期剔除不用的变体
class KeywordStripper : IPreprocessShaders
{
    ShaderKeyword unused = new ShaderKeyword("_SOME_UNUSED_KEYWORD");

    public void OnProcessShader(Shader shader, ShaderSnippetData snippet, IList<ShaderCompilerData> data)
    {
        for (int i = data.Count - 1; i >= 0; --i)
        {
            if (data[i].shaderKeywordSet.IsEnabled(unused))
                data.RemoveAt(i);   // 剔除含该关键字的所有变体
        }
    }
}
```

> [!warning] 注意点
>
> - **变体爆炸是 2^n 问题**：`n` 个 Keyword 组合理论上产生 $2^n$ 个变体，必须靠 SVC 预热 + Stripping 剥离双管齐下。
> - **`shader_feature` vs `multi_compile`**：前者按需进包（省包体，但运行时若遇到未预编译组合会卡顿/丢效果），后者全量进包（省心但膨胀）。
> - **全局关键字位有限**：Built-in 全局关键字上限 **128**，SRP 上限 **256**；超限会直接报错，应优先用 `_local` 变体。
> - **Stripping 有副作用**：若剥离了运行时实际会切到的关键字，会出现材质变紫或丢效果，需覆盖测试所有 Keyword 切换路径。
> - SVC 只解决**预热**，不解决**包体**；包体由 Stripping + 合理的 Keyword 设计共同决定。

**基于管线的差异**

- **Built-in**：变体数量相对可控，主要围绕前向光照、点光源数量、雾效等。
- **URP / HDRP（SRP）**：**变体爆炸重灾区**。阴影级联、光源类型、贴花、SSAO、HDR 输出等全部设计为 Shader Keyword，URP 单个 Shader 极易产生数十万变体。因此 SRP 架构下**几乎必须**同时使用 SVC 预热 + `IPreprocessShaders` 剥离，并配合 `multi_compile_local` / `shader_feature_local` 控制关键字位占用。

---

## 附：六类 Shader 决策速查

| 需求场景 | 推荐选择 |
|---|---|
| Built-in 管线快速出带光照材质 | Surface Shader |
| URP/HDRP 带光照材质 | Shader Graph / 手写多 Pass HLSL |
| 纯色/贴图、无光照、粒子/UI | Unlit Shader |
| 全屏后处理 | Image Effect（Built-in: OnRenderImage；URP: RendererFeature；HDRP: Custom Pass） |
| GPU 粒子/剔除/大数据并行 | Compute Shader |
| 真实反射/GI/阴影（HDRP + RT 硬件） | Ray Tracing Shader |
| 消除运行时变体卡顿 / 控制包体 | Shader Variant Collection + Stripping |
