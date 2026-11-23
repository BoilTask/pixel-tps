# AGENTS.md — SUNFALL / ZERO PROTOCOL

> 本文件是本仓库的**正式项目管理文档**，也是所有 Agent 与协作者的第一入口。
> 动手改代码之前先读它。本文件与 `README.md` 冲突时，以本文件为准（README 面向玩家，AGENTS 面向维护者）。

---

## 1. 项目定位

**SUNFALL / ZERO PROTOCOL** 是一个可玩的**像素风第三人称射击（TPS）最小验证 Demo**。

- 单文件 HTML，零外部依赖：像素美术、场景、角色、HUD、音效全部由 Canvas 2D + Web Audio 在运行时程序化生成。
- 内部分辨率固定 `384×216`，最近邻放大，桌面端整数倍缩放（最大 6x），触屏端等比缩放。
- 目标不是做完整游戏，而是验证**越肩相机 + 射击手感 + 波次压力 + 双端输入**这条体验链路。

**它不是**：产品级游戏、引擎工程、可联网项目、需要构建步骤的前端工程。

---

## 2. 唯一维护目录

| 用途 | 路径 |
|---|---|
| **唯一活跃仓库** | `C:\pixel-tps`（本目录） |
| 远端仓库 | <https://github.com/BoilTask/pixel-tps>（默认分支 `main`） |
| 迁移来源（**已冻结，禁止编辑**） | `C:\Users\Admin\Documents\deepseek-harness\default-workspace\pixel-tps-demo` |

**规则**：所有改动只发生在 `C:\pixel-tps`。旧目录仅作历史备份，不要在其中新增、修改或部署任何文件。若发现两处内容不一致，唯一可信来源是 `C:\pixel-tps` 的 git 历史。

---

## 3. 五分钟上手

无需安装依赖、无需 `npm install`、无构建步骤。

```powershell
# 方式 A：直接打开（最快）
start C:\pixel-tps\index.html

# 方式 B：本地 HTTP 服务（推荐用于验收，指针锁定 / 全屏 API 在 file:// 下行为不一致）
npx --yes serve C:\pixel-tps -l 4321
# 然后访问 http://localhost:4321
```

**调试用 URL 参数**（用于强制布局，验收时两端都要过）：

- `?desktop=1` — 强制桌面布局
- `?touch=1` — 强制触屏布局（虚拟摇杆 / 开火键 / 旋转提示）

触屏模式默认按设备能力自动判定：`(pointer: coarse)` 或 `maxTouchPoints > 0 && (hover: none)`。

---

## 4. 仓库结构

```
C:\pixel-tps\
├── index.html             # 全部实现：HTML + CSS + JS（单文件，约 5214 行）
├── README.md              # 玩家向说明：操作表、验证内容、原型边界
├── cloudflare.config.ts   # Cloudflare Workers 部署配置（cf CLI 读取）
├── preview.png            # 桌面实际游玩画面（README 引用）
├── mobile-preview.png     # 手机横屏触控布局（README 引用）
├── fullscreen-preview.png # 开始页全屏引导（README 引用）
├── AGENTS.md              # 本文件
├── .gitignore             # 忽略 dist/、.cloudflare/ 等生成物
└── .gitattributes         # 仅声明二进制资源，不做换行转换
```

**受控生成物**：`dist/`（发布暂存目录，见第 9 节）与 `.cloudflare/`（部署时生成的 Build Output）都已写入 `.gitignore`，**永不提交**。


---

## 5. `index.html` 代码地图

行号是**锚点**（会随修改漂移）。定位时用注释标题 grep，比记行号可靠：

```powershell
Select-String -Path index.html -Pattern '^\s*// -{3,}' -Encoding UTF8
```

