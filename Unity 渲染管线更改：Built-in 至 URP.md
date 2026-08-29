---
title: Unity 渲染管线切换指南 (Built-in 升级至 URP)
date: 2026-08-20
tags:
  - Unity
  - URP
  - Rendering
  - Workflow
---

# Unity 渲染管线切换指南 (Built-in -> URP)

将 Unity 项目从内置渲染管线（Built-in Render Pipeline）升级到通用渲染管线（Universal Render Pipeline, URP）的核心流程与注意事项。

## 1. 核心切换步骤

### Step 1: 安装 URP 包
1. 打开 `Window` > `Package Manager`。
2. 将左上角的包源切换为 **Packages: Unity Registry**。
3. 搜索 `Universal RP` 并点击 **Install** 进行安装。

### Step 2: 创建 URP 配置文件 (Pipeline Asset)
1. 在 `Project` 窗口中，建议新建一个 `Settings` 或 `URP_Profile` 文件夹。
2. 右键 > `Create` > `Rendering` > `Universal Render Pipeline` > `Pipeline Asset (Forward Renderer)`。
3. 这会生成两个文件：
   - **URP Asset**: 主配置文件（控制阴影、抗锯齿、光照质量等）。
   - **URP Renderer Data**: 渲染器数据（用于配置 Forward 渲染管线及添加自定义 Render Feature）。

### Step 3: 全局应用 URP 设置
1. 打开 `Edit` > `Project Settings` > `Graphics`。
2. 在顶部的 **Scriptable Render Pipeline Settings** 槽位中，拖入刚才创建的 **URP Asset**。
3. *(关键检查点)* 切换到左侧的 **Quality** 选项卡，确保各个质量等级（High, Medium 等）的 `Render Pipeline Asset` 没有被旧文件覆盖（可以直接留空以默认继承 Graphics 中的设置）。

> [!success] 状态确认
> 此时场景中原有的 Standard 材质物体会变成“粉红/紫红”色（材质丢失），这说明 URP 已经成功接管了渲染，由于旧 Shader 不兼容 URP 导致渲染报错。

### Step 4: 材质升级 (Material Upgrade)
1. 打开 `Edit` > `Render Pipeline` > `Universal Render Pipeline`。
2. 选择 **Upgrade Project Materials to UniversalRP Materials**。
3. Unity 会自动遍历项目，将所有使用内置 Standard Shader 的材质转换为 URP 兼容的 Lit 材质。

---

## 2. ⚠️ 重要操作注意事项

> [!warning] 升级前务必进行版本控制
> 材质升级是**不可逆**的破坏性操作。在执行 `Upgrade Project Materials` 之前，务必确保已经通过 Git 提交了当前的所有更改，或者备份了工程文件，以防升级出错导致资产损坏。

> [!important] 自定义 Shader 不会被自动升级
> Unity 的自动升级工具 **仅对官方内置 Standard 材质有效**。以下两类 Shader 升级后依然会保持粉红色，需要手动处理：
> 
> 1. **手写 Shader (HLSL/Cg):**
>    - 需要将内置管线的包含文件（如 `#include "UnityCG.cginc"`）替换为 URP 的核心库：`#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"`。
>    - 语法块从 `CGPROGRAM` ... `ENDCG` 更改为 `HLSLPROGRAM` ... `ENDHLSL`。
>    - 光照模型、矩阵宏（如 `UNITY_MATRIX_MVP` 变为 `GetVertexPositionInputs` 或相关 URP 宏）需要根据 URP 架构重写。
> 
> 2. **可视化连线 Shader:**
>    - 如果使用 **Amplify Shader Editor** 等工具制作的 Shader，需要在工具的主面板中，将输出目标（Output/Template）从 `Built-in` 切换为 `Universal` 或 `URP`。
>    - 切换模板后，重新编译并保存 Shader 即可恢复正常显示。

> [!note] 针对多相机的渲染栈 (Camera Stacking)
> URP 不再支持内置管线中通过设置 Camera `Depth` 属性来实现的多相机叠加渲染。
> 如果项目中使用了 UI 相机与主相机分离的架构，在 URP 中需要：
> 1. 将主相机的 Render Type 设置为 `Base`。
> 2. 将 UI 相机（或其他叠加相机）的 Render Type 设置为 `Overlay`。
> 3. 在主相机的 Inspector 中找到 `Stack` 列表，将 Overlay 相机添加进去。

---
## 3. 常见渲染问题排查

- **一切都是黑的，只有天空盒？** -> 检查 Directional Light 的层级，或者 URP Asset 中的 Lighting 选项是否意外关闭。
- **透明物体渲染排序错误？** -> URP Forward 渲染器对透明物体的排序依赖于材质设置。确保 Shader 中的 `RenderQueue` 和 `ZWrite` 设置正确，必要时在 Renderer Data 中添加额外的 Forward 渲染 Pass 来处理特殊层级。