# UE5.8 Custom Node Fields — PSX Post-Process Material

这个文件列出在 UE5.8 的 Material Editor 里创建每个 Custom 节点时需要填写的字段：Output 类型、按顺序的 Input 名称与类型，以及可直接复制粘贴到 Custom 节点 Code 区的 HLSL 代码。

注意：在材质编辑器中新建 Custom 节点后，右侧 Inputs 列表里逐行添加输入：Name（例如 UV）和 Type（Float/Float2/Float3/Float4）。输入顺序必须和 Code 中变量使用顺序一致，名字区分大小写。

---

| 节点名 (Node Name) | Output 类型 | Inputs（按顺序 — 名称:类型） | 备注 |
|---|---:|---|---|
| NearestLowResUV | Float2 | UV:Float2, LowRes:Float2 | 将 UV 对齐到 low-res 像素中心（nearest-downsample）。 |
| ScreenWobble | Float2 | UV:Float2, ScreenSize:Float2, WobbleStrength:Float | 基于像素位置的 cheap-noise，为 UV 加微偏移（像素单位的强度由 WobbleStrength 控制）。 |
| QuantizeColor | Float3 | Color:Float3, Levels:Float | 对 RGB 三通道按 Levels 做 round -> quantize（Levels = 31 对应 5-bit）。 |
| BayerDither | Float | PixelPos:Float2 | 4×4 Bayer 矩阵返回单一偏移值（用于量化抖动）。PixelPos 通常为 floor(UV * ScreenSize)。 |
| SampleSceneAtUV | Float4 | UV:Float2 | UE5.8 推荐：SceneTextureLookup(UV, 14) -> PostProcessInput0。若不可用，用 SceneTexture 节点替代。 |

---

下面是每个 Custom 节点的可直接复制粘贴的 HLSL 代码（将整段代码粘到 Custom 节点的 Code 区；并在 Inputs 区按表格创建对应输入）：

NearestLowResUV (Output = Float2) — Inputs: UV:Float2, LowRes:Float2
```hlsl
float2 pixelUV = UV * LowRes;
pixelUV = floor(pixelUV) + 0.5;
return pixelUV / LowRes;
```

ScreenWobble (Output = Float2) — Inputs: UV:Float2, ScreenSize:Float2, WobbleStrength:Float
```hlsl
float2 pix = UV * ScreenSize;
float n  = frac(sin(dot(pix, float2(12.9898,78.233))) * 43758.5453);
float n2 = frac(sin(dot(pix, float2(93.9898,67.345))) * 12741.2743);
float ox = (n - 0.5) * WobbleStrength;
float oy = (n2 - 0.5) * WobbleStrength;
return UV + float2(ox / ScreenSize.x, oy / ScreenSize.y);
```

QuantizeColor (Output = Float3) — Inputs: Color:Float3, Levels:Float
```hlsl
return round(Color * Levels) / Levels;
```

BayerDither (Output = Float) — Inputs: PixelPos:Float2
```hlsl
int xi = (int)fmod(PixelPos.x, 4.0);
int yi = (int)fmod(PixelPos.y, 4.0);
int idx = yi * 4 + xi;
static const int bayer[16] = {0,8,2,10, 12,4,14,6, 3,11,1,9, 15,7,13,5};
float v = (float)bayer[idx] / 16.0; // 0..0.9375
return v - 0.5; // roughly [-0.5, +0.4375]
```

SampleSceneAtUV (Output = Float4) — Inputs: UV:Float2
```hlsl
// UE5.8 推荐写法：采样 PostProcessInput0（14 通常对应 PostProcessInput0）
return SceneTextureLookup(UV, 14);
```

兼容性提示：
- 如果你的 UE5.8 构建在 Custom 节点里不能调用 SceneTextureLookup（少数自定义构建或策略限制），请改用材质编辑器内置的 SceneTexture 节点（选择 PostProcessInput0）并把 UV_Low 连接到其 UV 输入；若该节点没有 UV 输入，请采用 Render Target 方案（在 README 和 UE5.8_FULL_SETUP.md 中已有说明）。
- Input 名称必须与 Code 中变量一致（大小写敏感）。Input 的顺序必须与 Code 中变量使用顺序一致。

调试小技巧：
- 先把 WobbleStrength 设为 0，只观察 LowRes + Quantize + Dither 的效果；确认无误后再打开 Wobble。
- 若看到模糊或线性混合，尝试将流程改为 RT（320×240，Point filter）。

---

如果你希望我把该文件加入仓库中的特定目录（例如 docs/ 或 examples/），告诉我路径，我可以移动或复制该文件并提交。