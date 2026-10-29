# agent-purge

[![npm](https://img.shields.io/npm/v/agent-purge)](https://www.npmjs.com/package/agent-purge)
[![stars](https://img.shields.io/github/stars/ZiChuanLan/agent-purge)](https://github.com/ZiChuanLan/agent-purge/stargazers)
[![license](https://img.shields.io/npm/l/agent-purge)](LICENSE)

一份系统提示词，喂给多个 coding agent。支持替换（replace）、只发人设（strict）、追加（append）、清空（off）四种模式，可热切换，全程可逆。

```
agent-purge web          # 本地控制台 http://127.0.0.1:7788
agent-purge codex        # 用当前 persona 启动 codex
agent-purge pi --ask "…" # 一次性提问
```

## 它做什么

每家 coding agent 读系统提示词的位置都不一样：有的认启动参数，有的认配置文件，有的只能靠插件改运行时对象。agent-purge 把这些入口统一成三件事：

| 通道 | 机制 | 生效范围 |
|---|---|---|
| `install` | 写入该 harness 的托管配置块（Markdown / hooks / 插件） | 常驻，别人启动也生效 |
| `shim` | 往 PATH 前置同名小壳，转发给 `agent-purge <h>` | 裸敲命令也接管 |
| `slash` | 会话内 `/purge on\|off\|status` | 热切换，不动磁盘 |

三层都写的是同一份 `personas/<id>.md` 的正文。`--mode off` 或 Web 总开关关掉时，全部撤回到原样——不留残渣。

## 安装

```bash
npm i -g agent-purge     # 或 npx agent-purge web
```

要求 Node ≥ 20，零依赖。

**用 magpie 网关？** 插件不在 npm 包里，在仓库的 `magpie-plugin/` 子目录——不要指向仓库根（根 `package.json` 是 CLI，只有 `bin`，magpie 会拒绝）：

```bash
magpie plugin add /path/to/agent-purge/magpie-plugin
```

首次使用先跑一次 `agent-purge web`（打开 http://127.0.0.1:7788 的控制台，先看/选一个人设；不想开界面就用 `agent-purge ls`），它会生成 `~/.agent-purge/personas/`；再跑 `node magpie-plugin/gen-personas.mjs` 生成 Persona 页的人设清单。

详见下文「magpie 网关插件（可选）」。

**数据独立存放**，升级不会被覆盖：默认 `~/.agent-purge/`（Windows `C:\Users\<你>\.agent-purge\`），里面是 `state.json` 和 `personas/`。想换位置就设环境变量：

```bash
export AGENT_PURGE_HOME=/path/to/your/data     # Windows: setx AGENT_PURGE_HOME D:\ap-data
```

旧版本（0.1.x）把数据放在包目录里，首次运行会自动搬到新家，原文件不动。

## 人设

`personas/<id>.md` = frontmatter + 正文。正文就是会被写进系统提示词的那一段。

frontmatter 用 `description` 写「说明」（Web 列表和 `agent-purge ls` 显示的就是它）；旧的 `desc` 仍然认，不会报错。可选 `order`（排序）、`harness` / `model`（匹配用）。Web 里只改名字和说明时会保留 frontmatter 的其它键，不整段重写。

```bash
agent-purge ls                 # 列出全部 persona，标出当前匹配的
agent-purge use <id>           # 切换当前 persona
agent-purge pi --persona <id>  # 只这一轮用别的
```

首次运行会在 `personas/` 下生成 `neutral.md` 起步模板。

## 支持情况

每个数字都是实测值：本地假 provider 抓真实请求体，不是估算。

| harness | 机制 | 实测版本 | 默认 system | 注入后 |
|---|---|---|---|---|
| `codex` | 启动参数 `-c model_instructions_file=` | 0.163.0 | 21,026 | 24 |
| `claude` | `--system-prompt-file` + 二进制补丁清残留身份句 | 2.1.304 | 6,379 | 156 |
| `grok` | `--system-prompt-override` | 1.0.62 | 5,146 | 21 |
| `omp` | `--system-prompt` | 17.4.2 | 34,511 | 10,106 |
| `hermes` | 临时 HOME 里替换 `SOUL.md` | 0.21.5 | 8,139 | 7,554 |
| `zcode` | UserPromptSubmit hook（user 角色）+ `customSystemPrompt` | 0.16.9 | 9,359 | 9,359 |
| `opencode` | cwd 下 `.opencode/plugin/*.ts` 原地改 `output.system` | 1.19.3 | 19,332 | 112 |
| `opencode2` | `plugins/agent-purge-v2.ts` 的 `ctx.session.hook("context")` | 0.0.0-beta-17823 | 8,279 | 3,189 |
| `dsh` | cordis 插件（三个 profile 各一份） | 0.2.1-rc.2 | 5,713 | 68 |
| `pi` | 原生扩展，会话内 `/purge` | — | — | — |
| `antigravity` | 等长替换二进制里的身份句 | 1.23.x | ≈20,498 | — |
| `codebuddy` | `--system-prompt-file` | 2.171.0 | — | — |
| `qoder` / `workbuddy` | 不支持（提示词在服务端） | — | — | — |

`opencode` 和 `opencode2` 是两个不同的 npm 包（`opencode-ai` 1.x 与 `@opencode/cli` 2.x），插件接口完全不同，各自独立支持。

`agent-purge ls` 和 Web 控制台显示的是**运行期真值**：每个 harness 有没有装、版本、注入层（SYS / USER）、三个通道各自可用性。`matrix.json` 里是静态能力表，两者冲突时以运行期为准。

闭源桌面端（提示词在服务端或二进制里）只做原样启动，不做任何注入——`agent-purge <h>` 会明确打印原因，不会假装生效。

### 常驻安装（install）

```bash
agent-purge install <h>      # 写进该 harness 自己的托管位置，别人启动也生效
agent-purge uninstall <h>    # 摘掉，按字节还原
```

dsh 是唯一有多个形态的：desktop / web / headless 共用一套 cordis 插件，装在各自 profile 的 `node_modules/agent-purge/`，并在 `cordis.patch.yml` 里 `- insert` 挂上，不改 `app.asar`。装完重启一次，之后热生效；三个形态还能在 Web 面板分别勾选。

## magpie 网关插件（可选）

[magpie](https://github.com/yetone/magpie) 把各家 agent 的请求收进本地网关。挂上这个插件后，**凡是经过网关的请求**（Claude Code、Codex、OpenCode、pi…）的 system 段都按 agent-purge 的当前人设改写——不用逐个 harness 装，换人设也不用重启 magpie。

插件在仓库里，不进 npm 包（`files` 白名单），所以按路径装：

```bash
git clone https://github.com/ZiChuanLan/agent-purge
magpie plugin add /path/to/agent-purge/magpie-plugin     # 按路径装
magpie plugin options magpie-plugin '{"mode":"append"}'  # 插件名 = 目录名
magpie plugin list                                      # 应看到 async gateway hook + Persona 页
```

插件读的还是 `~/.agent-purge/`（同样认 `AGENT_PURGE_HOME`）：`agent-purge use <id>` 之后下一个请求就换人设。magpie 里也会多出 **Persona** 页，点一下就把人设和模式写进插件的 options。

四种模式和 CLI 一致，差别在网关手里只有文本、没有各家启动参数：

| 模式 | 网关插件做什么 |
|---|---|
| `off` | 不动请求 |
| `strict` | system 整段换成人设（工具说明也一起没了，harness 可能变笨） |
| `append` | 人设挂在最前，宿主 system 原样 |
| `replace` | 人设在前，再删掉宿主 system 里报名字的那句（近似；要干净用 `strict`） |

options 三键：`mode`、`persona`（人设 id）、`agents`（只对这些 magpie agent 生效，如 `["claude","codex"]`）。

**边界**：改的是请求体里的文本，harness 在客户端就拼死的东西拦不住；`replace` 只删「一行里同时有自称和点名模型/厂商」的行，比 CLI 那套 per-harness 处理粗。插件页是沙箱（没有 fetch、读不到磁盘），人设清单由 `gen-personas.mjs` 按你本机的人设生成到本地（不进仓库），**装完先跑一次**，之后增删人设再跑：`node magpie-plugin/gen-personas.mjs`。前提是已经运行过 `agent-purge`（如 `agent-purge web` 或 `agent-purge ls`），`~/.agent-purge/personas/` 里有 persona；目录不存在时脚本会直接提示。自测不依赖 magpie：`npm test`（钩子 26 项断言：四种协议 × 四种模式；页面逻辑 18 项：搜索过滤、options 合并、失败回滚）。

## 目录

```
bin/         CLI 入口
lib/         状态、persona、注入、通道、检测
backends/    每个 harness 的启动参数与身份句处理
web/         控制台（server.mjs + ui.html）
bench/       注入成功率实测与裁判脚本（开发用，不参与运行）
magpie-plugin/  magpie 网关插件（可选，不进 npm 包）
matrix.json  能力矩阵
```

## 边界

- 只改提示词相关的位置，不碰二进制、不破解、不绕过任何授权。
- 所有改动可逆：`agent-purge uninstall <h>` / `disarm` / `--mode off`。
- 不做文件夹级人设隔离：技术上做得到，但会打破「一份人设、所有入口、一起切换」，卸载也不再是字节还原。
- 状态只存本机 `state.json`，不上传任何数据。

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=ZiChuanLan/agent-purge&type=Date)](https://star-history.com/#ZiChuanLan/agent-purge&Date)

## License

MIT

---

感谢 [LINUX DO](https://linux.do) 社区
