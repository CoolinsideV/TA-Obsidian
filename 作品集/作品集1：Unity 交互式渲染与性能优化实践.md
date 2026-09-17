
五个案例恰好覆盖了从基础的“顶点控制”**、**“像素着色”**、**“空间坐标系转换”**，到进阶的**“光照模型计算”**和**“底层性能优化”，能全面展示你对渲染管线、数学原理以及引擎底层逻辑的掌控力。

为了让作品集兼具技术深度与展示度，建议将项目托管在 GitHub 上，并利用 Markdown 编写图文并茂的 README（你可以直接在 Obsidian 中组织这些文档，最后同步到仓库）。

以下是为你规划的作品集架构及逐个实现指南：


### 案例一：基于世界坐标交互的动态雪地/草地 (Vertex Displacement)

**展示重点**：C# 与顶点着色器的通信、平滑函数（Smoothstep）的应用、法线方向的向量运算。

- **实现步骤**：
    
    1. **C# 逻辑**：在玩家角色上挂载脚本，每帧获取 `transform.position`，并通过 `Material.SetVector("_PlayerPos", playerPosition)` 发送给地面的材质。
        
    2. **HLSL/Shader 面板**：在顶点着色阶段接收 `_PlayerPos`。
        
    3. **数学计算**：计算当前顶点世界坐标与玩家坐标的距离 `distance = length(vertexWorldPos - _PlayerPos)`。
        
    4. **形变逻辑**：使用 `smoothstep(radius, radius - edge, distance)` 圈出一个受影响的范围。将这个范围的值域乘以一个深度控制参数，然后沿着顶点的法线方向向下（雪地）或向四周（草地排开）偏移顶点位置。
        
- **作品集亮点**：录制一段角色走过雪地留下凹痕的动图。文档中可以展示顶点偏移的核心代码，并附带你在 Amplify Shader Editor 中的节点连线图或手写 HLSL 代码。
    

### 案例二：状态驱动的科幻受击护盾 (Fresnel Effect)

**展示重点**：点乘运算、菲涅尔效应、通过 C# 浮点数控制着色器表现。

- **实现步骤**：
    
    1. **C# 逻辑**：利用射线检测（Raycast），当击中护盾时，向材质传递两个参数：击中点的坐标 `_HitPos`，以及一个随时间递减的浮点数 `_HitIntensity`（模拟受击余波）。
        
    2. **HLSL 核心计算**：
        
        - **边缘发光**：计算视线方向 $V$ 与法线 $N$ 的点积，套用经典菲涅尔公式：$F = F_0 + (1 - F_0)(1 - V \cdot N)^5$。
            
        - **受击波纹**：在像素着色器中计算当前像素距 `_HitPos` 的距离，结合正弦函数 $\sin(distance \times frequency - time)$ 制作涟漪效果，并乘以传入的 `_HitIntensity` 作为蒙版。
            
    3. **混合**：将波纹特效叠加在基础的菲涅尔护盾之上。
        
- **作品集亮点**：展示护盾在不同受击频率下的颜色渐变与能量波纹扩散，体现你对视觉反馈与逻辑挂钩的理解。
    

### 案例三：基于切线空间的视差映射地表 (Parallax Mapping & TBN)

**展示重点**：矩阵变换、TBN 空间理解、光线步进（Raymarching）思想的雏形。

- **实现步骤**：
    
    1. **准备阶段**：准备一张带有深度/高度信息的纹理（Height Map）。
        
    2. **空间转换**：在顶点着色器中，构建 TBN 矩阵。将摄像机的视线方向（View Direction）从世界空间转换到**切线空间（Tangent Space）**，然后传递给片段着色器。
        
    3. **UV 偏移计算**：在片段着色器中，基于切线空间的视线方向提取 XY 分量。根据高度图采样得到的高度值，计算出 UV 的偏移量：`offset = viewDir.xy * height * scale`。
        
    4. **进阶（陡峭视差/POM）**：如果不满足于简单偏移，可以在 Shader 中写一个 `for` 循环，将视线在深度方向上分层步进，直到找到高度图与视线相交的点，这能展现出极强的立体感。
        
- **作品集亮点**：在文档中利用 LaTeX 详细推导一下 TBN 矩阵的构建过程，对比开启/关闭视差映射时的砖块路面立体感差异。
    

### 案例四：自定义多光源 Blinn-Phong 光照模型 (Custom Lighting & Radiometry)

**展示重点**：脱离引擎默认光照、缓冲区数据传递、对光度学概念的理解。

- **实现步骤**：
    
    1. **C# 光源管理器**：创建一个脚本收集场景中所有自定义光源的数据（位置、颜色、辐射强度/Radius）。将这些数据存入 `ComputeBuffer` 或使用 `Shader.SetGlobalVectorArray` 传给 GPU。
        
    2. **HLSL 光照循环**：在 Shader 中声明对应的 `StructuredBuffer`。在片段着色器中写一个循环，遍历所有光源。
        
    3. **辐射度量学计算**：引入平方反比衰减法则（Inverse Square Falloff）。针对每个光源计算漫反射与高光。高光部分使用 Blinn-Phong 模型，计算半程向量 $H = \frac{L + V}{\vert{}\vert{}L + V\vert{}\vert{}}$，然后求 $(N \cdot H)^{power}$。
        
    4. **累加**：将所有光源的贡献进行累加（积分思想的离散体现），得到最终像素颜色。
        
- **作品集亮点**：展示一个场景中拥有几十个动态彩色点光源的流畅运行画面，并在文档中写明你是如何推导并实现这个光照衰减公式的。
    

### 案例五：基于 GPU Instancing 的海量草海渲染 (Performance & ECS Architecture)

**展示重点**：底层 API 调用、Draw Call 优化、面向数据编程思想。

- **实现步骤**：
    
    1. **C# 数据构建**：摒弃传统的 GameObject 挂载 MeshRenderer。在 C# 中生成一万个 `Matrix4x4` 矩阵，代表一万棵草的位置、旋转和缩放。
        
    2. **API 调用**：使用 `Graphics.DrawMeshInstanced`（或更底层的 `DrawMeshInstancedIndirect`），将同一个草的 Mesh、相同的 Material 和这包含一万个矩阵的数组直接推给 GPU。
        
    3. **Shader 配置**：在 HLSL 顶部添加 `#pragma multi_compile_instancing`。在结构体中使用 `UNITY_VERTEX_INPUT_INSTANCE_ID` 宏。
        
    4. **矩阵读取**：GPU 渲染每一根草时，会自动通过 Instance ID 去显存中抓取对应的变换矩阵，完成世界坐标转换。
        
- **作品集亮点**：放上 Unity Profiler 的截图，对比“实例化前（一万个 Draw Call，严重卡顿）”与“实例化后（1个 Draw Call，高帧率）”的性能数据。这是极其硬核且受面试官欢迎的加分项。
