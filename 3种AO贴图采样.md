## 1. AO 贴图采样（最常用，最适合填进 `SurfaceData`）

AO 贴图是一张灰度图（美术烘焙或手绘），运行时直接 `tex2D` 采样即可。通常取 **r 通道**。

```hlsl
// 声明贴图（放在 CBUFFER 之外的全局区）
sampler2D _OcclusionMap;   // AO 灰度贴图

// 填充 SurfaceData 时采样
SurfaceData FillSurfaceData(float2 uv)
{
    SurfaceData surface;
    surface.albedo    = tex2D(_AlbedoTex, uv).rgb;
    surface.normal    = UnpackNormal(tex2D(_NormalTex, uv)); // 法线贴图
    surface.roughness = tex2D(_RoughnessTex, uv).r;          // 粗糙度（r 通道）
    surface.metallic  = tex2D(_MetallicTex, uv).r;           // 金属度（r 通道）

    // AO：灰度图，取 r 通道
    surface.occlusion = tex2D(_OcclusionMap, uv).r;

    return surface;
}
```

> ⚠️ **通道打包提醒**：Unity 标准的 Metallic 贴图（`_MetallicGlossMap`）里，**R = 金属度，G = AO**，AO 是塞在 g 通道的。如果你用的是这种打包贴图，就要改成：
> 
> ```hlsl
> surface.metallic  = tex2D(_MetallicGlossMap, uv).r;   // R：金属度
> surface.occlusion = tex2D(_MetallicGlossMap, uv).g;   // G：AO
> ```

---

## 2. SSAO（屏幕空间 AO，运行时算）

SSAO **不是在光照 shader 里算的**，它是一个独立的**后处理 Pass**：先拿深度+法线缓冲，在屏幕空间对每个像素做方向采样，算出「周围有多少几何遮挡」，渲染成一张 AO 图，再供光照采样。流程分两步：

**第一步：后处理 Pass 生成 SSAO 图（示意，实际是整屏计算）**

```hlsl
// 伪代码：屏幕空间每个像素，沿随机方向采样深度判断遮挡
half ComputeSSAO(float2 screenUV)
{
    float depth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, screenUV);
    float3 posWS = ReconstructWorldPos(screenUV, depth);
    float3 normalWS = SAMPLE_TEXTURE2D(_CameraNormalsTexture, screenUV).xyz;

    half ao = 0;
    for (int i = 0; i < _SampleCount; i++)
    {
        // 在法线半球内取随机方向，采样偏移点
        float2 offset = _SampleKernel[i].xy * _Radius;
        float2 sampleUV = screenUV + offset;

        float sampleDepth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, sampleUV);
        float3 samplePosWS = ReconstructWorldPos(sampleUV, sampleDepth);

        // 采样点比表面更「深」→ 说明被周围几何挡住 → AO 变暗
        float occluded = (samplePosWS - posWS);   // 简化示意
        ao += saturate(occluded);
    }
    return 1.0 - ao / _SampleCount;   // 1 = 无遮挡，0 = 全黑
}

// 输出到一张 _SSAOTexture，供光照 Pass 使用
```

**第二步：光照 shader 里采样这张 SSAO 图**

```hlsl
// SSAO 结果存在 _SSAOTexture 里，光照时用屏幕坐标采样
half4 SSAOFragment(Varyings input) : SV_Target
{
    float2 screenUV = input.positionCS.xy / _ScreenParams.xy;  // 屏幕 UV

    SurfaceData surface = FillSurfaceData(input.uv);
    surface.occlusion = SAMPLE_TEXTURE2D(_SSAOTexture, sampler_SSAOTexture, screenUV).r;

    // 之后正常走 EvaluateRenderingEquation
    float3 Lo = EvaluateRenderingEquation(surface, lights);
    return half4(Lo, 1);
}
```

> 关键区别：**AO 贴图是 `input.uv`（模型 UV）采样，SSAO 是 `screenUV`（屏幕坐标）采样**。

---

## 3. Lightmap AO（烘焙进光照贴图，静态物体用）

烘焙 GI 时，Unity 可以顺带把「这个静态表面被周围挡住多少环境光」也烤进光照贴图。运行时不用自己算，直接从 lightmap 里拿。URP 里通过 `SampleLightmap` 或 `SAMPLE_GI` 宏统一处理，AO 通常存在 lightmap 的**绿(g) 通道**里。

```hlsl
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

// URP 里，光照贴图的 AO 和颜色一起编码在 lightmap 里
half3 GetBakedGI(float2 lightmapUV, half3 normalWS, out half bakedAO)
{
    // SampleLightmap 采样出烘焙的颜色（含间接光）
    // AO 常打包在 lightmap 的 .g 通道（具体通道因管线/版本而异）
    half4 bakedColor = SampleLightmap(lightmapUV, normalWS);

    bakedAO = bakedColor.g;      // 示意：AO 在 g 通道
    return bakedColor.rgb;       // 烘焙的漫反射 GI 颜色
}
```

更简单、更推荐的做法：直接用 URP 的 `SAMPLE_GI` 宏，它内部已经帮你把 lightmap AO 和 SH 探针都处理好了：

```hlsl
half4 LightFragment(Varyings input) : SV_Target
{
    SurfaceData surface = FillSurfaceData(input.uv);

    // SAMPLE_GI 一次性处理：光照贴图 / SH 探针 / 烘焙 AO
    half3 bakedGI;
    half bakedAO;
    SAMPLE_GI(input.lightmapUV, input.sh, surface.normal, bakedGI, bakedAO);

    surface.occlusion = bakedAO;   // 用烘焙的 AO

    float3 Lo = EvaluateRenderingEquation(surface, lights);
    return half4(Lo, 1);
}
```

> ⚠️ 说明：`SAMPLE_GI` 的签名（是否带 `bakedAO` 参数、AO 存在哪个通道）在 URP 不同版本间有差异。这个示例是**概念正确**的写法，实际工程里要以你安装的 URP 版本的 `Lighting.hlsl` 为准（项目里 `Packages/.../Lighting.hlsl` 没被拷进工程，我这边读不到，所以没法给你逐字核对）。

---

## 三种来源一句话对比

|来源|采样坐标|谁算的|适用|
|---|---|---|---|
|**AO 贴图**|模型 `uv`|美术烘焙/手绘|静态+动态都行，最通用|
|**SSAO**|屏幕 `screenUV`|运行时后处理|动态物体，实时但有小噪点|
|**Lightmap AO**|光照贴图 `lightmapUV`|烘焙 GI 时|仅静态物体|

其中 **AO 贴图**最贴合你现在 `Code.shader` 里的 `SurfaceData` 写法，也是你最容易直接落地的。要不要我帮你把 AO 贴图采样完整接进 `Code.shader`，做成能跑的真实版本？