| 行 | 区块 | 职责 |
|---:|---|---|
| 9 | `<style>` | 全部 CSS（含 320 行的 touch/mobile 段） |
| 534 | `<body>` | DOM：`#frame`、`#game`，菜单/暂停/结算、HUD、触屏控件、旋转提示 |
| 647 | `<script>` | IIFE，`'use strict'`，全部逻辑作用域 |
| 651 | 总注释 | 说明 384×216 + 最近邻 + 无外部资源 |
| 719 | fullscreen | 全屏 API、`is-fullscreen` 状态、降级提示（Safari / 添加到主屏幕） |
| 771 | math / colour | 数学、颜色、像素抖动辅助 |
| 860 | bitmap font | 位图字体，HUD 文本渲染 |
| 968 | sprites | 程序化像素精灵（角色含空中收腿帧、敌人、枪、道具） |
| 1372 | audio | Web Audio 音效合成，需一次用户手势解锁 |
| 1537 | world | 场景/掩体/世界数据 `WORLD`；含货台、楼梯、集装箱攀爬路线 |
| 1722 | game state | `game` 对象、波次、计分、玩家状态（`y/vy/grounded/climb`） |
| 1819 | projection / camera | 越肩相机、肩部偏移、ADS 拉近、垂直阻尼跟随、投影 `project()` |
| 2000 | collision / rays | 高度感知碰撞（`bodyBlocked` / `surfaceBelow`）、射线 `castWorldRay()`、视线检测 |
| 2170 | effects | 弹着火花、tracer、屏幕震动、伤害数字 |
| 2216 | shooting / enemies | 射击判定（hitscan / 爆头）、跳跃/攀爬入口、敌人 AI、弹丸、波次生成 |
| 3074 | rendering: sky | 天空 |
| 3250 | rendering: ground | 地面、投影阴影（按脚下支撑面） |
| 3487 | rendering: boxes | 掩体方块（含 `deck` 货台样式与细节） |
| 3692 | rendering: billboards / effects | billboard 精灵按深度排序、特效层 |
| 4128 | HUD | 雷达、生命、弹药、波次、准星与攀爬提示（4333 生命区 / 4370 计分区） |
| 4514 | full-screen effects | CRT 扫描线、暗角、受击闪红 |
| 4619 | main render | 单帧渲染总装 |
| 4642 | UI / state | 状态机 `menu / playing / paused / gameover`、开始/暂停/结算切换 |
| 4808 | input | 键鼠：WASD、左键、右键 ADS、Space 跳跃/攀爬、Shift、R、Esc、M、Enter、指针锁定与右键拖动降级 |
| 4911 | touch controls | 虚拟摇杆、右半屏滑动、FIRE/AIM/JUMP/DASH/RELOAD/II，pointer capture |
| 5157 | loop | `requestAnimationFrame` 主循环 + `window.SUNFALL` 调试句柄 |

### 垂直移动契约

改地形/移动前先看这一段，否则很容易做出"人卡在台子里"的 bug：

- 玩家是**圆柱 + 脚底高度**：`p.y` 是脚底，身高 `BODY_H = 1.72`。所有落地判定都走 `surfaceBelow(x, z, maxY)`。
- 水平碰撞走 `resolveCircle(ent, r, feetY, stepUp)`：与身体区间不重叠的方块不再推人，所以站在箱顶上不会再被同一个箱子弹开。
- `STEP_UP = 0.34` 是自动上台阶上限（楼梯踏步按 0.28–0.3 设计）；`STEP_DOWN = 0.42` 以内走下台阶不进入下落状态。
- `CLIMB_REACH = 2.05` 是可攀爬的台沿高度；`buildWorld()` 里 `climbable` 由 `type !== 'wall'` 决定，**外围墙永远不可攀爬**（它是场地边界）。
- 敌人不参与这套系统（`y` 恒为 0、不传 feetY），所以只能在地面绕行；新增抬高的可站立平台时记得别把敌人出生点放上去（`findSpawn` 已按 `surfaceBelow` 过滤）。

### 自动化调试句柄

文件末尾挂载了 `window.SUNFALL`，可在 DevTools / CDP 中驱动验收：

```js
SUNFALL.start()          // 进入 playing
SUNFALL.state()           // 'menu' | 'playing' | 'paused' | 'gameover'
SUNFALL.pause(); SUNFALL.resume(); SUNFALL.endGame()
SUNFALL.castWorldRay(...); SUNFALL.project(...); SUNFALL.surfaceBelow(...)
SUNFALL.jump(); SUNFALL.mantle()   // 手动触发跳跃 / 攀爬探测
// 另外暴露：game, cam, WORLD, SPRITES, sound
```

---

## 6. 硬性技术约束（不可违反）

1. **保持单文件**。所有 HTML/CSS/JS 都在 `index.html` 内。不要拆分文件、不要引入打包器、不要加 `package.json` 依赖。
2. **零外部资源**。不允许出现 `<script src>`、`<link rel="stylesheet">`、外部图片/字体/音频、`fetch()`、`import`（当前已验证：仅有一个内联 SVG data-URI favicon）。新增美术与音效必须**程序化生成**。
3. **不改变内部分辨率契约**。渲染仍在 `384×216` 画布上完成，再最近邻放大；不要在逻辑坐标里直接写屏幕像素。
4. **编码与换行**：全仓库 **UTF-8 无 BOM + LF**。本仓库已设 `core.autocrlf=false`，**不要**改回去，也不要添加 `* text=auto eol=...` 之类会改写字节的规则。
   - Windows PowerShell 5.1 的 `Get-Content` 默认按 GBK 读取，会把中文显示成乱码。核对内容时用 `-Encoding UTF8` 或 `[System.IO.File]::ReadAllText($p, [Text.Encoding]::UTF8)`。**乱码是工具问题，不要因此"修复"文件编码。**
