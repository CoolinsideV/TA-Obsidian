
> [!abstract] 核心转变：从“平面印章”到“魔方网格”
> 
> 在处理 3D 数据（如 `Texture3D`）时，Shader 里的 `[numthreads(x, y, z)]` 不再是一个二维平面，而变成了一个**“微型魔方”**。
> 
> 例如 `[numthreads(8, 8, 8)]` 代表这个微型魔方内部包含了 $8 \times 8 \times 8 = 512$ 个线程。

## 1. 场景一：生成 3D SDF 场 (三维数据写入)

**SDF（Signed Distance Field）** 的本质是空间中的每一个点（体素），都存储着该点到最近物体表面的**距离**。为了预计算这个场，我们需要往一张 3D 贴图（`Texture3D`）里写入数据。

### 心智模型：遍历体素

假设我们要生成一个 $64 \times 64 \times 64$ 分辨率的 SDF 空间场。

- **Shader 端印章：** `[numthreads(4, 4, 4)]` （每个小组负责 $4 \times 4 \times 4$ 的小区域，共 64 个线程）。
    
- **C# 端派发：** `Dispatch(kernel, 64/4, 64/4, 64/4)`，即 `Dispatch(kernel, 16, 16, 16)`。
    

### HLSL 实现：球体 SDF 数学模型

> [!info] 球体 SDF 公式
> 
> 设空间中一点为 $p$，球心为 $c$，半径为 $r$。则该点的 SDF 值计算公式为：
> 
> $$f(p) = \Vert{}p - c\Vert{} - r$$

High-level shader language

```hlsl
// 定义 3D 贴图作为输出目标 (RW = Read/Write)
RWTexture3D<float> SDFVolume;

// 空间中心与球体半径
float3 SphereCenter;
float SphereRadius;
float VolumeSize; // 假设体素空间大小为 64

[numthreads(4, 4, 4)]
void GenerateSDF (uint3 id : SV_DispatchThreadID)
{
    // 1. 越界保护 (针对 3D 空间的 x, y, z 同时判断)
    if (id.x >= VolumeSize || id.y >= VolumeSize || id.z >= VolumeSize)
        return;

    // 2. 将离散的体素坐标 (0~63) 映射到连续的真实物理空间坐标 (如 -1 到 1)
    // 这里的 id.xyz 就是当前线程在 3D 空间中的绝对坐标！
    float3 worldPos = (id.xyz / VolumeSize) * 2.0 - 1.0; 

    // 3. 计算 SDF 公式：距离 = 当前点到球心的长度 - 半径
    float dist = length(worldPos - SphereCenter) - SphereRadius;

    // 4. 将结果写入 3D 贴图对应的体素中
    SDFVolume[id.xyz] = dist;
}
```

## 2. 场景二：体积云 (Volumetric Clouds) 的“两步走”管线

在体积云的真实开发中，很多人会混淆 3D 噪声的**生成**和体积云的**渲染**。它们对线程的运用完全不同：

### 阶段 A：离线/预计算 3D 噪声贴图 (3D Dispatch)

体积云的形状通常由 Perlin 噪声和 Worley 噪声雕刻而成。这需要一张 3D 纹理。

- **线程调度：** 同上面的 SDF 一样，是真正的 3D 派发。
    
- **配置：** `[numthreads(8, 8, 8)]`，遍历并填满一张 $128 \times 128 \times 128$ 的 3D 噪声贴图。
    

### 阶段 B：屏幕空间光线步进 Raymarching (2D Dispatch)

> [!danger] 核心易混淆点（体积云渲染）
> 
> 误区：以为渲染体积云时，Compute Shader 的调度是 3D 的。
> 
> 真相：实际渲染时，调度是 **2D** 的！因为我们最终是要把云画在**屏幕（2D）**上。

- **线程调度：** `[numthreads(8, 8, 1)]`。
    
- **逻辑：** 每一个线程对应**屏幕上的一个像素**。这个线程会发射一条虚拟射线（Ray），在 `while` 循环中沿着视线方向在刚才生成的 3D 空间中“步进（Marching）”，累加采样到的 3D 噪声密度，最后输出该像素的颜色。
    

High-level shader language

```
// 渲染阶段：这是一个 2D 印章！处理的是屏幕像素！
[numthreads(8, 8, 1)]
void RenderClouds (uint3 id : SV_DispatchThreadID)
{
    // id.xy 是屏幕像素坐标
    // 1. 根据屏幕坐标生成一条射线 (Ray)
    float3 rayOrigin = CameraPos;
    float3 rayDir = GetRayDirection(id.xy);

    // 2. 在线程内部进行 3D 步进 (Raymarching)
    float density = 0;
    float3 currentPos = rayOrigin;
    
    for (int i = 0; i < 64; i++) // 步进 64 次
    {
        // 去读取阶段 A 生成的 3D 数据
        density += Noise3DTexture.SampleLevel(sampler, currentPos, 0).r;
        currentPos += rayDir * StepSize;
    }

    // 3. 将结果输出到屏幕贴图
    ResultScreen[id.xy] = float4(1, 1, 1, density);
}
```