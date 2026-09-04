# DeepSeek · GLM · Qwen 三家视觉多模态模型对比报告

> 调研日期：2026-09-02
> 数据时效：所有信息截至 2026-09-02，主要模型均于 2026 年 8 月发布
> 调研方式：3 个并行研究代理分别检索官方源（API 文档 / 官网 / Hugging Face / 阿里云百炼）

---

## 一、模型名核实（重要）

用户提到的三个名称经官方源核实如下：

| 用户说法 | 官方准确名称 | API 调用 ID | 发布日 | 状态 |
|---|---|---|---|---|
| DeepSeek 视觉模型（截图含 `DeepSeek-V4-Flash-Vision-Exp`） | **DeepSeek-V4-Flash-Vision-Exp** | `deepseek-v4-flash-vision-exp` | 2026-08-21（API）/ 2026-08-31（开源权重） | ✅ 准确 |
| GLM-5.3-flash | **GLM-5.3-Flash** | `glm-5.3-flash` | 2026-08-26 | ✅ 准确 |
| Qwen-3.8-flash | **Qwen3.8-Flash** | `qwen3.8-flash` | 2026-08-26 | ⚠️ 名称有误（是 Qwen3.8，无横杠） |

---

## 二、核心对比表（三家最新旗舰多模态模型）

| 维度 | DeepSeek-V4-Flash-Vision-Exp | GLM-5.3-Flash | Qwen3.8-Flash |
|---|---|---|---|
| 厂商 | DeepSeek | 智谱 AI | 阿里通义千问 |
| 发布日 | 2026-08-21 | 2026-08-26 | 2026-08-26 |
| 定位 | 多模态 Agent 模型 | GLM-5 系列首个原生多模态 | 最新多模态 MoE |
| 参数规模 | 约 3046 亿（304.6B） | 320B 总参 / 18B 激活 | 125B + 51B N-gram Embedding（激活 6B） |
| 上下文 | 1M 输入 / 最大 384K 输出 | 1M / 最大 128K 输出 | 262K 原生，可扩 1M；API 1M |
| 输入模态 | 文本 + 图片 | 文本 + 图片 + 视频 + 文件 | 文本 + 图片 + 视频 |
| 输出模态 | 文本 | 文本 | 文本 |
| 开源 | 是（MIT） | 是（MIT） | 主模型 API；开源 `Qwen3.8-Flash-Next`（Qwen Community License 1.0） |
| 后端接入 | OpenAI / Anthropic 兼容 | Chat Completion API（`image_url`） | 阿里云百炼，OpenAI / Anthropic / DashScope 兼容 |
| 输入价（元/M） | 峰值 $0.44/M（≈3.1 元） | 0.8 | 0.15（≤32k）/0.3/0.6 |
| 输出价（元/M） | 峰值 $1.32/M（≈9.4 元） | 2.8 | 1.5/3/6 |

> 说明：DeepSeek 官方以美元计费（非峰值减半），上表按峰值折算人民币约数；Qwen3.8-Flash 输入价分段（32k 内 / 32–128k / 128–256k）。

---

## 三、逐个模型详解

