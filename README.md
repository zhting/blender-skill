# blender-skill

**3D Art Direction** — 一个 Agent Skill，用于通过 AI 驱动 Blender / Unreal Engine 产出高质量 3D 内容，并按参考图控制最终质量。

> An open-standard Agent Skill (`SKILL.md`) that gives AI agents the art-direction discipline needed to drive Blender and Unreal Engine from a reference image — diagnosis, staged specs, and **numeric verification gates** instead of "does it look good?".

遵循 [Agent Skills 开放标准](https://agentskills.io)，可在 Claude、Claude Code、Cursor、Codex 等任何读取 `SKILL.md` 的代理中使用。

---

## 解决什么问题

用 AI 驱动 3D 制作，瓶颈通常不是"怎么调用 API"——那部分模型本来就会。瓶颈是**判断力**：

- 二十个参数里，哪一个真正决定成败？
- 模型看不准自己渲的图，怎么验证结果对不对？
- 参考图自相矛盾时听谁的？
- 哪些步骤的顺序本身就是对错问题？

这个 Skill 把这些判断固化下来。

### 核心洞察

**AI 对自己产出的画面判断力很弱。**

问「这样够蓬松吗」，答案不可靠；问「透红了没有」，答案可靠。

所以整套方法的核心是——**把主观的艺术判断，尽可能转译成可以用数字或二值判定的检查点**。这不是形式主义，在无法目视验收的前提下，数值门槛是唯一可靠的反馈通道。

| 主观目标 | 客观代理量 |
|---|---|
| "够蓬松" | 包围盒 Z 高度达到设计值 |
| "够密" | 底布改纯红，任何角度透红即不合格 |
| "转轴正不正" | 旋转 45° 前后包围盒中心偏移 < 0.1mm |
| "有没有 CG 感" | 硬边是否产生可见高光线 |
| "和设计图像不像" | 正交剪影 IoU ≥ 0.95 + 质心偏移 |

---

## 五条核心原则

1. **诊断根因，不要修症状** — "嵌墙件像贴纸"的原因是内腔缺明暗梯度，不是深度不够
2. **找出权重最高的那一个参数** — 地毯是 `clump_factor`，抱枕是 `bending_stiffness`，照明是曝光锁定顺序
3. **用数字门槛替代目测** — 每阶段必须有可数值判定的验收项
4. **顺序即正确性** — 锁曝光必须先于调灯；模拟必须先于滚边；设原点必须先于 apply transform
5. **参考图权重分层** — 冲突时按预定优先级裁决，不追求 100% 复刻。**透视图不能用来量尺寸**

另有一节 **工具分工**：Blender / Houdini / 引擎各负责哪一段，以及三条硬规则——模拟输出即成品几何、材质灯光做在交付目标处、数据流单向。

---

## 安装

### Claude Code / Cursor / Codex 等（推荐）

```bash
# 全局安装
git clone https://github.com/zhting/blender-skill.git ~/.claude/skills/3d-art-direction

# 或项目级安装
git clone https://github.com/zhting/blender-skill.git .claude/skills/3d-art-direction
```

> 目录名建议用 `3d-art-direction`（与 `SKILL.md` 中的 `name` 一致），而非仓库名。

### Claude Desktop / Claude.ai

下载 [`dist/3d-art-direction.skill`](dist/3d-art-direction.skill)，在 **Settings → Capabilities → Skills → Upload skill** 上传。

### 验证是否生效

新开一个对话，输入：

```
帮我做一个丝绒沙发的 Blender 建模方案
```

若 Skill 正常触发，回答中应当出现：

- 主动去读 `cloth-sim.md` 与 `fur-and-velvet.md`
- 指出 `bending_stiffness` 是褶皱性格的总开关
- 警告滚边必须在布料模拟**之后**做
- 给出可数值判定的验收门槛，而不是"看起来够软就行"

三项都中，说明触发和内容都对。

---

## 目录结构

```
.
├── SKILL.md                        主文件：五条原则 + 工作流 + 六段式规格结构
├── references/
│   ├── gates.md                    ★ 验收门槛库（写任何方案都应读）
│   ├── reference-matching.md       ★ 按设计图建模：图纸审计 + 剪影 IoU 闭环
│   ├── cloth-sim.md                压力布料模拟：抱枕、床品、软包
│   ├── fur-and-velvet.md           毛发几何 + 绒面着色：地毯、天鹅绒
│   ├── lighting-ue.md              实时引擎照明、色调、体积雾
│   ├── blender-to-ue.md            跨软件交付、导出验证、Groom 限制
│   └── realism-pass.md             真实感优化轮、实时伪影排查
├── assets/
│   └── spec-template.md            执行规格模板，填空即用
└── dist/
    └── 3d-art-direction.skill      预打包，供 Claude Desktop 上传
```

采用渐进式加载：`SKILL.md` 常驻（157 行），`references/` 按任务查阅，不会一次性占满上下文。

---

## 输出规格的六段式结构

Skill 引导代理产出的方案统一采用这个结构（模板见 `assets/spec-template.md`）：

| 段落 | 内容 | 为什么需要 |
|---|---|---|
| 0. 范围与诊断 | 根因、目标特征、**明确排除什么** | 防止执行者顺手加灯、加形变、改相机 |
| 1. 命名规范 | 物体名、材质槽名 | 跨软件交付时材质槽错位是重灾区 |
| 2. 分阶段步骤 | 参数表 + **参数意图说明** | 只给值执行者不敢偏离；说明区间才能自适应 |
| 3. 逐阶段验收 | 可数值判定的门槛 | 唯一可靠的反馈通道 |
| 4. 明确禁止 | 反模式清单 | 比正面指导更能防止翻车 |
| 5. 症状 → 原因对照表 | 出问题时按表定位 | 防止凭感觉乱调参数 |

第 5 段是这套方法的差异点：它把通常只存在于资深美术脑子里的排错知识写了下来。

---

## 覆盖范围

| 领域 | 内容 |
|---|---|
| **软体模拟** | 压力布料、褶皱性格控制、滚边与压线、缝合附件 |
| **毛发与绒面** | 双层结构、簇状收拢、Hair BSDF、Sheen、跨引擎限制 |
| **实时照明** | 曝光锁定顺序、Emissive + 区域光配对、冷暖配比、体积雾、接触阴影 |
| **跨软件交付** | 三项必测、共享枢轴、材质槽风险、Reimport vs Replace References |
| **真实感优化** | 边缘倒角、不完美与非对称、Roughness 破坏、实时伪影排查、性能取舍 |
| **按设计图建模** | 正交/透视判定、像素测量、关键点标注、剪影 IoU、特征清单 |
| **工具分工** | Blender / Houdini / 引擎的职责边界、反模式清单、何时不需要 Houdini |

---

## 已知限制

**这个 Skill 解决不了"AI 看不准自己渲的图"这个根本问题。** 它能做的是固化流程纪律、编码已经踩过的坑、把主观判断转译成客观检查点。

因此：

- 那些数值验收门槛不是形式，是唯一可靠的反馈通道，**不要跳过**
- 遇到无法数值化的判断，Skill 要求代理**报告具体数值而非结论**，最终判断仍需人来做
- 参数值是量级参考，绝对值依赖具体场景（尤其灯光强度依赖曝光基准）

---

## 扩展

最值得扩充的是 `references/gates.md`。新门槛的设计判据：

1. **二值或数值** — "透红了没有"而非"够不够密"
2. **不依赖审美** — 换个执行者结论相同
3. **能在阶段内完成** — 不需要等全流程跑完
4. **失败时指向明确** — 知道该回去改什么

欢迎提 Issue 或 PR 补充新领域的参数表与症状对照表。

---

## License

MIT
