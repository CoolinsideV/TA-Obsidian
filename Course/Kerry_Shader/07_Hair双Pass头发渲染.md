---
日期: 2026-08-20
tags:
  - Shader
  - 头发渲染
  - 各向异性
---

> 来源：知乎《基于双Pass的头发渲染》(zhuanlan.zhihu.com/p/1982131863398155170) 同系列《U3D实时渲染 | 角色头发各向异性表达》。文章被反爬拦截无法逐字抓取，此处为等效还原 + 关键代码。

## 一、 为什么要用双 Pass

头发大多用**插片模型**，大量使用**半透明混合**。透明物体不写深度，容易产生排序错误、Overdraw。

双 Pass 的核心目的：**解决半透明头发的渲染次序问题**。

> 第一个 Pass 只渲染背面关闭深度；第二个 Pass 只渲染正面开启深度。由于 Unity 会顺序执行 SubShader 中的各个 Pass，因此可以保证背面总是在正面之前渲染，从而获得正确的深度关系。

| Pass | 剔除 | 深度写入 | 作用 |
|---|---|---|---|
| Pass 1（背面） | `Cull Front` | `ZWrite Off` | 先画背面，输出透光/次表面散射 |
| Pass 2（正面） | `Cull Back` | `ZWrite On` | 后画正面，输出主高光 + 次高光 |

## 二、 双 Pass 结构骨架

```hlsl
SubShader
{
    Tags { "Queue"="Transparent" "RenderType"="Transparent" }

    // Pass 1：只渲染背面，关闭深度写入
    Pass
    {
        Cull Front
        ZWrite Off
        Blend SrcAlpha OneMinusSrcAlpha
        // ... 背面透光
    }

    // Pass 2：只渲染正面，开启深度写入
    Pass
    {
        Cull Back
        ZWrite On
        Blend SrcAlpha OneMinusSrcAlpha
        // ... 正面高光
    }
}
```

## 三、 各向异性高光（Kajiya-Kay）

头发高光不是普通高光，而是**各向异性高光**。核心思想：用发丝**切线方向 T 代替法线 N** 计算高光，从而产生沿发丝方向的条状高光。

```hlsl
float StrandSpecular(float3 T, float3 V, float3 L, float exponent)
{
    float3 H = normalize(L + V);
    float dotTH = dot(T, H);
    float sinTH = sqrt(1.0 - dotTH * dotTH);   // 关键：用 sin 而非 cos，产生条状高光
    float dirAtten = smoothstep(-1.0, 0.0, dotTH);
    return dirAtten * pow(sinTH, exponent);
}
```

## 四、 切线偏移（ShiftTangent）

让高光沿法线方向上下移动，主/次高光可以分开偏移。

```hlsl
float3 ShiftTangent(float3 T, float3 N, float shift)
{
    return normalize(T + N * shift);
}
```

## 五、 双层高光

第一层**主高光**代表直射光直接反射（锐利、靠发梢）；第二层**次高光**带颜色偏移，模拟次表面散射/透光（柔和、靠发根）。

```hlsl
float3 T1 = ShiftTangent(T, N, _Shift1);
float spec1 = StrandSpecular(T1, V, L, _Shininess1);

float3 T2 = ShiftTangent(T, N, _Shift2);
float spec2 = StrandSpecular(T2, V, L, _Shininess2);

fixed3 specular = _SpecColor1.rgb * spec1 + _SpecColor2.rgb * spec2;
```

## 六、 注意点

- **切线方向与 UV**：各向异性依赖切线，但没处理的切线会让高光非常混乱。需要打直 UV、绘制 FlowMap 或通过球体虚拟球心统一切线方向。
- **URP 限制**：URP 鼓励单 Pass 完成所有光照，一个 SubShader 最多走两个 Pass。若要 4-Pass 完整 Scheuermann 方案（Alpha → Opaque → Back → Front），需靠 `RenderObjects` RenderFeature 额外插入。
- 完整内置管线模板见 [[08_Hair双Pass内置管线模板]]。
