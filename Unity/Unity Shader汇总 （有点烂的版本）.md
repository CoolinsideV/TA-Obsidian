在Unity中，随着渲染管线从Built-in Render Pipeline (BiRP) 向 Scriptable Render Pipeline (SRP, 包含URP和HDRP) 的演进，Shader的编写范式发生了巨大的变化。

## 1. Standard Surface Shader (表面着色器)

Surface Shader 是 Built-in 管线特有的**代码生成器（Code Generator）**，它将光照模型、阴影投射、前向/延迟渲染路径的复杂性封装在了底层。

- **管线支持**：**仅支持 Built-in**。在 URP 和 HDRP 中已被弃用，官方推荐使用 Shader Graph 或直接编写 HLSL Lit Shader。
    
- **核心语法**：使用 `CGPROGRAM` 和 `#pragma surface` 指令。
    
- **注意点**：
    
    - **高开销**：即使只写了几行代码，编译器也会生成包含 ForwardBase, ForwardAdd, Deferred, ShadowCaster 等多个 Pass 的庞大代码。
        
    - **黑盒效应**：很难精准控制指令数（ALU）和寄存器占用，不适合极致的移动端性能优化。
        


```hlsl
Shader "Custom/StandardSurface" {
    Properties { _Color ("Color", Color) = (1,1,1,1) }
    SubShader {
        Tags { "RenderType"="Opaque" }
        CGPROGRAM
        // 指定表面函数 surf，使用 Standard 基于物理的光照模型
        #pragma surface surf Standard fullforwardshadows
        #pragma target 3.0

        struct Input { float2 uv_MainTex; };
        fixed4 _Color;

        void surf (Input IN, inout SurfaceOutputStandard o) {
            o.Albedo = _Color.rgb;
            o.Metallic = 0.0;
            o.Smoothness = 0.5;
            o.Alpha = _Color.a;
        }
        ENDCG
    }
}
```

## 2. Unlit Shader (无光照着色器)

Unlit Shader 提供了最基础的顶点/片元控制权。从 BiRP 迁移到 URP/HDRP 时，主要的差异在于**着色器语言 (CG -> HLSL)**、**包含文件库**以及**矩阵变换宏**的改变。

- **管线支持**：全管线支持（BiRP, URP, HDRP）。
    
- **注意点**：
    
    - **SRP Batcher 兼容性**：在 URP/HDRP 中，所有材质属性必须封装在 `CBUFFER_START(UnityPerMaterial)` 和 `CBUFFER_END` 中，否则无法合批。
        
    - **空间变换**：SRP 中废弃了 `UNITY_MATRIX_MVP` 等全局变量，改为按需获取（利用 `GetWorldToObjectMatrix()` 等函数或专门的 Transform API）。
        

以下代码展示了从 Built-in 迁移到 URP 的 Unlit Shader 核心差异：

Diff

```diff
Shader "Custom/Unlit_BiRP_to_URP" {
    SubShader {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass {
-           CGPROGRAM
+           HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
-           #include "UnityCG.cginc"
+           #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

-           struct appdata {
-               float4 vertex : POSITION;
-               float2 uv : TEXCOORD0;
-           };
+           struct Attributes {
+               float4 positionOS : POSITION; // OS = Object Space
+               float2 uv : TEXCOORD0;
+           };

-           struct v2f {
-               float4 pos : SV_POSITION;
-           };
+           struct Varyings {
+               float4 positionCS : SV_POSITION; // CS = Clip Space
+           };

+           CBUFFER_START(UnityPerMaterial)
+           half4 _BaseColor;
+           CBUFFER_END

-           v2f vert (appdata v) {
-               v2f o;
-               o.pos = UnityObjectToClipPos(v.vertex); // 或者 mul(UNITY_MATRIX_MVP, v.vertex)
-               return o;
-           }
+           Varyings vert (Attributes input) {
+               Varyings output;
+               // 现代SRP API，利用内联函数处理平台差异
+               output.positionCS = TransformObjectToHClip(input.positionOS.xyz); 
+               return output;
+           }

-           fixed4 frag (v2f i) : SV_Target { return fixed4(1,0,0,1); }
+           half4 frag (Varyings input) : SV_Target { return _BaseColor; }
-           ENDCG
+           ENDHLSL
        }
    }
}
```

