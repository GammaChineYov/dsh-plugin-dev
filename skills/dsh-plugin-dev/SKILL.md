---
name: dsh-plugin-dev
description: >
  用户要开发、维护、发布 DSH 插件时使用：「开发个插件」「写个 dsh 插件」「改一下
  那个插件」「插件更新/修复」「把插件发布到 GitHub」。覆盖本机编译专用仓库 +
  隔离测试环境（docker 容器）+ 主环境装包 + 人工核验的完整流水线（实验插件一律
  不直装生产 dsh）、溢出/性能/交互闭环/视觉核验测试清单、插件包三要素与
  client bundle 契约、开发踩坑清单、官方发布规范。
  只找/装现成插件（「有没有插件能……」「装个 XX」）转 find-plugins skill；
  写/改 Cordis 组合（agent 预设、cordis.yml）另加载 editing-cordis-compositions。
---

# dsh 插件开发 · 维护 · 发布

本 skill 是插件开发的**规范流程**可执行版，核心思想：**实验验证与正式环境彻底隔离**。
**职责分离约定**：本 skill 只承载"做什么、怎么判断"（流程、判定、契约、清单）；
所有**机器特定事实**（编译仓库路径、隔离测试服务器地址/SSH 密钥/隧道命令、每只
插件的仓库与安装状态、历史踩坑复盘）由使用方存放在自己的记忆/档案区（如
`$DSH_HOME/memory/`），本 skill 只保留指针——事实变更只改档案，skill 永不漂移，
同一份 skill 可在多台机器复用。

## 0. 动手前必读（指针，按需读）

- **自研插件档案**：编译仓库路径、每只插件的仓库/安装状态/更新流程、通用安装约定、踩坑复盘
- **隔离测试环境档案**：测试服务器地址、SSH 密钥、隧道命令、docker 测试版、主环境装包步骤
- **DSH 机制速查**：slot/事件契约、client bundle 契约、启动方式
- 动态插件开发：加载 `cordis-plugin-development`；组合编写：加载 `editing-cordis-compositions`

## 1. 铁律（为什么有这条流程）

**实验插件一律不得直接装进生产 dsh 环境。** 曾因插件开发阻塞正式开发一整天。
所有插件代码只在**本机编译专用仓库**编译，测试验证只在**隔离测试环境**
（docker 容器 → 主环境装包 → 人工核验）做，用户确认测试通过后**才**装回生产/本机测试环境。

动手前先确认两件事，缺一问用户：
1. 目标任务是**开发新插件 / 维护现有插件 / 发布插件**（决定走哪几节）；
2. 若维护，先读档案里该插件的段落，别凭记忆动手。

## 2. 完整流水线（开发、维护、发布都走这条）

1. **本机编译**：代码在本机编译专用仓库编译通过（`node --check` / `pnpm build` / `tsc -b`，
   按仓库形态）。**不**在生产 checkout 编译实验插件，**不**装进生产 profile。
2. **部署隔离测试环境**：编译产物部署到隔离测试服务器（路径按档案）。
3. **docker 容器独立快照测试**：在服务器上用 docker 容器（隔离，不污染主环境）做快照测试，
   逐项过第 4 节测试清单。
4. **主环境装包**：容器测试通过后装进服务器**主环境 dsh**（保持与生产插件环境一致），
   确认：**与其它插件无冲突、渲染没问题**。
5. **桥接人工核验**：经 SSH 隧道桥接，让**用户在本机 GUI 最终人工核验**。
6. **装回生产**：**用户确认测试通过后**才装到生产/本机测试环境（正式包）。
7. **登记档案**：同一会话内主动更新自研插件档案（新增/更新该插件段），不等用户提醒。

每步完成点：编译零报错 → 服务器文件就位 → 测试项全过 → 主环境加载正常 → 用户人工核验通过
→ 生产安装成功 → 档案已更新。任何一步不过，停下修、重新走，**不跳过、不把问题带进下一步**。

## 3. 开发规范

### 3.1 形态选择

