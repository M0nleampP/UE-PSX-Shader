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

5) （可选）SampleSceneAtUV (输出 float4)
Inputs: UV (float2)

```hlsl
// 在某些版本中可以：
return SceneTextureLookup(UV, 14); // 14 = PostProcessInput0 （引擎版本差异）
```

说明：在 Custom 里直接调用 SceneTextureLookup 在不同 UE 版本中可能不可用；若不可用，用材质编辑器内置 SceneTexture(PostProcessInput0)节点并把 UV 运算后的结果连到其 UV 输入。

## 主流程（节点连线）
1. ScreenPosition 节点（取 .xy） => 原始 UV
2. If UseWobble:
   - UV_W = ScreenWobble(UV, ScreenSize, WobbleStrength)
   else UV_W = UV
3. UV_Low = NearestLowResUV(UV_W, LowRes)
4. ColRGBA = SceneTexture(PostProcessInput0) sampled at UV_Low
5. Col = ColRGBA.rgb
6. Col_Q = QuantizeColor(Col, ColorLevels)
7. PixelPos = floor(UV_W * ScreenSize) => pass to BayerDither => d = BayerDither * (1.0 / ColorLevels) * DitherStrength
8. OutColor = saturate(Col_Q + d)
9. 连接 OutColor 到 Emissive Color 输出（Alpha = 1）

## 注意事项与可选优化
- 精确 nearest 采样：材质内对 UV 做 floor 近似 nearest-downsample，但若想保证点过滤（Point）与最准确的像素对齐，建议在引擎中额外创建一个低分辨率 Render Target（比如 320x240），把场景渲染到该 RT（设置 Texture Filter = Point），然后在后处理里直接采样该 RT。代价更高但结果更一致。
- 若 SceneTextureLookup / SceneTexture 节点无法按预期工作，请告知你的 UE 版本，我会提供针对该版本的替代实现。
- 若想更真实地模拟“无透视贴图插值（affine）”的外观，需要在顶点阶段对 clip-space 顶点做 snap（vertex-snapping），这需要引擎层面的自定义 shader 或把 WorldPositionOffset 作为近似实现——那属于高级方案，不在本 README 的后处理单 pass 范围内。

## 在游戏中设置 ScreenSize
建议在 BeginPlay 或每次分辨率变化时通过 Blueprint 更新材质实例参数：
- 在 BP 的 BeginPlay 中使用 `Get Viewport Size`，然后 `Set Vector Parameter Value`（ScreenSize）到材质实例。

## 推荐起始参数
- LowRes = (320,240)
- ColorLevels = 31
- WobbleStrength = 0.8
- DitherStrength = 1.0

---

如果你愿意，我可以：
- 把一个完整的 Material Instance 和示例 Blueprint 添加到仓库（需要 UE 内容导出或示例工程文件）；
- 或者把后处理 + 可选 RenderTarget 实现的详细 Blueprint/步骤写成单独文档并提交。

要我把 Material Asset / 示例 Blueprint 放进仓库吗？