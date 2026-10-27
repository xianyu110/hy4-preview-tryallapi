# 混元 Hy4 Preview API 调用教程：用 TryAllAPI 做 OpenAI 兼容对话接入

> **混元 Hy4 Preview**（TryAllAPI 模型 ID：`hy4-preview`）是腾讯混元系列预览模型：总参数约 770B、激活参数约 49B，重点面向 Agent、Coding 与生产力场景。价格接口标签为「对话 / 工具」。本文记录如何通过聚合站 **[TryAllAPI](https://tryallapi.com/)**，从模型广场 / 价格页选型、读基价与分组倍率，到 `POST /v1/chat/completions` 的完整接入流程。价格与分组倍率取数 **2026-10-27（北京）**，**以页面实时为准**；示例仅为请求形态说明，未跑通真实长任务计费。

**Keywords:** 混元 Hy4 · hy4-preview · 腾讯混元 · TryAllAPI · OpenAI 兼容 · Anthropic 兼容 · `/v1/chat/completions`

---

## 目录

- [一、为什么用聚合站调混元](#一为什么用聚合站调混元)
- [二、在价格页 / 模型广场找到 hy4-preview](#二在价格页-模型广场找到-hy4-preview)
- [三、模型怎么选：与相邻档对照](#三模型怎么选与相邻档对照)
- [四、价格怎么读](#四价格怎么读)
- [五、调用流程：Key → Base URL → chat/completions](#五调用流程key-base-url-chatcompletions)
- [六、curl / SDK 最小示例](#六curl-sdk-最小示例)
- [七、常见注意](#七常见注意)
- [FAQ](#faq)
- [八、小结与试用入口](#八小结与试用入口)

---

## 一、为什么用聚合站调混元

只用腾讯云官方通道、能走厂商控制台充值开票时，直连往往最干净；如果项目里还要同时调 GPT、Claude、Gemini、国内模型，为每家维护一套 Key 和重试逻辑也比较散。

聚合站（如 [TryAllAPI](https://tryallapi.com/)）的常见价值：

1. **统一鉴权**：一个 Bearer Key，切换 `model` 字段即可试不同模型；
2. **OpenAI 兼容形态**：Base URL 固定为 `https://tryallapi.com/v1`，Cursor / Dify / 各类 SDK 改地址即可；
3. **模型广场可对照**：标签、分组倍率、端点类型一目了然。

代价是多一跳：延迟与可用性取决于分组上游，建议先用小流量探针。注册试用入口：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 二、在价格页 / 模型广场找到 hy4-preview

打开 [价格页](https://tryallapi.com/pricing) 或模型广场，按供应商 **腾讯混元** / **Hunyuan**、类型 **对话**、关键词 `hy4` / `hy4-preview` 筛选。详情卡截图如下：

![混元 Hy4 Preview 模型广场详情卡：模型 ID、API 与分组价格](docs/model-square.png)

详情卡要点（取自价格接口 2026-10-27 快照）：

| 字段 | 内容 |
| --- | --- |
| **模型 ID** | `hy4-preview` |
| **供应商** | 腾讯混元 |
| **标签** | 对话、工具 |
| **端点类型** | `openai`（`POST /v1/chat/completions`）、`anthropic` |
| **可用分组** | hunyuan-1 |
| **计费** | 按量（token） |

文中 UI 品牌可能显示为模型广场 / 第三方壳，实际对接产品以 [TryAllAPI](https://tryallapi.com/) 控制台与价格页为准。本文以 OpenAI 兼容为主。

---

## 三、模型怎么选：与相邻档对照

只根据价格接口可见信息做「可见对照」，**这是价格接口对照，不是官方对标评测**；不编造未出现的能力差异。基价按价格接口换算（$/1M，未乘分组倍率）：

| 模型 ID（可见） | 基价 输入 / 输出 | 怎么理解 |
| --- | --- | --- |
| **`hy4-preview`** | $0.834 / ≈$2.501 | 本文主线：混元 Hy4 预览档 |
| `glm-5.3` | $1.4 / $4.4 | 同站可见国内对话模型 |
| `qwen3.8-max` | $2 / $6 | 同站可见国内对话模型 |
| `deepseek-v4.1-flash` | $0.3 / $1.2 | 同站可见国内对话模型 |

实务上：经 TryAllAPI 调用统一写 `hy4-preview`；遇限流或分组无货，再在同站可见国内对话 ID 之间切换，而不是改 Base URL。

---

## 四、价格怎么读

对话模型常见公式：**≈ 基价 × 分组倍率**（以控制台为准）。取数快照：**2026-10-27（北京）**；**以页面实时为准**。

换算约定：基价输入 $= model_ratio×2；输出 $= 输入×completion_ratio；缓存读 $= 输入×cache_ratio。分组估价 = 基价 × group_ratio。

### 基价（每百万 tokens，由价格接口换算）

| 计费项 | 基价（$/1M，未乘分组） |
| --- | --- |
| 输入 | **$0.834** |
| 输出 | **≈ $2.501** |
| 缓存读 | **$0.0417** |

公开接口按 **基价 × group_ratio** 估算即可；**实际扣费以控制台为准**。

### 分组价格估算（基价 × group_ratio，$/1M）

| 分组 | group_ratio | 输入估 | 缓存读估 | 输出估 |
| --- | --- | --- | --- | --- |
| **hunyuan-1** | 2.09 | ≈ $1.743 | ≈ $0.0872 | ≈ $5.227 |

**最终扣费以控制台流水为准**。

---

## 五、调用流程：Key → Base URL → chat/completions

```text
控制台创建 API Key（选本模型 enable_groups 中的分组）
  → Base URL = https://tryallapi.com/v1
  → POST /v1/chat/completions   （model=hy4-preview）
  → 读 choices[0].message.content（或流式 delta）
```

建议：

- **超时**：长输出 / Agent 工作流建议开流式并调高 timeout；
- **密钥**：只放环境变量（如 `TRYALLAPI_KEY`），勿提交仓库；
- **Anthropic 兼容**：价格接口标注支持 `anthropic` 端点类型，具体 Messages 配置以站点说明为准；本文以 OpenAI 兼容为主，不展开完整 Anthropic 示例。

---

## 六、curl / SDK 最小示例

占位 Key `sk-xxx`（**不要**填入真实 Key）。

### 1）OpenAI 兼容 chat/completions

```bash
curl -sS https://tryallapi.com/v1/chat/completions \
  -H 'Authorization: Bearer sk-xxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "hy4-preview",
    "messages": [
      {"role": "user", "content": "把这段日志里的错误码提取成 JSON 数组"}
    ]
  }'
```

### 2）Python（OpenAI SDK）

```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-xxx",  # 建议改为读取环境变量
    base_url="https://tryallapi.com/v1",
    timeout=180,
)

r = client.chat.completions.create(
    model="hy4-preview",
    messages=[{"role": "user", "content": "用三句话总结：如何给这段正则写单元测试边界。"}],
)
print(r.choices[0].message.content)
```

Cursor：Settings → Models → OpenAI API Key + Override Base URL `https://tryallapi.com/v1` → 自定义模型 `hy4-preview`。Dify：OpenAI-API-compatible，endpoint 填 `https://tryallapi.com/v1`。

---

## 七、常见注意

1. **模型名**：请求体 `"model"` 必须写广场上的精确 ID `hy4-preview`（大小写敏感）；写混容易 404。
2. **基价 × 倍率**：估算用价格接口快照；峰谷 / 促销字段若页面另有说明，以控制台为准。
3. **分组**：当前快照可见分组为 `hunyuan-1`；403 多为分组无货，先核对控制台 `enable_groups`。
4. **报错速查**：401 查 Bearer / 令牌；404 确认模型名且 base_url 含 `/v1`；429 退避或换分组。
5. **合规**：第三方聚合不是腾讯云官方服务，敏感代码先看条款。

---

## FAQ

### 混元 Hy4 Preview 的模型 ID 是什么？

TryAllAPI 上是 **`hy4-preview`**。请求体 `"model"` 填该字符串。

### TryAllAPI 的对话 endpoint 是什么？

- Base URL：`https://tryallapi.com/v1`
- 对话：`POST /v1/chat/completions`
- 价格接口同时标注 `anthropic` 端点类型，具体配置以站点说明为准；本文以 OpenAI 兼容为主。

### 基价大概是多少？

按 2026-10-27 价格接口快照：输入约 $0.834 / 输出约 $2.501 / 缓存读约 $0.0417（$/1M，未乘分组）。再 × `group_ratio`（hunyuan-1 = 2.09）得分组估价；**以页面实时为准**。

### 如何试用 TryAllAPI？

站点首页：[https://tryallapi.com/](https://tryallapi.com/)；注册试用入口：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 八、小结与试用入口

把混元 Hy4 Preview 接到业务里，核心是：**在价格页确认 model id 与 OpenAI 兼容 Base URL → 读懂基价 × 分组倍率 → 用 chat/completions 先跑通非流式 ping，再开流式 / 接工具调用**。

| 资源 | 链接 |
| --- | --- |
| TryAllAPI 首页 | [https://tryallapi.com/](https://tryallapi.com/) |
| 注册试用（aff） | [https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A) |
| 价格页 | [tryallapi.com/pricing](https://tryallapi.com/pricing) |

价格取数 2026-10-27（北京），以页面实时为准。

---

## License

本文档以个人接入笔记形式发布，示例代码可自由复用。截图与模型广场文案归原平台；价格以实时页面为准。TryAllAPI 为第三方聚合，非腾讯云官方产品。