## 3. Image Effect Shader (屏幕后处理着色器)

后处理的实现机制在管线迭代中发生了根本性改变。

- **管线支持与差异**：
    
    - **Built-in**：依赖 `Camera.OnRenderImage` C# 脚本，使用 `Graphics.Blit`，Shader 约定俗成读取 `_MainTex`。
        
    - **URP**：废弃 `OnRenderImage`。需要编写 `ScriptableRendererFeature`，并在 Shader 中包含 `Blitter.hlsl`。主纹理通常被称为 `_BlitTexture`。
        
    - **HDRP**：使用 `CustomPostProcessVolume` 框架，提供极高自由度但框架约束极严（需考虑全屏三角形生成 `GenerateFullscreenTriangle`）。
        
- **注意点**：现代 SRP 中，为了避免 Quad 的两个三角形对角线像素重复光栅化，标准做法是生成一个覆盖全屏的**大三角形 (Fullscreen Triangle)**。
    

Diff

```diff
// URP 全屏后处理 Shader 片段对比旧版
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment Frag
-           #include "UnityCG.cginc"
+           #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
+           #include "Packages/com.unity.render-pipelines.core/Runtime/Utilities/Blit.hlsl"

-           sampler2D _MainTex;
+           // URP Blit 框架自动绑定的输入纹理
+           TEXTURE2D_X(_BlitTexture); 
+           SAMPLER(sampler_LinearClamp);

            // 顶点着色器无需自定义，直接使用 Blit.hlsl 中内置的 Vert 函数即可
            
-           fixed4 frag (v2f i) : SV_Target {
-               return tex2D(_MainTex, i.uv);
-           }
+           half4 Frag (Varyings input) : SV_Target {
+               // 处理 VR/XR 的立体渲染纹理数组 (TEXTURE2D_X)
+               float2 uv = input.texcoord;
+               half4 color = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_LinearClamp, uv);
+               return color * half4(1.0, 0.5, 0.5, 1.0); // 染色测试
+           }
            ENDHLSL
```

## 4. Compute Shader (计算着色器)

Compute Shader 独立于图形管线（Graphics Pipeline），直接利用 GPU 的 Compute Units 进行通用计算（GPGPU）。

- **管线支持**：全管线通用（本质是 DirectX Compute / OpenGL Compute / Vulkan）。
    
- **核心语法**：`#pragma kernel`，`[numthreads(x,y,z)]`，`RWTexture2D` / `RWStructuredBuffer`。
    
- **注意点**：
    
    - **线程组尺寸 (Thread Group Size)**：`[numthreads(x, y, z)]` 的设定极度影响性能。通常 AMD 架构 Wavefront 为 64，NVIDIA Warp 为 32。一维计算常设为 `[numthreads(64, 1, 1)]`，二维图像处理常设为 `[numthreads(8, 8, 1)]`。
        
    - **内存屏障 (Memory Barrier)**：在处理如 `GroupMemoryBarrierWithGroupSync()` 的共享内存（`groupshared`）时，需谨慎处理死锁。
        


```hlsl
// 一个标准的 Compute Shader 示例 (.compute)
#pragma kernel CSMain

// 读写纹理 (UAV - Unordered Access View)
RWTexture2D<float4> Result;
// 只读结构化缓冲区 (SRV - Shader Resource View)
StructuredBuffer<float3> Positions;

// 定义线程组维度
[numthreads(8, 8, 1)]
void CSMain (uint3 id : SV_DispatchThreadID, uint3 groupThreadID : SV_GroupThreadID)
{
    // 获取当前像素坐标
    uint x = id.x;
    uint y = id.y;
    
    // 防越界保护 (极重要，否则可能导致GPU Device Lost)
    uint width, height;
    Result.GetDimensions(width, height);
    if(x >= width || y >= height) return;

    // 简单分形计算或数据回写
    Result[id.xy] = float4(id.x / (float)width, id.y / (float)height, 0.0, 1.0);
}
```