5. **不要整文件重排**。`index.html` 是一个 4800+ 行的紧凑文件，禁止全局格式化、重新缩进或批量改引号；改动必须是最小 diff，方便 review 与回滚。
6. **触屏与桌面必须同时成立**。任何输入/布局改动都要在 `?desktop=1` 与 `?touch=1` 下各验一遍。

---

## 7. 修改工作流

```text
1. 读 AGENTS.md（本文件）→ 2. git status / git log 确认起点干净
3. 用注释标题定位区块（第 5 节）→ 4. 做最小 diff
5. 本地双端验证（第 8 节清单）→ 6. 若玩家可见变化：更新 README + 预览图
7. 按第 10 节规范提交 → 8. 需要上线时执行第 9 节部署
```

**同步文档的判定标准**：

| 改动类型 | 必须同步 |
|---|---|
| 新增/修改操作键位、触屏按钮 | `README.md` 操作表 |
| 视觉、UI、HUD 明显变化 | 重截对应预览图（`preview.png` / `mobile-preview.png` / `fullscreen-preview.png`） |
| 增删能力、改变原型边界 | `README.md`「这个 Demo 验证的内容」「原型边界」 |
| 改部署目标、Worker 名、域名 | `cloudflare.config.ts` + 本文件第 9 节 |

---

## 8. 验证清单

提交前至少完成以下项，并把结论写进 commit body 或 PR 描述：

- [ ] 页面加载无 console 报错（含未捕获异常、资源 404）
- [ ] `?desktop=1`：WASD 移动、鼠标转向、左键连发、右键 ADS、Shift 冲刺、Space 跳跃、R 换弹、Esc 暂停、M 静音、Enter 快速开始
- [ ] `?touch=1`：虚拟摇杆移动、右半屏滑动转向、FIRE 按住连发且可滑动瞄准、AIM / JUMP / DASH / RELOAD / II 全部生效
- [ ] 立体地形：走楼梯上货台（0.3→0.6→0.9→1.2→1.5）、贴台沿按跳跃能翻上去、矮箱→集装箱两段攀爬、从台面走落/跳落回地面不卡住
- [ ] 状态机完整走通：`menu → playing → paused → playing`，以及死亡 `→ gameover →` 重开
- [ ] 波次推进、击杀计分、换弹、冲刺无敌帧、生命归零结算均符合预期
- [ ] 竖屏显示旋转提示（ROTATE TO PLAY）
- [ ] 全屏：进入/退出正常；API 不可用时给出可操作降级提示，不阻塞游戏
- [ ] 音频：首次交互后可播放，M 键静音生效
- [ ] 改动前后 `git diff` 只包含预期行

自动化辅助：用 `window.SUNFALL`（见第 5 节）跳过手动点击直接驱动状态，再做断言。

---

## 9. 构建与部署

站点以 Cloudflare Workers 静态资源形式发布，使用 **`cf` CLI**（本项目有 `cloudflare.config.ts`，**不要**引入 Wrangler 配置文件）。

```powershell
cf --version                # 本机已验证：1.0.0-rc.2
cf auth whoami              # 本机已验证：boiltask@qq.com（OAuth，token 有效）
```

### 正确流程：部署预构建目录（assets-only）

本仓库**没有** `package.json`、没有构建链，属于纯静态资源项目。因此必须走 `cf deploy <目录>` 的 assets-only 路径：

```powershell
# 1) 只把要对外发布的文件放进 dist/（dist/ 已在 .gitignore 中）
New-Item -ItemType Directory -Force dist | Out-Null
Copy-Item index.html,preview.png,mobile-preview.png,fullscreen-preview.png dist -Force

# 2) 先干跑，确认它识别为 assets-only 且文件数正确
cf deploy dist --dry-run

# 3) 正式部署
cf deploy dist --message "简述本次变更"
```

> ⚠️ **不要直接跑 `cf build` 或裸 `cf deploy`**。已实测：在没有构建链时它会报
> `No Cloudflare dev server is installed`，要求安装 `@cloudflare/vite-plugin` / `wrangler` 等。
> 本项目的正确做法就是上面三步，不要为此引入 node 依赖。

