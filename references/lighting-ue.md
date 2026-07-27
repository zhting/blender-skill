# 实时引擎照明（以 Unreal Engine 为例）

原则大多可迁移到其他实时引擎；属性名以 UE5 为准。

---

## 顺序：曝光必须先锁

**在调整任何灯光强度之前锁定曝光。**

自动曝光会随加灯反向补偿：加一盏灯 → 画面整体变暗 → 判断"灯不够亮" → 再加 → 循环。这是照明工作最常见的翻车方式。

```
AutoExposureMethod = Manual
AutoExposureBias   = <记录此值>
```

不支持 Manual 时的退化方案：令 Min/Max Brightness 相等。

**锁定后，所有灯光强度数值都以此曝光为基准。中途改动曝光需报告并复核之前调好的全部灯光。**

---

## 后处理的第一大坑：override 开关

UE 的 Post Process Volume 里，每一个 `Settings.X` 都必须同时把对应的 `bOverride_X` 置为 True，否则写入的值被**静默忽略**。

"改了没反应"九成是这个原因。执行者踩了自己往往发现不了，只会以为数值不对然后乱调。

**在规格里显式提醒这一条。**

---

## 发光体：Emissive + 区域光成对布置

单用 Emissive 材质不够——实时 GI 对 Emissive 的间接光采样弱且噪，会得到"亮但不照亮"的贴纸效果。单用区域光则灯具本体不发光。

**必须成对：**

| 组件 | 职责 | 要点 |
|---|---|---|
| 薄片 Emissive Mesh | 让灯"看得见"、产生光晕 | Emissive Intensity 20～100 |
| 紧贴的 Rect Light | 真正照亮环境 | 颜色与 Emissive 一致 |

Emissive Mesh 的组件上必须启用 `bEmissiveLightSource`，否则 Lumen 完全不计入 GI。

**灯带用 Rect Light 的参数模板：**

```
SourceHeight   = 2~5 (很窄，模拟线性灯带)
BarnDoorAngle  = 20~40   # 必开，防止光往墙里漏
CastShadows    = False   # 装饰灯带关阴影
```

装饰性灯带关阴影：开了既昂贵又容易产生条状伪影，而它们的作用本就是氛围而非塑形。承担塑形任务的灯（主光、台灯）保留阴影。

---

## 色温分工

**冷暖配比比绝对色温更重要。** 典型的"高级感"配置是冷占八成、暖占两成且集中在 2～3 个点。

全屋一个色调会抹平所有层次。暖色越少越集中，对比越强。

用色温驱动（`bUseTemperature`）而非手调 RGB，更容易保持整组灯的一致性。

---

## 找回暗部

实时 GI 不代替接触阴影。三件事分开做：

| 项目 | 作用 | 典型值 |
|---|---|---|
| **AO** | 中尺度遮蔽（家具底部） | Intensity 0.5, Radius 60~100 |
| **Contact Shadows** | 小尺度接触（桌上杯子、相框） | 逐灯 0.02~0.05 |
| **Tonemapper Toe** | 整体暗部压制 | 0.55~0.65 |

家具"浮"在地上通常是缺 AO；小物件"飘着"通常是缺 Contact Shadow。两者覆盖的尺度不同，不能互相替代。

若某灯的 Contact Shadow 出现细长条伪影，把该灯的值降到 0.015。

---

## 体积雾

**没有体积雾的室内是"真空中的房间"**——光从灯到墙瞬间到达，中间空气不存在。

它同时解决"灯像贴纸"和"空间没有深度"两个问题，性价比很高。

```
bEnableVolumetricFog = True
FogDensity           = 0.02~0.05     # 超过 0.06 画面发灰、丢对比度
ScatteringDistribution = 0.7
```

**逐灯设置 `VolumetricScatteringIntensity`（1.0~2.0）。** 只开雾不设这一项，只会得到一层灰蒙蒙的膜，看不到任何光晕。这是最常遗漏的一步。

块状伪影调 `r.VolumetricFog.GridPixelSize`。固定开销约 1～2ms，但收益覆盖全画面。

---

## 反射

- 改动灯光后**必须重建 Reflection Capture**，否则反射里残留旧的环境色
- 摆位：房间中心一个 Box，主要功能区各一个 Sphere
- 玻璃与镜子 Lumen 处理不好，需要 Planar Reflection。**前置条件是项目设置里允许全局裁剪平面，且改动后必须重启编辑器**，否则 Planar Reflection 静默不工作
- Planar Reflection 开销高，每个场景最多两个

---

## 中间状态的心理准备

**关掉主光源、压低环境光之后，画面会暗到看不清。这是正确的中间状态。**

规格里必须写明这一点，否则执行者会自己把环境光加回去，前面的工作全部白做。亮度由后续布置的发光体提供。

---

## 症状 → 原因

| 症状 | 应检查 |
|---|---|
| 改了后处理没反应 | `bOverride_X` 未置 True |
| 加了灯画面反而变暗 | 曝光未锁 |
| 灯带亮但不照亮环境 | 缺配对区域光，或未开 `bEmissiveLightSource` |
| 灯带像贴纸 | Bloom 未生效，或缺体积雾 |
| 开了体积雾但灯没光晕 | 逐灯散射强度未设 |
| 画面发灰、丢对比 | 雾密度过高，或 Tonemapper Toe 太低 |
| 全屋一个色调、没层次 | 冷暖未分工 |
| 家具"浮"在地上 | 缺 AO |
| 小物件"飘着" | 缺 Contact Shadow |
| 反射里有旧颜色残留 | Reflection Capture 未重建 |
| 墙面出现条状伪影 | 装饰灯带误开了 CastShadows |
| Planar Reflection 完全不工作 | 全局裁剪平面未开或未重启编辑器 |
| 灯带周围闪烁斑点 | 纯 Emissive 未配区域光，或 GI 质量不足 |
