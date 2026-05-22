下面是一版可以直接放进博客的草稿。

## 为 CLI 设计 AI Skills：按领域组织，而不是按命令堆叠

当我们希望一个 CLI 程序不仅服务人类用户，也能被 AI Agent 稳定调用时，单纯提供 `--help` 和命令文档是不够的。AI 需要的不只是“有哪些命令”，而是“在什么场景下该用哪些命令、执行前要注意什么、遇到错误该如何判断”。

这就是 CLI Skills 的价值。

以飞书 CLI 为例，它把 skills 作为面向 AI Agent 的分发资产，用户安装 CLI 后，还会通过类似下面的命令安装 skills：

```bash
npx skills add larksuite/cli -y -g
```

飞书的 skills 不是简单把每个子命令拆成一个文档，而是按能力域组织，比如 `lark-calendar`、`lark-doc`、`lark-base`、`lark-im`，同时提供 `lark-shared` 作为共享基础规则。它的模板中也明确要求领域 skill 在开始前读取 shared skill，例如 `../lark-shared/SKILL.md`，用于承载认证、权限、安全等通用规则。参考：[larksuite/cli skill-template](https://github.com/larksuite/cli/tree/main/skill-template)。

## 核心原则：一个领域一个 Skill

我认为 CLI skills 的顶层组织，应该优先按“领域”划分，而不是按“用例”或“单个命令”划分。

比如认证相关能力，不应该拆成：

```text
login-skill
logout-skill
switch-skill
list-auth-skill
```

更好的方式是：

```text
crater-cli-auth/
  SKILL.md
  references/
    crater-cli-auth-login.md
    crater-cli-auth-identities.md
    crater-cli-auth-remove.md
```

`crater-cli-auth` 负责整个认证域，包括登录、重新登录、查看身份、切换身份、删除凭据、登出，以及 401/403/token/Keyring 等认证问题的判断。

这样做的好处是，AI 在路由用户意图时不需要在一堆非常相近的小 skill 中选择。用户说“我好像没登录”“帮我切换账号”“token 失效了”，这些都自然落入 auth 域。

## Shared Skill：承载所有通用约束

除了领域 skill，还需要一个共享 skill。比如：

```text
crater-cli-shared/
  SKILL.md
  references/
    crater-cli-global-flags.md
```

它不对应某个业务功能，而是告诉 AI 如何安全调用整个 CLI：

- 如何使用 `--json`
- `--no-interactive` 的含义
- 什么时候查看 `--help`
- 错误从 stdout 还是 stderr 读取
- 不要让用户在聊天里发送密码、token、cookie
- 执行会修改状态的命令前必须确认用户意图
- 不要静默添加 `--yes`

这类信息如果散落在每个领域 skill 里，会重复、难维护，也容易不一致。更好的方式是让所有领域 skill 在开头明确依赖 shared skill：

```md
**CRITICAL — 开始前 MUST 先读取 `crater-cli-shared`（可能路径：[`../crater-cli-shared/SKILL.md`](../crater-cli-shared/SKILL.md)）。**
```

这里同时提供 skill name 和可能路径，是为了兼容不同 AI 工具。有些工具能解析相对路径，有些工具只能根据名称理解要找的 skill。

## Reference：按场景拆，而不是按命令机械拆

领域 skill 的 `SKILL.md` 应该保持短小，负责意图识别、核心模型和安全边界。更细的内容放到 `references/`。

但 reference 也不应该机械地一个命令一个文件。更好的拆法是按场景或操作簇拆分。

比如 auth 域可以拆成：

```text
references/
  crater-cli-auth-login.md        # 登录、重新登录、401/token 失效
  crater-cli-auth-identities.md   # 查看身份、筛选身份、切换 active_context
  crater-cli-auth-remove.md       # logout、rm、两者区别和删除安全
```

为什么不是 `login.md`、`ls.md`、`switch.md`、`logout.md`、`rm.md`？

因为 AI 处理的是用户意图，而不是命令索引。`ls` 和 `switch` 经常一起出现：切换身份前通常要先查看有哪些身份。`logout` 和 `rm` 也需要放在一起解释，因为用户经常混淆“退出当前账号”和“删除某个保存凭据”。

reference 的拆分标准应该是：AI 在完成一个用户任务时，是否通常需要同时理解这些命令。如果需要，就应该放在同一个 reference 里。

## Skill 不是命令手册

一个常见误区是把 skill 写成完整命令手册，把所有选项、所有字段、所有边界都塞进去。

这不一定有用。

AI 真正需要的是：

- 什么时候用这个能力
- 推荐先用哪个命令
- 哪些命令有副作用
- 哪些信息不能让用户发到聊天里
- 出错时如何判断
- 常见场景的范例命令
- 精确语法不确定时如何查看 `--help`

所以 skill 应该更像“Agent 操作指南”，而不是“人类 API Reference”。

例如：

```md
用户需要查看当前身份时，先运行：

crater auth ls --json

如果 active_context 为空，说明当前没有激活身份。
如果 auth_infos 中没有目标身份，需要先登录，而不是切换。
```

这比单纯罗列 `crater auth ls` 的所有参数更有价值。

## 命名也很重要

如果一个项目既有面向用户的 CLI skills，也有面向开发者的工程 skills，命名上应该区分清楚。

例如：

```text
crater-cli-auth
crater-cli-shared
crater-cli-config
```

这些表示“帮助用户的 AI Agent 调用 Crater CLI”。

而开发相关的 skill 可以叫：

```text
crater-devel-cli-command
crater-devel-snapshot-test
crater-devel-release
```

这样 AI 在选择 skill 时，会更容易判断：当前是在“使用 Crater CLI 完成任务”，还是在“开发 Crater 项目本身”。

## 推荐结构

最终，一个比较稳妥的结构是：

```text
cli/skills/
  crater-cli-shared/
    SKILL.md
    references/
      crater-cli-global-flags.md

  crater-cli-auth/
    SKILL.md
    references/
      crater-cli-auth-login.md
      crater-cli-auth-identities.md
      crater-cli-auth-remove.md
```

未来如果 CLI 增加更多领域，可以继续扩展：

```text
crater-cli-config/
crater-cli-project/
crater-cli-job/
crater-cli-image/
crater-cli-workflow-onboarding/
```

其中 `workflow-*` 适合放跨领域任务。比如“从零开始配置并登录 CLI”，可能同时涉及 config、auth、completion，就可以沉淀成 onboarding workflow。

## 总结

CLI skills 的组织方式，应该服务 AI 的意图路由和任务执行，而不是复刻命令树。

我的经验判断是：

- 顶层按领域拆 skill。
- 通用规则放 shared skill。
- 复杂场景放 workflow skill。
- reference 按用户任务场景拆，不机械按子命令拆。
- `SKILL.md` 保持短小，负责路由、模型和安全边界。
- 具体命令范例放 reference。
- 精确参数不确定时，引导 AI 使用 `--help` 检查当前 CLI 版本的真实用法。

一句话：**Skill 不是给人读的命令手册，而是给 AI Agent 用的操作心智模型。**