**发布范围**：只发布运行所需资产（`index.html` + 三张预览图）。`AGENTS.md`、`cloudflare.config.ts`、`.git/` 一律不对外暴露；`previews` 会被 README 引用，所以要一起发。

- Worker 名：`pixel-tps-demo`
- 生产域名：`tps.page.boiltask.com`
- 备用域名：`https://pixel-tps-demo.boiltask.workers.dev`
- `compatibilityDate`：`2026-11-17`，`workersDev`: `true`
- 自动化/CI 场景：用 `CLOUDFLARE_API_TOKEN` 代替交互式登录。

### 部署后必须校验

`cf deploy` 返回成功不等于线上已更新，按下面两步确认：

1. 看输出里的 `Current Version ID` 已变化，且 `Uploaded ... assets` 中包含 `index.html`。
2. 拉取线上 `index.html` 与本地做 SHA256 比对，必须完全一致：

```powershell
curl.exe -sS -m 60 --compressed -o "$env:TEMP\live.html" https://pixel-tps-demo.boiltask.workers.dev/
(Get-FileHash "$env:TEMP\live.html").Hash -eq (Get-FileHash index.html).Hash   # 必须为 True
```

> **已知环境问题**：部分网络到 `tps.page.boiltask.com`（104.21.x / 172.67.x）吞吐极低甚至超时，
> 但这是本机出口链路问题，不是部署失败。校验优先用 `workers.dev` 域名；自定义域名失败时
> 换网络或用浏览器人工确认，不要误判为发布异常、更不要反复重发版本。


---

## 10. 提交与版本规范

- 采用 **Conventional Commits**：`feat:` / `fix:` / `perf:` / `refactor:` / `docs:` / `chore:` / `style:`。
- subject 用一句话说明"改了什么"，≤ 72 字符；body 用中文说明**为什么改**与**验证方式**（可列清单）。
- 一次提交只做一件事；迁移、重排、格式化不要混在功能改动里。
- 分支：`main` 保持可玩可部署；较大改动走特性分支再合并。
- 提交前确保 `git status` 干净且 `.cloudflare/` 未被暂存。

```powershell
git checkout -b feat/xxx
# ... 改动与验证 ...
git add -A
git commit   # 或 git commit -F 消息文件（含中文时更稳）
git push -u origin feat/xxx
```

---

## 11. 已知限制（原型边界）

这些是**已知且被接受**的取舍，修它们属于新需求，不属于 bug：

- 角色与敌人是 billboard 精灵，遮挡按深度排序近似，近距离会有穿插。
- 掩体绕行是方向采样，不是导航网格，复杂地形会卡住；**敌人不会攀爬也不会跳**，玩家上高台后它们只在地面绕。
- 攀爬是固定时长的一次性翻越动画：没有悬挂、横向挪移、蹬墙跳或墙跑；空中只有一段跳。
- 只有玩家有垂直移动；敌人与弹丸的落点仍按地面高度近似（弹丸会被台面阻挡，但不会停在台面上滚动）。
- 触屏准星只有轻微辅助吸附，不是自动瞄准。
- 旧版 iOS Safari 的指针捕获/手势可能冲突，需要真机复测。
- 没有蹲伏、武器切换、存档、联网、性能分级。
- 音频必须由一次用户手势启动（浏览器策略）。

---

## 12. 待办 Backlog

按优先级排列，动手前先在 issue 或提交里说明取舍：

1. **性能分级**：低端机降低粒子/后处理密度（目标：稳定 60fps）。
2. **敌人寻路**：方向采样升级为轻量流场或路点图，并让敌人能上/绕开货台（当前它们只会在地面打转）。
3. **武器多样性**：第二把武器 + 切换键位表（需同步 README）。
4. **真机回归**：iOS Safari / Android Chrome 触屏手势实测记录。
5. **CI 冒烟**：用 headless 浏览器 + `window.SUNFALL` 跑状态机断言。
6. **预览图自动化**：改动后一键重截三张预览图，避免文档与实现漂移。
7. **LICENSE**：明确开源协议（当前未声明）。

---

## 13. Agent 禁区

- ❌ 不要在旧目录 `pixel-tps-demo` 里改任何东西。
- ❌ 不要提交 `.cloudflare/`、`node_modules/`、编辑器与系统临时文件。
- ❌ 不要为"顺手优化"而全局格式化、拆分单文件或引入依赖。
- ❌ 不要在没有本地双端验证的情况下宣称改动完成。
- ❌ 不要修改 `.gitignore` / `.gitattributes` 去绕过上述规则；确需调整先说明理由。
- ❌ 不要用 `git push --force` 覆盖 `main` 历史。
