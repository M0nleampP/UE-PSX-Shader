# UE-PSX-Shader

Unreal Engine PSX-style shader: 后处理材质实现，用以模拟 PS1 风格的低精度三角形抖动、低分辨率像素块、5-bit 颜色量化与 Bayer 抖动。

该仓库包含一个轻量级的后处理实现思路（不改引擎），可直接用 Material Editor（Post Process 材质）实现。下面是可复制粘贴到材质 Custom 节点的代码、参数与连接步骤。

## 目标
- 在单个后处理材质里实现：低分辨率采样（nearest-like downsample）+ 颜色量化（5-bit）+ Bayer 抖动 + 可选屏幕抖动（wobble）。
- 简单易用：无需修改引擎或写自定义 .usf（高级可选）。

## 兼容性
- 适用于 UE4 / UE5 的 Post Process 材质。不同的引擎版本对 SceneTextureLookup 的支持略有差异：若无法在 Custom 节点中直接调用 SceneTextureLookup，请用材质编辑器内置的 SceneTexture (PostProcessInput0) 节点进行采样，或使用额外的低分辨率 Render Target（Point filter）作为更稳妥的替代。

## 材质设置
- Material Domain: Post Process
- Blendable Location: Before Tonemapper（或 After，根据需求）

### 公开参数（创建为材质参数以便实例化后调整）
- LowRes (Vector2) — 默认 (320,240)
- ScreenSize (Vector2) — 运行时设置为当前视口分辨率（例如 1920,1080）
- ColorLevels (Scalar) — 默认 31（5-bit -> 0..31）
- WobbleStrength (Scalar) — 默认 0.8（像素单位，0=关闭）
- DitherStrength (Scalar) — 默认 1.0
- UseWobble (StaticSwitch) — true/false

## Custom 节点代码（可直接复制到 Custom 节点的 Code 区）

1) NearestLowResUV (输出 float2)
Inputs: UV (float2), LowRes (float2)

```hlsl
float2 pixelUV = UV * LowRes;
pixelUV = floor(pixelUV) + 0.5;
return pixelUV / LowRes;
```

说明：把 UV 对齐到 low-res 像素中心，从而模拟 nearest-downsample。

2) BayerDither (输出 float)
Inputs: PixelPos (float2)  // 一般传 floor(UV * ScreenSize)

```hlsl
int xi = (int)fmod(PixelPos.x, 4.0);
int yi = (int)fmod(PixelPos.y, 4.0);
int idx = yi * 4 + xi;
static const int bayer[16] = {0,8,2,10, 12,4,14,6, 3,11,1,9, 15,7,13,5};
float v = (float)bayer[idx] / 16.0; // 0..0.9375
return v - 0.5; // roughly [-0.5, +0.4375]
```

说明：返回一个小偏移用于在量化阶段破坏条带。

3) QuantizeColor (输出 float3)
Inputs: Color (float3), Levels (float)

```hlsl
return round(Color * Levels) / Levels;
```

说明：将颜色量化到指定级别（Levels = 31 对应 5-bit）。

4) ScreenWobble (输出 float2)
Inputs: UV (float2), ScreenSize (float2), WobbleStrength (float)

```hlsl
float2 pix = UV * ScreenSize;
float n  = frac(sin(dot(pix, float2(12.9898,78.233))) * 43758.5453);
float n2 = frac(sin(dot(pix, float2(93.9898,67.345))) * 12741.2743);
float ox = (n - 0.5) * WobbleStrength;
float oy = (n2 - 0.5) * WobbleStrength;
return UV + float2(ox / ScreenSize.x, oy / ScreenSize.y);
```

说明：基于屏幕像素位置的 cheap-noise，为 UV 添加像素级小偏移以模拟 PSX 顶点精度导致的抖动。

5) SampleSceneAtUV (输出 float4) — UE5.8 推荐实现
Inputs: UV (float2)

```hlsl
// In UE5.8 Post-Process Custom nodes, SceneTextureLookup is available.
// Use the PostProcessInput0 ID for the second parameter. If your build errors,
// fall back to the built-in SceneTexture node in the Material Editor.
return SceneTextureLookup(UV, 14);
```

说明：在 Custom 里直接调用 SceneTextureLookup 可按任意 UV 采样场景纹理（在 UE5.8 的 Post Process 材质中通常可用）。如果编译器报错，改用材质编辑器内置的 SceneTexture (PostProcessInput0) 节点并将 UV 运算连接到该节点的 UV 输入（若该节点在你的版本中无 UV 输入，参考下面的 Render Target 备选方案）。

## 主流程（节点连线）
1. ScreenPosition 节点（取 .xy） => 原始 UV
2. If UseWobble:
   - UV_W = ScreenWobble(UV, ScreenSize, WobbleStrength)
   else UV_W = UV
3. UV_Low = NearestLowResUV(UV_W, LowRes)
4. ColRGBA = SampleSceneAtUV(UV_Low)   // 或用 SceneTexture(PostProcessInput0) 节点
5. Col = ColRGBA.rgb
6. Col_Q = QuantizeColor(Col, ColorLevels)
7. PixelPos = floor(UV_W * ScreenSize) => pass to BayerDither => d = BayerDither * (1.0 / ColorLevels) * DitherStrength
8. OutColor = saturate(Col_Q + d)
9. 连接 OutColor 到 Emissive Color 输出（Alpha = 1）

## UE5.8 额外兼容提示
- SceneTextureLookup 的枚举值（第二参数）在历史上有变化，但在 UE5.x 系列中 14 常用作 PostProcessInput0。若 Custom 节点提示未定义函数或采样异常，请改为：
  - 在材质里使用 SceneTexture 节点（选择 PostProcessInput0），并通过 ScreenPosition 进行采样；或
  - 创建一个低分辨率 Render Target（例如 320x240，Point filter），把场景先渲染到该 RT（使用一个简单的全屏后处理把 PostProcessInput0 复制到 RT），然后在最终后处理材质中采样该 RT（此方式能保证点过滤与完全可控的采样）。

## Blueprint：在运行时设置 ScreenSize（示例步骤）
1. 在你的主场景 Blueprint（例如 GameMode / PlayerController / 一个全局 Manager）里，在 BeginPlay 执行：
   - Get GameViewport -> GetViewportSize (Outputs: X, Y)
   - Create Dynamic Material Instance (Target: the Post Process Material you制作)
   - Set Vector Parameter Value (Material Instance, Parameter Name = "ScreenSize", Value = MakeVector(X, Y, 0))

示例伪节点流程：
- Event BeginPlay -> Get GameViewport -> GetViewportSize -> CreateDynamicMaterialInstance -> SetVectorParameterValue(ScreenSize <- X,Y)

## 精确 nearest 采样的备选方案（更稳妥）
1. 创建一个低分辨率 Render Target（例如 320x240），Texture Filter = Point。
2. 在一个高优先级的后处理（或自定义渲染通道）里把当前 SceneTexture (PostProcessInput0) 画到该 RT（使用 DrawTexture 或一个简单材质复制）。
3. 在你的最终后处理材质中采样这个 RT（它是一个普通的纹理采样节点，可以使用 Point filter），然后对其进行量化与抖动处理。

此方案最能保证 PSX 像素块感，但会略增成本（额外的 RT 和一次复制）。

## 推荐起始参数
- LowRes = (320,240)
- ColorLevels = 31
- WobbleStrength = 0.8
- DitherStrength = 1.0

---