| 形态 | 用途 | 说明 |
|------|------|------|
| 动态 Cordis 插件（`cordis_define`/`cordis_run`） | 会话内快速迭代原型 | 进程内存 + 会话级，宿主重启即消失；开发调试可用，**不是发布形态**。先加载 `cordis-plugin-development` |
| 独立 bundle 包（仓库） | 发布/长期安装 | `lib/index.js`（host）+ `lib/client.js`（bundle）+ `cordis.patch.yml` + `package.json` 三要素 |
| 纯 skill（非运行时插件） | 给 Agent 的指令集 | 仓库 `skills/<name>/` 目录复制到 `$DSH_HOME/skills/<name>/`，目录 watcher 立即生效无需重启 |

### 3.2 独立包结构三要素

1. `package.json`：`"dsh": {"bundle": {"patch": "./cordis.patch.yml"}, "client": {"inject": [...], "platform": "web"}}` + `exports["./client"]`；
2. `cordis.patch.yml`：`- insert: [{id, name}]`（bundle patch 自动发现，通常无需手动合入 profile patch）；
3. `lib/`：host 端 ESM `lib/index.js` + client bundle `lib/client.js`。

### 3.3 client bundle 契约（关键）

- 格式：`window.__ModuleLoader__.load({ id, factory })`，**id 必须带 @scope**（如 `@dsh-external/xxx`），
  factory 返回 `{ apply(ctx) {} }`（挂载逻辑放 apply）；返回空对象报 invalid plugin。
- `React = require('react')` 走 shell 模块表（external，勿内嵌 React——双 React 会致 slot 渲染空白）。
- **样式一律用 React 行内 style**（`style={{...}}`，fallback 抄 `--dsw-*` 主题变量默认值）。
  **严禁全局 `<style>` 注入**——实测会跨插件污染其它插件渲染。行内样式物理隔离，不可能影响他人。
- 注册 slot 需先查 Slots 树拿精确契约；keyed 槽按 priority 升序、最小者赢——要覆盖官方渲染器
  用 `priority: -1`，但要意识到这会**完全接管**（渲染返回 null 不会回退官方，需要官方子视图时
  声明 `children` 并 `renderSlot` 转发）。
- 多客户端插件并存的「样式/渲染」问题：用 profile 配置 disabled + 刷新页面快速二分
  （无需重启 web）；刷新第一时间正常 + 过一会才坏 = 运行时副作用，优先怀疑轮询/全局监听/样式注入。

### 3.4 编码与文件约束

- 改 profile `package.json` 必须**无 BOM UTF-8**（带 BOM 会让 harness `JSON.parse` 炸
  `Unexpected token ''`；用 `[System.IO.File]::WriteAllText` + `UTF8Encoding($false)`，
  勿用会写 BOM 的 `Set-Content -Encoding UTF8`）。
- npm 包可能自带 BOM 的 package.json——用 pnpm patch 固化。
- **所有描述性内容用中文**（package.json description、README、commit message、PR/issue 文案），
  代码注释与标识符不限。

### 3.5 开发高频踩坑（动手前过一遍；历史复盘细节在档案）

- **Cordis 插件**：访问 ctx 服务必须先写进 `inject` 数组（`cannot get property without inject`
  是运行时才炸）；可选服务用 `ctx.get('name')` 软取 + undefined 守卫。
- **手写 bundle 循环**：`for(;;)`/`while` 必须检查数组越界（曾因此死循环致打开任何会话 CPU 100% 卡死）。
- **React Hooks**：条件 return 之后不得调用 hooks（hooks 数量不一致 #310）。
- **React state 弹层不渲染**（slots 组件里谜团未解）：绕过方案 = 命令式 DOM
  （createElement + appendChild 到 body），徽标本体仍 React。
- **外点关闭监听**：忽略菜单内部的 mousedown（`e.target.closest(...)`），否则点菜单项时
  先卸载、click 永不触发。
- **数据卫生**：不序列化 live block/服务引用；只读叶子字段构造小对象；事件风暴要分帧
  （条数 + 单帧字节双上限）、丢弃中间态。
- **host↔client 通信**：独立包用 webServer HTTP 路由 + fetch（`harness.handle`/`host.call`
  是动态插件专用）。权限守卫：回环 127.0.0.1/localhost 放行，非回环 403。