### 3.1 DeepSeek-V4-Flash-Vision-Exp
- 这是 DeepSeek **V4 系列首个实验性多模态模型**，API 于 2026-08-21 上线，开放权重于 2026-08-31，许可证 MIT，仓库约 168GB / 48 个 safetensors 分片（[官方 API 新闻](https://api-docs.deepseek.com/news/news260821/)、[开源报道](https://aiidelist.com/blog/deepseek-v4-flash-vision-open-weights)）。
- 图片输入支持三种方式：Base64 内联、外部 URL（最长 8192 字符）、Files API（单图可达 64MiB）（[官方 Vision 文档](https://api-docs.deepseek.com/guides/vision/)）。
- 图片转 Token 规则：每张图自动重采样至约 800×800，单图上限 384 tokens，单请求最多 600 张图（[官方 Vision 文档](https://api-docs.deepseek.com/guides/vision/)）。
- 官方定位是多模态 Agent 模型，公布的多模态 Agent 基准（对比 Opus-4.8）：ApexBench Pass@1 36.5、Agents' Last Exam 27.3、Chartography 64.3、ZeroBench Pass@5 35.0。**注意：这些为厂商自测数据，尚未经独立复测**（[腾讯新闻](http://news.qq.com/rain/a/20260901A03YV500)、[开源报道](https://aiidelist.com/blog/deepseek-v4-flash-vision-open-weights)）。
- 部署注意：开放权重需多节点张量并行（MP=4 起步），官方参考实现为基础 PyTorch，**未确认** vLLM / SGLang / Ollama 正式支持。
- 说明：早期开源的 **DeepSeek-VL2**（2024-12 发布，上下文仅 4096）已被本模型取代；**DeepSeek-V3 / V3.1 为纯文本模型，不支持图片输入**。

### 3.2 GLM-5.3-Flash
- **GLM-5 系列首个原生多模态模型**，2026-08-26 上线，此前曾以「Ox-Alpha / 牛来」为名匿名发布并冲至调用榜第一（[官方文档](https://docs.bigmodel.cn/cn/guide/models/vlm/glm-5.3-flash)、[钛媒体](https://www.tmtpost.com/8118344.html)）。
- 采用线性注意力 + 稀疏注意力混合架构，总参 320B / 激活 18B，为**首个采用该混合架构的开源前沿模型**；Artificial Analysis Intelligence Index 57 分，与 Claude Opus 4.8 持平（[官方文档](https://docs.bigmodel.cn/cn/guide/models/vlm/glm-5.3-flash)）。
- 视觉应用场景丰富：前端 UI Coding、Office 文档交付、金融研究、视频剪辑、3D Blender 场景构建、游戏开发、CAD 视觉复刻（[官方文档](https://docs.bigmodel.cn/cn/guide/models/vlm/glm-5.3-flash)）。
- 定价约为 GLM-5.3（8/28 元）的 1/10，约为 Claude Opus 4.8 的 1/40（[智谱开放平台价格](https://open.bigmodel.cn/pricing)）。
- 智谱当前视觉家族全景：`glm-5.3-flash`（旗舰）＞ `glm-4.6v`（106B/12B，视觉 SOTA 推理）＞ `glm-4.6v-flash`（9B，免费）＞ `glm-4.5v`（开源）；老款 `glm-4v` 系列已边缘化（[官方文档](https://docs.bigmodel.cn/cn/guide/models/vlm/glm-4.6v)）。

### 3.3 Qwen3.8-Flash
- 阿里通义千问最新多模态 MoE 大模型，2026-08-26 发布，调用 ID `qwen3.8-flash`（[阿里云百炼](https://help.aliyun.com/zh/model-studio/qwen3-8-flash)、[IT之家](http://m.toutiao.com/group/7678319425862320680/)）。
- 主模型 125B + 51B N-gram Embedding，每 token 激活 6B；原生 262K 上下文，可通过 YaRN 扩至 1M（[阿里云百炼](https://help.aliyun.com/zh/model-studio/qwen3-8-flash)）。
- 能力：多模态理解（编程辅助、智能体协作、图文理解、图表与长视频分析）、Function Calling、结构化输出、联网搜索、上下文缓存；最大输入 991,808、最大输出 131,072（[阿里云百炼](https://help.aliyun.com/zh/model-studio/qwen3-8-flash)）。
- 开源情况：开源的是 **Qwen3.8-Flash-Next**（Qwen4 架构预览版），许可证 Qwen Community License 1.0（[亿邦动力](https://m.ebrun.com/700429.html)、[阿里研究院](https://m.sohu.com/a/1068184306_384789/)）。
- 若需完全开源自部署，Qwen 家族的 **Qwen3-VL** 系列（2B~235B 六种规模，256K 可扩 1M，支持 20 分钟视频 + 秒级定位 + 100+ 语言 OCR）是当前视觉开源旗舰，宽松开源协议利于商用（[Qwen VL 官网](https://qwen3lm.com/qwen-vl/)）。

---

## 四、横向结论与选用建议

| 关注点 | 推荐 |
|---|---|
| 极致上下文 + 开源权重（本地/私有化） | GLM-5.3-Flash（1M + MIT） |
| 纯视觉 Agent 基准 / 多模态编程 | DeepSeek-V4-Flash-Vision-Exp |
| 性价比 + 超长视频分析 | Qwen3.8-Flash（输入 0.15 元起） |
| 完全免费视觉 API | 智谱 `glm-4.6v-flash`（9B 免费） |
| 开源自部署 | Qwen3-VL 系列（6 规模、支持量化） |

- **DeepSeek**：参数最大（304B），1M 上下文，MIT 开源，但官方仅提供基础 PyTorch 推理参考，部署门槛高，且视觉基准为厂商自测。
- **GLM**：320B/18B 激活效率最高，混合架构，1M 上下文，MIT 开源，视觉场景覆盖广，性价比（0.8/2.8 元）在旗舰中很突出。
- **Qwen**：上下文与视频能力均衡（262K→1M，长视频分析），输入价最低（0.15 元起），生态接入最成熟（阿里云百炼多 region + 多协议兼容），但主模型不开源，开源的是 Next 预览版。

> 提醒：三家厂商公布的基准（尤其 DeepSeek 多模态 Agent 分）多为自测数据，建议选型前用真实业务样本做独立 A/B 验证；数据如超 6 个月需重新核实。

---

## 五、信息来源

- DeepSeek：官方 [API 新闻页](https://api-docs.deepseek.com/news/news260821/)、[Vision 指南](https://api-docs.deepseek.com/guides/vision/)、[Models & Pricing](https://api-docs.deepseek.com/quick_start/pricing/)、[DeepSeek-VL2 GitHub](https://github.com/deepseek-ai/DeepSeek-VL2/)、[更新日志](https://api-docs.deepseek.com/zh-cn/updates/)；[aiidelist 开源报道](https://aiidelist.com/blog/deepseek-v4-flash-vision-open-weights)、[腾讯新闻](http://news.qq.com/rain/a/20260901A03YV500)
- 智谱 GLM：官方 [GLM-5.3-Flash 文档](https://docs.bigmodel.cn/cn/guide/models/vlm/glm-5.3-flash)、[GLM-4.6V 文档](https://docs.bigmodel.cn/cn/guide/models/vlm/glm-4.6v)、[GLM-4.5V 文档](https://docs.bigmodel.cn/cn/guide/models/vlm/glm-4.5v)、[新品发布](https://docs.bigmodel.cn/cn/update/new-releases)、[开放平台价格](https://open.bigmodel.cn/pricing)；[钛媒体](https://www.tmtpost.com/8118344.html)、[新浪科技](https://finance.sina.com.cn/tech/digi/2026-08-27/doc-inipucsw0795908.shtml)
- 阿里 Qwen：官方 [qwen3.8-flash 模型信息](https://help.aliyun.com/zh/model-studio/qwen3-8-flash)、[视觉理解模型列表](https://help.aliyun.com/zh/model-studio/model-list-visual-understanding/)、[通义千问 VL 使用指南](https://help.aliyun.com/zh/dashscope/developer-reference/getting-started-with-tongyi-qianwen-vl)、[Qwen VL 官网](https://qwen3lm.com/qwen-vl/)；[IT之家](http://m.toutiao.com/group/7678319425862320680/)、[阿里研究院](https://m.sohu.com/a/1068184306_384789/)、[亿邦动力](https://m.ebrun.com/700429.html)