## 5. Ray Tracing Shader (光线追踪着色器)

基于 DXR (DirectX Raytracing) 或 Vulkan RT，完全不同于传统光栅化着色器体系。

- **管线支持**：主要在 **HDRP** 中成熟支持（URP目前需自行实现底层逻辑或依赖极新版本的扩展）。
    
- **核心架构**：由一系列专门的 Shader 类型组成：`RayGeneration`, `ClosestHit`, `AnyHit`, `Miss`, `Intersection`。
    
- **注意点**：
    
    - **负载 (Payload)**：`TraceRay` 携带的 Payload 结构体应当尽可能小，因为其存储在极昂贵的硬件寄存器中。
        
    - **递归深度 (Recursion Depth)**：通过 `#pragma max_recursion_depth` 限制，过深会导致栈溢出或极大的性能下降。
        


```hlsl
// DXR Ray Tracing Shader 语法片段 (.raytrace)
#pragma max_recursion_depth 1

// 加速结构 (TLAS)
RaytracingAccelerationStructure g_SceneAccelStruct;
RWTexture2D<float4> g_Output;

// Payload 结构
struct RayPayload {
    float4 color;
};

// 光线生成着色器入口
[shader("raygeneration")]
void MyRaygenShader() {
    uint2 dispatchIdx = DispatchRaysIndex().xy;
    uint2 dispatchDim = DispatchRaysDimensions().xy;
    
    // 计算屏幕 UV
    float2 uv = (float2(dispatchIdx) + 0.5f) / float2(dispatchDim);
    
    RayDesc ray;
    ray.Origin = float3(uv.x, uv.y, -1.0f);
    ray.Direction = float3(0.0f, 0.0f, 1.0f);
    ray.TMin = 0.01f;
    ray.TMax = 1000.0f;
    
    RayPayload payload = { float4(0, 0, 0, 0) };
    
    // 发射光线
    TraceRay(g_SceneAccelStruct, RAY_FLAG_NONE, 0xFF, 0, 1, 0, ray, payload);
    
    g_Output[dispatchIdx] = payload.color;
}

[shader("miss")]
void MyMissShader(inout RayPayload payload) {
    payload.color = float4(0.2, 0.5, 0.8, 1.0); // 天空色
}
```

## 6. Shader Variant Collection (变体收集器与宏管理)

着色器变体控制是 TA（Technical Artist）优化的重中之重。它不是一种 Shader 语言，而是管理预编译宏组合（`#pragma multi_compile` vs `#pragma shader_feature`）的机制。

- **差异解析**：
    
    - `multi_compile A B C`：无论材质是否使用，打包时都会编译所有组合。常用于全局环境控制（如 `multi_compile _ _MAIN_LIGHT_SHADOWS`）。
        
    - `shader_feature A B C`：打包时会剔除（Strip）没有被任何场景材质引用的变体。常用于材质自身的开关（如 `shader_feature _NORMALMAP`）。
        
- **注意点**：
    
    - **变体爆炸**：$V_{total} = \prod (N_{keyword\_group})$。宏的数量会导致编译时间呈指数级增长。
        
    - **SVC (.shadervariants)**：在游戏启动或场景加载前，利用 `ShaderVariantCollection.WarmUp()` 提前将特定宏组合送入 GPU 驱动进行编译，避免运行时首次遇到该变体导致的卡顿（Shader 编译掉帧）。
        


```glsl
// 在 Shader 中的标准声明方式
Pass {
    // 定义局部关键字，限制在当前 Shader，防止全局关键字超出 256/384 上限 (Unity 2021+)
    #pragma multi_compile_local _ QUALITY_LOW QUALITY_MED QUALITY_HIGH
    #pragma shader_feature_local _ ALBEDO_MAP
    
    // 如果是 SRP 管线，管线自身的宏定义
    #pragma multi_compile _ _MAIN_LIGHT_SHADOWS
    #pragma multi_compile _ _MAIN_LIGHT_SHADOWS_CASCADE
    
    // C# 侧激活方式:
    // material.EnableKeyword("ALBEDO_MAP");
    // Shader.EnableKeyword("QUALITY_HIGH"); // 全局
}
```