- **联动第三方插件**：走其公开 service（`ctx.get` 软取 + `typeof fn === 'function'` 守卫），
  缺失时优雅降级；不依赖内部 store/DOM。
- **性能直觉**：大对象缓存必须设上限；会话历史全量常驻内存 = 每会话 +300~600MB，多会话必 OOM。

## 4. 测试清单（隔离环境 docker 容器内逐项过）

> 容器内完成 4 类测试；全部通过才进主环境装包。用浏览器工具经 SSH 隧道操作 GUI 实测
> （navigate/click/type/snapshot/screenshot）。

**溢出测试（内存）**
1. 多会话加载后内存异常增量（逐个点开多个未加载会话，观察 host 内存；参照官方基线，
   插件不应随会话数线性暴涨）；
2. 刷新页面内存溢出（刷新后进程内存是否回落到合理值）；
3. 刷新页面后首次点击页面内存溢出（刷新后第一个交互是否触发异常分配）。

**性能测试（卡顿）**
1. 发指令消息卡顿；
2. 初次点击打开长会话卡顿（大会话/长历史）；
3. 初次点击打开短会话卡顿；
4. 刷新页面卡顿。

**交互闭环测试**：点击/展开/收起/输入/切换/关闭全链路闭环，无死交互、无悬空状态。

**视觉模型核验**：视觉模型渲染核验（界面渲染是否正确、有无错位/缺失/污染）。

排查手段（验证过有效）：
- 多会话压力测试是硬指标（单会话测不出累积泄漏）；
- 二分禁用插件（web profile bundles 逐个禁用）定位冲突/性能元凶；
- host 以 `--inspect=9229` 启动后 CDP `Runtime.evaluate` 读 `process.memoryUsage()`；
  堆快照用 CDP `HeapProfiler.takeHeapSnapshot` + 流式解析（>500MB 不能整块 toString）。

## 5. 维护：插件出问题怎么办

**一律按流水线处理，不直接在生产环境改/试：**
隔离环境 docker 容器测试 → 主环境装包测试 → 桥接测试 → 用户确认 → 再装回正式包。

排查方法（验证过）：
- 先二分"官方纯配置 vs 加插件"定位是不是插件；
- 再对比插件 git 版本缩小范围；**注意插件是否真正生效**（priority shadow 关系会干扰对比——
  `priority:-1` 才真正替换官方渲染器，否则插件代码不执行、测不出真实行为）；
- 性能问题优先怀疑：无边界循环、无限增长 Map、全量历史缓存、全局监听/轮询、样式注入。

## 6. 发布

- **官方发布规范**：DSH checkout 内 `docs/user/develop/basic/publish.md`。
- **GitHub 发布三要素**：LICENSE(MIT) + README（中文）+ package.json（三要素齐）。
  `lib/` 构建产物直接提交 → git 安装无需 prepare/allowBuilds；`dsh plugin --profile web add
  git+https://github.com/<owner>/<repo>.git` 可装。发布前 `node --check lib/*.js` + 冒烟测试。
- **官方不收外部 PR**（CONTRIBUTING.md 原文）→ 改动官方 checkout 的发布落点 = 自己账号下
  fork 分支（`git push --no-verify fork <branch>`，pre-push 钩子 corepack 门禁会拦）；想给
  官方看功能 → 发 Discussion（Ideas 分类，先扫重复，中文人类语气，别用 AI 模板腔）。
- 发布到 npm 注意：`dsh.profile.bundles` 与 `exports["./client"]` 必须就位；描述用中文。

## 7. 参考档案（按需读，勿复制事实进本 skill）

- 自研插件档案（编译仓库、各插件状态、通用安装约定、踩坑复盘）
- 隔离测试环境档案（服务器地址、密钥、隧道、docker）
- DSH 机制速查（slot/事件契约、启动方式）
- 官方文档（DSH checkout 内）：`docs/architecture.md`、`docs/user/develop/basic/publish.md`
- 动态插件开发：加载 `cordis-plugin-development` skill；组合编写：加载 `editing-cordis-compositions` skill
