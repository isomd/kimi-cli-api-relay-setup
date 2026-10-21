# Kimi CLI 怎么接入中转站？config.toml、自定义模型与密钥配置

## 仓库附件

- [config.example.toml](config.example.toml)
- [封面提示词](cover-prompt.md)

模板只含占位符或环境变量引用，请按下文合并到自己的配置，保留现有设置。资料核对日期为 2026-10-21；本仓库没有声称已经完成付费线路实测。

---

搜 Kimi CLI 中转配置时，经常会遇到 `~/.kimi/config.toml` 和 `openai_legacy`。这些写法出现在旧版资料里，不能直接当成当前 Kimi Code CLI 的配置。

本文由 Orbis 项目方整理，按当前 Kimi Code CLI 官方文档，使用 `~/.kimi-code/config.toml` 和 `openai` Provider 接入 [Orbis](https://orbiorbi.com)。如果你安装的仍是旧版，先核对版本和它对应的文档，不要同时维护两套不确定是否生效的文件。

## 先确认运行的是哪一个客户端

运行 `kimi --version` 记录版本。当前产品与安装入口见 [Kimi Code CLI 官方文档](https://moonshotai.github.io/kimi-code/en/)。

当前默认配置文件是：

| 系统 | 路径 |
| --- | --- |
| Windows | `%USERPROFILE%\.kimi-code\config.toml` |
| macOS / Linux | `~/.kimi-code/config.toml` |

设置了 `KIMI_CODE_HOME` 时，文件位于对应目录的 `config.toml`。迁移旧配置前先备份，不能只因为搜索到一篇旧教程就改另一个目录。

## 从 Orbis 选择兼容线路

到 [模型广场](https://orbiorbi.com/pricing) 选择支持 OpenAI Chat Completions 的文本模型线路，为它创建 Key。本文使用调用 ID `deepseek-v4.1-flash`，地址为 `https://api.orbiorbi.com/v1`。

核对时该模型的公开线路提供 `openai` 协议。实际使用时，再从所选线路的快速接入页确认地址、模型及限制。

Kimi 是客户端名称，配置里的 `model` 才决定请求使用什么模型；使用 Kimi CLI 不要求把中转站模型 ID 改成 `kimi-for-coding`。

## 添加 Provider 和本地模型别名

在用户级 `config.toml` 中合并下面的内容。`default_model` 放在第一个表头之前；同名表已存在时修改原表，不要重复添加。

```toml
default_model = "orbis-deepseek"

[providers.orbis]
type = "openai"
base_url = "https://api.orbiorbi.com/v1"
api_key_env = "ORBIS_API_KEY"

[models.orbis-deepseek]
provider = "orbis"
model = "deepseek-v4.1-flash"
max_context_size = 128000
```

这里有三个名字：`orbis` 是 Provider 名，`orbis-deepseek` 是本地模型别名，`deepseek-v4.1-flash` 是发给 API 的模型 ID。`default_model` 指向本地别名，不是直接写服务端模型名。

`max_context_size` 是客户端声明，示例用 128000 作为初始值，不是这条线路已经实测支持 128K 的结论。使用前核对线路上限，并按实际限制调整；网页只标模型理论窗口时，不能据此保证线路可接收同样长度。

## 环境变量必须被配置引用

Windows PowerShell：

```powershell
$env:ORBIS_API_KEY = "替换为这条线路的完整 API Key"
kimi
```

macOS / Linux：

```bash
export ORBIS_API_KEY="替换为这条线路的完整 API Key"
kimi
```

当前官方文档支持用 `api_key_env` 显式引用 Shell 变量。仅执行 `export KIMI_API_KEY=...`，但 Provider 没有引用它，并不等于完成凭据配置。

`api_key` 和 `api_key_env` 是互斥方式，选一个。本文使用后者，配置文件中不保存真实 Key。不要把 `${ORBIS_API_KEY}` 写成普通 TOML 字符串，再假定客户端会自动展开。

如果不想手改文件，也可以先查看当前版本的 `/provider` 管理入口；不过从内置厂商目录导入模型，并不自动等同于配置了 Orbis 地址。

## 先读项目，再做一处修改

在演示项目目录启动 `kimi`，确认当前选中 `orbis-deepseek`。发送短问答后，核对 Orbis 调用日志，再给它一个文件任务：

> 阅读这个项目的 README 和源代码目录，列出三个适合新维护者先看的文件，并说明各自的作用。引用实际路径，不要根据常见项目结构猜文件名。先不要改动文件。

人工打开它提到的文件，检查说明是否成立。随后可以让它把这份导读写到一个新文档，先预览内容，再允许保存。

这一过程分别检查模型响应、文件读取和写入。能成功回答短问题，不能代替后两项验证。

## 这些错误不要靠换模型名猜

| 现象 | 优先检查 |
| --- | --- |
| 改完文件仍是旧配置 | 客户端版本、`KIMI_CODE_HOME`、实际读取目录 |
| 缺少 Key | `api_key_env` 对应变量是否已设置，是否误配了互斥字段 |
| 默认模型不存在 | `default_model` 是否对应 `[models.…]` 的别名 |
| 404 | Provider 类型与接口地址是否匹配 |
| 多轮工具调用失败 | 流式返回、工具调用、推理字段与线路兼容性 |

当前官方类型中，Chat Completions 对应 `openai`，Responses 对应 `openai_responses`。本文只配置前者。需要换线路时，也要准备对应 Key；Orbis 不会仅按新模型 ID 自动换线路。

## 参考资料

- [Kimi Code CLI 官方配置文件](https://moonshotai.github.io/kimi-code/en/configuration/config-files)
- [Kimi Code CLI 官方 Provider 说明](https://moonshotai.github.io/kimi-code/en/configuration/providers)
- [Orbis 快速开始](https://orbiorbi.com/docs/quickstart/)

资料核对日期：2026-10-21。本文按当前文档整理，没有进行本轮付费调用；旧版 Kimi CLI 请使用对应版本的配置规则。
