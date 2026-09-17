
> [!abstract] 核心原则：两层嵌套结构
> 
> GPU 的计算是由“线程组（Thread Group）”和“组内线程（Thread）”共同完成的。
> 
> 要让 GPU 跑起来，必须在 **C# 端**（决定派发多少个组）和 **Shader 端**（决定每个组有多大）分别定义，两者相乘才是实际工作的总线程数。

## 1. 核心概念拆解（盖章理论）

我们可以用“盖印章”来完美对应这两层结构：

- **Shader 端 `[numthreads(x, y, z)]` = 印章本身的尺寸**
    
    - 它定义了**一个线程组**内部包含了多少个微小的执行单元（线程）。
        
    - 例如 `[numthreads(8, 8, 1)]` 表示这个印章是一个 $8 \times 8$ 的正方形，按一次会同时产生 64 个工作线程。
        
- **C# 端 `groupCount` = 盖印章的次数**
    
    - 它定义了针对目标数据，我们需要把这个印章在 X/Y/Z 轴上**盖多少次**。
        
    - 例如 `Dispatch(kernel, 240, 135, 1)` 表示在 X 轴连续盖 240 次，Y 轴连续盖 135 次。
        

> [!danger] 核心易混淆点（必考）
> 
> 误区：以为 Shader 里的 `[numthreads(8, 8, 1)]` 是指在 X 轴上派发 8 个线程组。
> 
> 真相：`numthreads` 永远只管**“一个组内部”**的人数！真正决定在 X 轴上派发多少个组的，是 C# 里的 `groupCountX`。
> 
> 公式：总工作线程数 = groupCount（组数） × numthreads（组内线程数）

## 2. 线程组数量计算公式

在游戏开发中，我们派发 Compute Shader 的目的通常是处理特定维度的数据（如 `texWidth` × `texHeight` 的贴图，或长度为 `N` 的 ComputeBuffer）。

> [!info] 核心计算公式
> 
> 为了确保每一个数据单元都能分配到一个线程，我们需要用**数据总维度**除以**印章尺寸**，并且**必须向上取整**：
> 
> $$GroupCount_X = \lceil \frac{DataWidth}{ThreadPerGroup_X} \rceil$$

**C# 端标准代码范式：**

C#

```C sharp
// 1. 获取目标数据的维度（假设是 1920x1080 的贴图）
int texWidth = 1920;
int texHeight = 1080;

// 2. 这里的数值必须与 Shader 中的 [numthreads(8, 8, 1)] 严格对应
int threadsX = 8;
int threadsY = 8;

// 3. 计算需要派发多少个组（使用 Mathf.CeilToInt 向上取整）
int groupCountX = Mathf.CeilToInt((float)texWidth / threadsX); // 1920 / 8 = 240
int groupCountY = Mathf.CeilToInt((float)texHeight / threadsY); // 1080 / 8 = 135

// 4. 执行派发
computeShader.Dispatch(kernelIndex, groupCountX, groupCountY, 1);
```

## 3. 边界保护机制（向上取整的副作用）

因为数据维度不一定能被 `numthreads` 完美整除，使用向上取整会导致**实际生成的总线程数 > 目标数据量**。

> [!warning] 越界拦截
> 
> 多出来的“溢出线程”如果在 Shader 中去读取数据坐标，就会发生数组越界。因此，必须在 Shader 顶部进行拦截：

**HLSL 端标准代码范式：**

High-level shader language

```hlsl
// 定义印章尺寸
[numthreads(8, 8, 1)]
void CSMain (uint3 id : SV_DispatchThreadID)
{
    // id.x 和 id.y 就是当前线程在全局数据中的绝对坐标
    // 拦截掉超出实际数据维度的多余线程
    if (id.x >= (uint)TextureWidth || id.y >= (uint)TextureHeight)
        return;
        
    // 下方编写安全的着色计算逻辑...
    ResultTexture[id.xy] = float4(1, 0, 0, 1);
}
```

## 4. 常见维度与最佳实践配置

GPU 硬件在底层会将线程打包执行（NVIDIA 为 Warp 32 线程，AMD 为 Wavefront 64 线程）。因此，`numthreads` 的总数（$X \times Y \times Z$）最好是 **64 的倍数**。

| **数据类型**    | **应用场景**           | **Shader 推荐 [numthreads]** | **C# 派发参数 Dispatch** |
| ----------- | ------------------ | -------------------------- | -------------------- |
| **一维 (1D)** | ComputeBuffer、顶点形变 | `[numthreads(64, 1, 1)]`   | `(Count/64, 1, 1)`   |
| **二维 (2D)** | 贴图处理、屏幕后处理         | `[numthreads(8, 8, 1)]`    | `(W/8, H/8, 1)`      |
| **三维 (3D)** | 体积云、SDF 场、3D 纹理    | `[numthreads(4, 4, 4)]`    | `(W/4, H/4, D/4)`    |