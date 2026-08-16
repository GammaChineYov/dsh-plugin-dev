# dsh-plugin-dev

<p align="center">
  <strong>简体中文</strong> | <a href="README.en.md">English</a>
</p>

给 DeepSeek Harness（DSH）开发、维护、发布插件的 Agent Skill。对 DSH 说一句
「开发个插件」「改一下那个插件」「把插件发布到 GitHub」，它就会按一套完整的
**开发 → 隔离测试 → 发布**流水线工作。

## 它解决什么问题

DSH 插件跑在你的 DSH 进程里，装一个实验插件等于在你的正式环境里引入风险：
样式污染、内存泄漏、与其它插件冲突，甚至拖垮日常开发。这套 Skill 把
**「实验验证」和「正式环境」彻底隔离**：

1. **本机编译专用仓库**：实验插件只在独立克隆的编译仓库里编译，不装进正式环境；
2. **隔离环境测试**：部署到隔离测试服务器，用 docker 容器做独立快照测试
   （不污染主环境），逐项过四类测试清单；
3. **主环境装包验证**：容器测试通过后再装进服务器主环境 dsh，确认与其它插件
   无冲突、渲染无误；
4. **人工核验后再装回**：经 SSH 隧道桥接做最终人工核验，**用户确认测试通过后**
   才装回本机测试环境。

## 它包含什么

- **完整流水线**：编译 → 部署 → 容器快照测试 → 主环境装包 → 桥接核验 → 装回 → 登记档案；
- **四类测试清单**：
  - **溢出测试（内存）**：多会话加载后内存异常增量 / 刷新页面内存溢出 /
    刷新页面后首次点击页面内存溢出；
  - **性能测试（卡顿）**：发指令消息 / 初次打开长会话 / 初次打开短会话 / 刷新页面；
  - **交互闭环测试**：点击 / 展开 / 收起 / 输入 / 切换全链路闭环；
  - **视觉模型核验**：界面渲染是否正确、有无错位 / 缺失 / 污染；
- **开发规范**：动态 Cordis 插件 vs 独立 bundle 包的形态选择、包结构三要素、
  client bundle 契约（`__ModuleLoader__`、行内样式、slot priority）、编码约束（无 BOM）；
- **高频踩坑清单**：`inject` 声明、循环越界死循环、React Hooks 顺序、外点关闭、
  数据卫生、host↔client 通信、性能直觉；
- **维护流程**：插件出问题一律按「容器隔离测试 → 主环境装包测试 → 桥接测试 →
  再装回正式包」处理，不直接在本机正式环境改；
- **发布规范**：官方 `publish.md`、GitHub 三要素（LICENSE + README + package.json）、
  官方不收外部 PR 时的 fork / Discussion 姿势。

## 安装

把下面这句话发给 DSH：

```text
请从 https://github.com/GammaChineYov/dsh-plugin-dev 安装 dsh-plugin-dev skill
```

手动安装时，把 `skills/dsh-plugin-dev/` 整个目录复制到 `$DSH_HOME/skills/`
（默认是 `~/.dsh/skills/`）；只想给当前项目使用，则复制到 `<项目根>/.dsh/skills/`。
如果还想与其他 Agent 共用，也可以放在 `<项目根>/.agents/skills/`。
目录 watcher 会让它立即生效，无需重启。

## Skill 与记忆的分工

本 Skill 只承载**可执行流程**（做什么、怎么判断），所有**机器特定事实**（编译仓库
路径、测试服务器地址、SSH 密钥、每只插件的仓库与安装状态）由使用方放在自己的
记忆/档案区，Skill 通过指针引用——事实变更只改档案，Skill 永不漂移，也方便
多台机器复用同一份 Skill。

## 依赖

- DeepSeek Harness（DSH），动态插件开发需要 `cordis-plugin-development` skill，
  组合编写需要 `editing-cordis-compositions` skill（随 DSH 自带）；
- 隔离测试需要一个可部署 DSH 的服务器（docker 可选但推荐，用于纯净快照测试）。

License: [MIT](LICENSE)
