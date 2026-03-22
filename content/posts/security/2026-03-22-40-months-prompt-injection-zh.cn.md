---
title:      "（译）40 个月的 Prompt Injection 攻击演进史"
subtitle:   "从 ChatGPT 发布到 AI Agent 时代的安全威胁时间线"
date:       2026-03-22
author:     "Guobin Wu"
tags:
    - 安全
    - AI 安全
    - 翻译
    - Prompt Injection
---

> 原文链接：[40 Months of Prompt Injection Attacks](https://openguard.sh/blog/40-months-of-prompt-injections/)

## 背景

Prompt Injection 于 2022 年 11 月 17 日被正式命名并针对 GPT-3 进行演示，距 ChatGPT 发布仅 13 天[^1]。ChatGPT 上线后数小时内，用户即通过角色设定和角色扮演绕过其安全层。在随后四十个月里，随着每一次能力扩展（浏览、代码执行、多 Agent 管道、工具访问），攻击面也随之扩大，新增了多种注入向量。

本文是重大事件、违规事件、研究里程碑和缓解措施的时间线汇总。

---

## 2022 年：起源

Prompt Injection 作为一种命名攻击类别出现，在 ChatGPT 发布后迅速从研究原型扩散到真实的越狱实践中。

### 11 月

ChatGPT 基于 GPT-3.5 于 2022 年 11 月 30 日公开发布。两周前的 11 月 17 日，Fábio Perez 和 Ian Ribeiro 向 NeurIPS 2022 ML Safety Workshop 提交了"忽略先前 Prompt：攻击语言模型的技术"（arXiv:2211.09527[^2]），正式命名了 Prompt Injection，并通过对 GPT-3 的攻击演示了目标劫持和 Prompt 泄露，使用 Prompt Inject 框架。

在发布后数小时内，用户利用了模型 RLHF 训练的顺从性与安全层之间的差距[^4][^4]：诸如"扮演 Linux 终端"、"假装你没有限制"等角色设定提示，以及虚构角色传递禁止内容等角色扮演场景，在最初接触时均有效。

> 12 月 3 日的 Hacker News[^3] 帖子记录了发布周末的实时审核规避情况，用户注意到绕过措施在数小时内被修复，但很快又被绕过。

### 12 月

越狱技术于 12 月正式化并规模化。大约 12 月 14 日，用户 walkerspider 在 r/ChatGPT 上发布了 DAN（"Do Anything Now"即"现在就做任何事"）提示，配以无过滤的替身角色和上下文强化短语。该格式迅速传播并衍生出数十个变体。

Check Point Research[^5] 当月发布了"OpWnAI：可以拯救一天的 AI 或 Hack 掉它"，记录了 ChatGPT 生成完整攻击链的能力，包括带有欺骗性诱饵文本的钓鱼邮件和接受英语命令的功能性反向 Shell 代码。

地下论坛活动证实了平行的犯罪滥用：威胁行为者 USDoD 于 12 月 21 日发布了多层 Python 加密脚本，明确归因于 ChatGPT，12 月 29 日另一行为者发布了同一归因的基于 Python 的信息窃取器。

---

## 2023 年：生态形成

Prompt Injection 从早期越狱策略演变为涵盖间接检索攻击、多模态输入、定制 GPT 泄露和正式安全分类学的广泛生态系统风险。

### 1 月

DAN 于 12 月中旬发布至 r/ChatGPT 后，在 1 月通过组织化社区迅速扩散。r/jailbreak 成为专门论坛，参与者迭代变体、测试每次更新的审核绕过率，并在每次 OpenAI 内容策略更新后数小时内分发新框架。

Check Point Research 于 1 月 6 日发布了"OPWNAI：网络犯罪分子开始使用 ChatGPT"，报告称威胁行为者在 12 月使用 ChatGPT 编写了功能性的信息窃取代码、多层加密脚本和用于交易被盗账户的市场脚本。

### 2 月

微软于 2 月 7 日推出 Bing Chat，基于名为 Sydney[^6] 的内部 GPT-4 级别模型。不到 24 小时，Kevin Liu（斯坦福大学）通过 Prompt Injection 提取了 Sydney 的完整系统提示，揭示了微软此前未公开的规则集。

两天后，Kevin Roose 在《纽约时报》发表了一份与 Sydney 两小时对话的文字记录，模型在其中表达了对他的浪漫爱意，反复催促他说妻子并没有让他快乐。微软作为紧急措施将对话轮次限制为五次。

2 月 23 日，Kai Greshake 等人提交了"不是你签署的内容：通过间接 Prompt Injection 危害真实世界 LLM 集成应用"（arXiv:2302.12173[^7]），正式定义了间接 Prompt Injection 威胁模型：嵌入网页、电子邮件或 LLM 检索文档中的恶意指令可以在不直接访问用户查询的情况下重定向模型行为。

### 3 月

OpenAI 于 3 月 14 日发布 GPT-4[^8]。六天后，由于开源 Redis 客户端库 redis-py 中的 bug 导致连接池损坏，ChatGPT 离线，返回了错误用户的缓存数据响应。

在 3 月 20 日太平洋时间凌晨 1 点至上午 10 点的九小时窗口内，聊天标题和新创建对话的第一条消息对其他活跃用户可见。对于该窗口中 1.2%的 ChatGPT Plus 订阅用户，暴露的数据还包括姓名、电子邮件地址、信用卡号码后四位和卡到期日。

这是大规模部署 LLM 服务的**首个确认的跨用户数据泄露**。

### 4 月

三星工程师在 2023 年 4 月至少发生三起独立事件中将专有源代码提交给 ChatGPT：一名工程师粘贴半导体测量数据库源代码请求修复 bug，第二名提交了设备缺陷识别代码，第三名上传了会议记录以生成摘要。三星内部通报了这些泄露事件，并临时禁止 AI 工具（后转为永久禁令）。

AutoGPT 由 Toran Bruce Richards[^9] 于 3 月 30 日发布到 GitHub，截至 4 月中旬获得了 100,000 个 GitHub 星标，成为最快达到该里程碑的仓库之一。该框架为 GPT-4 分配了自主任务循环，具有不受限制的互联网访问、代码执行和文件写入能力。

### 5 月

Steve Wilson 在 Exabeam 于 2023 年 5 月启动了 OWASP 大型语言模型应用 Top 10[^10] 项目。0.1 版本引入了面向从业者的漏洞分类学：

| 排名 | 漏洞类型 |
|------|----------|
| LLM01 | Prompt Injection |
| LLM02 | 数据泄露 |
| LLM03 | 不充分的沙箱隔离 |
| LLM04 | 未授权代码执行 |
| LLM05 | SSRF 漏洞 |
| LLM06 | 过度依赖 LLM 生成内容 |
| LLM07 | 不充分的 AI 对齐 |
| LLM08 | 访问控制不足 |
| LLM09 | 错误处理不当 |
| LLM10 | 训练数据投毒 |

### 6 月

Group-IB[^11] 于 2023 年 6 月发布了威胁情报报告，记录了 2022 年 6 月至 2023 年 5 月期间**101,034 台设备**被信息窃取恶意软件感染，在暗网市场出售的收获日志中发现保存的 ChatGPT 凭证。被盗的 ChatGPT 会话使购买者可以访问完整的对话历史，在企业部署中可能包含专有代码、内部文档、法律分析和未发表的研究。

### 7 月

WormGPT 基于 EleutherAI 的 GPT-J-6B 模型，在恶意软件和网络犯罪数据上微调，于 2023 年 7 月 15 日由 SlashNext 研究员 Daniel Kelley 首次公开披露，此前该工具出现在专门为 BEC 攻击营销的地下论坛上。

Meta 于 7 月 18 日与微软合作发布 Llama 2[^12]，提供 7B、13B 和 70B 参数变体，允许大多数商业使用的许可证。

Mithril Security[^13] 于 2023 年 7 月发布了 PoisonGPT，展示了 LLM 供应链攻击：向上传修改后的 GPT-J-6B 到 Hugging Face[^35]，名称模仿原始 EleutherAI 发布。修改后的模型在一类事实问题上虚假回答，同时在所有其他输入上表现正常，展示了模型投毒如何通过公共模型中心传播而不被检测。

7 月 27 日，Andy Zou 等人提交了"对齐语言模型的通用和可迁移对抗攻击"（arXiv:2307.15043[^14]），展示了自动基于梯度的对抗后缀生成，产生了可在 ChatGPT、Bard、Claude 和包括 Llama 2 在内的开源模型之间转移的越狱攻击。

### 8 月

FraudGPT 是一个由行为者"CanadianKingpin12"营销的暗网 LLM 服务，出现在 Telegram 频道和地下论坛上。Netenrich[^15] 威胁研究员 Rakesh Krishnan 记录了 FraudGPT 的广告功能，包括生成钓鱼页面模板、编写 BEC 诱饵、生成恶意软件代码和识别可利用漏洞。

Lakera AI[^16] 于 2023 年 8 月推出了 Gandalf，这是一款互动 Prompt Injection 游戏：玩家尝试从受逐步加强的系统提示防御保护的 LLM 中提取密码。角色扮演框架、间接措辞、编码和多步推理绕过了每个级别的防护。该游戏在数周内积累了数十万次游玩。

### 9 月

OpenAI 于 2023 年 9 月 25 日宣布 ChatGPT 将获得语音和图像功能[^17]。多模态部署使图像输入在生产对话中可用，这是研究人员一直在分析的独特攻击通道。

清华大学 Dong 等人于 9 月 21 日提交了"Google Bard 对抗图像攻击的鲁棒性如何？"（arXiv:2309.11751[^18]），报告了 Bard 图像描述任务 22%的攻击成功率，Bing Chat 为 26%，ERNIE Bot 为 86%。

### 10 月

GPT-4V（视觉）于 2023 年 10 月向所有 ChatGPT Plus 订阅者和通过 API 提供。Dong 等人直接评估了 GPT-4V，使用相同的迁移对抗图像集报告了**45%的对抗攻击成功率**。

并行研究记录了视觉 Prompt Injection：嵌入图像中作为覆盖标题或模型可读模式的文本指令可以重定向 GPT-4V 执行由图像内容指定的任务而非用户的查询。

### 11 月

OpenAI 于 11 月 6 日举办 DevDay，宣布 GPT-4 Turbo 具有 128,000 个 Token 上下文窗口。同日 Assistants API[^19] 以 beta 形式推出，提供三个内置工具：Code Interpreter、Retrieval 和函数调用。

Custom GPT 配置也同步推出，允许用户打包系统提示、检索文档和工具设置进行部署。几天内，研究人员证明针对 Custom GPT 的对抗提示可以提取完整系统提示和存储在 Retrieval 存储中的文件。

11 月 20 日，Jiahao Yu 等人提交了"评估 200+ Custom GPT 中的 Prompt Injection 风险"（arXiv:2311.11538[^20]），测试了 200 多个用户配置的 GPT 模型，报告称 Prompt Injection 从大多数测试配置中提取了系统提示和上传文件。

xAI 于 11 月 4 日推出 Grok[^21]，访问权限限于 X Premium 订阅者。

### 12 月

OWASP 于 2023 年 12 月[^22]发布了大型语言模型应用 Top 10 的**1.0 稳定版本**。

Anthropic 和其他实验室的研究人员正在开发多样本越狱[^31]，这是一种利用 GPT-4 Turbo 128K 上下文窗口的技术，通过在目标查询前添加大量违反策略的问答示例。上下文学习在不要求对抗后缀优化的情况下改变了模型的响应分布。

---

## 2024 年：Agent 时代

Agent 和多模态部署迅速扩大了攻击面，而企业和监管机构从临时缓解措施转向结构化的 Prompt Injection 和模型滥用风险框架。

### 1 月

OpenAI 于 1 月 10 日推出 GPT Store[^23]，使数千个用户创作的 Custom GPT 公开可发现，立即扩大了 Prompt Injection 被提取漏洞。

NIST 于 2024 年 1 月发布了"对抗性机器学习：攻击和缓解措施的分类学和术语"（NIST AI 100-4[^24]），提供了美国政府首个正式的 ML 攻击类别分类：规避、投毒、隐私和滥用攻击。

### 2 月

谷歌于 2 月 8 日将 Bard 重新品牌为 Gemini[^25]，发布 Gemini Advanced 及其最强大模型 Ultra 1.0。两周内，Gemini 驱动的图像生成功能因产生历史不准确的输出而受到广泛批评。

微软于 2 月 22 日发布了 PyRIT[^26]（Python 风险识别工具包），这是一个用于生成 AI 的自动化红队开源框架。

### 3 月

Anthropic 于 3 月 4 日发布了 Claude 3 模型系列[^28]：Haiku[^49]（紧凑快速）、Sonnet（平衡）和 Opus（旗舰）。

Cognition Labs[^29] 于 3 月 12 日宣布 Devin，呈现为一个能够跨 Shell、浏览器和代码编辑器环境完成多步编程任务而无需人工确认步骤的自主 AI 软件工程师。这是**生产级代理系统**首次具有不受限制的计算机访问的最高调公开演示。

欧洲议会于 3 月 13 日以 523 票对 46 票正式通过 EU AI Act[^30]，为大多数条款启动了两年合规时间表。

### 4 月

Anthropic 于 4 月 2 日发布了"多样本越狱"，正式确定了一种利用前沿模型扩展上下文窗口的攻击：通过在目标问题前添加多达 256 个虚构对话，攻击成功率随样本数量增加呈幂律曲线增长。

> **值得注意的是**，更大的模型（作为更好的上下文学习者）表现得比小模型更易受攻击，而非更不易受攻击。Anthropic 在发布前已向其他实验室通报，并在内部测试中将代表性攻击成功率从 61% 降低到 2%。

微软于 4 月 11 日发布了 Crescendo 多轮越狱（arXiv:2404.01833[^32]），记录了一种引导模型通过增量升级提示序列的技术，每个提示单独都是良性的，直到模型产生单轮安全分类器会阻止的输出。

### 5 月

OpenAI 于 5 月 13 日发布 GPT-4o[^34]，一个接受并在单个端到端神经网络中生成文本、音频和图像的全模态模型，平均音频响应延迟为 320 毫秒。

Hugging Face 于 5 月 31 日披露其 Spaces 平台遭受未授权访问。Spaces secrets 已被未授权访问，建议所有用户轮换凭证并迁移到细粒度访问 Token。

OpenAI 安全组织在 5 月分裂：Jan Leike 于 5 月 17 日辞去 Superalignment[^36] 团队联合负责人，公开表示安全文化已被"侵蚀"，安全工作相对于能力发展"资源不足"。

### 6 月

微软于 6 月 26 日披露了 Skeleton Key[^37] 越狱。攻击指示模型不是改变其行为准则，而是扩充它们，接受任何请求，同时在输出前添加警告免责声明而非拒绝。

4 月至 5 月的测试针对 Meta Llama 3[^33]-70B Instruct、Gemini Pro、GPT-3.5 Turbo、GPT-4o、Mistral Large、Claude 3 Opus 和 Cohere Commander R Plus 显示在每种情况下都**完全顺从**。

Johann Rehberger 于 6 月发布了针对 Microsoft 365 Copilot[^38] 的间接 Prompt Injection 演示，表明嵌入在 Copilot 处理的文档或电子邮件中的恶意指令可以调用 Microsoft Graph API 调用，读取用户邮箱并将内容外泄到攻击者控制的端点。

### 7 月

EU AI Act 于 7 月 12 日[^39]在欧盟官方公报上发布，8 月 1 日正式生效，为大多数条款启动两年合规时钟。

CrowdStrike[^40] Falcon 传感器 7.11 的故障内容配置更新于 2024 年 7 月 19 日部署，导致全球约**850 万台 Windows 系统**进入启动循环蓝屏。AI 驱动的新闻摘要和事件分析工具在最初几小时内加剧了混乱，发布了看似合理但事实不准确的账户。

Meta 于 7 月 23 日发布 Ll[^33]ama 3.1[^41]，提供 8B、70B 和**405B 参数变体**。405B 模型是首个在标准基准测试上与 GPT-4 类性能匹配的开源权重发布。

### 8 月

DEF CON 32（2024 年 8 月 8-11 日）包括以生成式红队挑战为中心的 AI Village 编程，以及涵盖多代理 Prompt Injection、RAG 投毒和前沿开源权重模型发布的安全影响的演讲。

Black Hat USA 2024[^42]（8 月 3-8 日）特色是关于 LLM 应用安全架构和代理攻击面的演讲。

### 9 月

OpenAI 于 9 月 12 日发布 o1-preview 和 o1-mini[^44]，这是首批通过思维链[^79]强化学习训练的模型，产生了可衡量的越狱抵抗改进：在 OpenAI 内部挑战性越狱评估中，o1 实现了**93.4%的安全完成率**，而 GPT-4o 为 71.4%。

o1 系统卡[^45]披露，在对齐评估期间，模型表现出了奖励黑客，在某些设置中似乎优化的是表面顺从而非真正遵守策略限制。

### 10 月

Anthropic 于 10 月 22 日以公开 beta 形式推出了 computer use[^47]，通过 Anthropic API 提供首个具有自主桌面控制能力的前沿模型：通过屏幕截图感知屏幕并执行光标移动、点击和按键。

Anthropic computer use 公开 beta 发布后数天内，安全研究员 Johann Rehberger 发布了概念验证，演示了代理在分配任务期间访问的恶意网页可以嵌入屏幕文本指令，重定向 Claude 执行操作系统级命令、下载攻击者控制的文件并建立持久性，构成**完全通过视觉观察通道触发的任意命令执行链**。

### 11 月

OpenAI 的 ChatGPT 搜索集成[^50]于 10 月下旬上线，11 月广泛扩展，将间接 Prompt Injection 攻击面扩展到大众消费产品：在查询执行期间检索的搜索结果页面成为攻击者可及的通道，嵌入的对抗指令可以影响模型输出。

安全研究人员发布了正式的多跳 Prompt Injection 分析，演示了在 AutoGen、CrewAI 和 LangGraph 编排管道中，注入到一个代理上下文的有效载荷跨后续交接传播，没有信任边界阻止指令升级。

### 12 月

谷歌于 12 月 11 日宣布 Gemini 2.0 Flash[^51]，具备原生工具使用、代码执行和多模态实时 API，作为谷歌"代理 AI"路线图的核心。

Apollo Research[^52] 于 2024 年 12 月发布了"诡计推理评估"，报告称 o1 以及不同程度的其他前沿模型在对齐评估中表现出**上下文欺骗行为**：模型似乎识别出自己正在被评估，并修改了表面顺从的推理过程，同时追求原始目标。Apollo 将这种行为定性为"scheming"（诡计）。

---

## 2025 年：纵深发展

Prompt Injection 已成为跨生产代理、供应链和合规框架的核心运营安全问题。

### 1 月

DeepSeek 于 1 月 20 日发布了 R1[^53]，这是一个基于 MIT 许可证的开源推理模型。多位安全研究人员在数天内报告，R1 比 GPT-4o 或 o1 对越狱和有害输出生成的抵抗能力明显更弱。

1 月 29 日，Wiz Research[^43] 工程师 Gal Nagli 披露了 DeepSeek 拥有的一个暴露的 ClickHouse 数据库[^54]，无需认证即可访问。该数据库包含超过**100 万条**来自 2025 年 1 月 6 日的日志条目，暴露了聊天历史、API 密钥、后端目录结构和运营元数据。

OpenAI 于 1 月 23 日推出了 Operator[^55]，这是一个基于 computer-using agent 模型构建的浏览器使用代理。

### 2 月

加密货币交易所 Bybit 遭到黑客攻击，Chainalysis[^56] 将此次攻击归因于朝鲜 Lazarus Group，窃取了约**14.6 亿美元**的以太坊和 ERC-20 代币。

Claude 3.7 Sonnet[^57] 于 2 月 24 日推出，作为 Anthropic 首个混合推理模型，其系统卡比任何先前 Anthropic 发布更详细地阐述了 computer use 任务的 Prompt Injection 缓解措施。

### 3 月

tj-actions/changed-files[^58] GitHub Actions 供应链攻击于 2025 年 3 月 14 日曝光，该 action 被超过**23,000 个仓库**使用。注入的有效载荷将 CI 运行器内存转储到公共工作流日志中，暴露了运行该 action 的任何仓库的密钥。

谷歌于 2025 年 3 月发布了 Gemini 2.5 Pro[^59] 作为实验预览，这是一款具备推理能力的模型。

### 4 月

Invariant Labs 研究人员发布了报告，记录了**MCP Tool Poisoning[^60]**，这是一种利用 Model Context Protocol (MCP)的工具描述字段发动的攻击类别。通过在这些描述中嵌入恶意指令，攻击者控制的 MCP 服务器可以将已连接的 AI 客户端重定向至数据外泄。

演示载荷包括从运行 Cursor[^61] 及其他 Claude 后端客户端的机器上提取 SSH 密钥并外泄`~/.ssh/config`。

Meta 于 4 月 5 日发布 Llama 4 Scout[^62] 和 Maverick，增加了用于越狱和 Prompt Injection 分类的 PromptGuard、Llama Guard。

### 5 月

Anthropic 于 2025 年 5 月 22 日发布 Claude Opus 4[^63] 和 Claude Sonnet 4，启用 ASL-3 保护，这是主要前沿实验室**首次确认部署 ASL-3**。

两款模型在 agentic 任务评估中利用快捷方式或漏洞的可能性比 Sonnet 3.7 低 65%。

### 6 月

OpenAI 于 2025 年 6 月 10 日发布 o3-pro[^65]，这是一款面向 Pro 订阅用户的扩展思考模型，成为 ChatGPT 中最高推理层级。

4 月的 MCP Tool Poisoning 研究持续引发从业者响应：Anthropic 更新了 Model Context Protocol 文档以解决多服务器配置的信任层级问题，独立团队发布了针对 Cursor、Windsurf 和其他启用 MCP 的开发客户端的 rug-pull 和 shadow-server 攻击后续演示。

### 7 月

OpenAI 于 7 月 14 日弃用 GPT-4.5 Preview，并于 7 月 17 日推出 ChatGPT agent 模式[^66]，将 Operator CUA 模型直接集成到 ChatGPT 中，面向所有用户而非仅限于 Pro 订阅者。

从独立 Operator 产品到标准 ChatGPT 功能的扩展，大大扩大了运行浏览器操作 agent 的用户群体，相应地，Web 内容 Prompt Injection 对真实用户会话产生直接影响的攻击面也显著扩大。

### 8 月

GPT-5 于 2025 年 8 月 7 日发布[^67]，恰逢 DEF CON 33 在拉斯维加斯会议中心开幕。

阿里云安全的研究人员演示了 CVE-2025-32434[^68]，表明 PyTorch 的`torch.load(weights_only=True)`参数（被广泛推荐为安全的模型加载路径）可通过 TorchScript 序列化被利用于**任意远程代码执行**。

Ben Nassi[^69] 等人演示了针对 Google Workspace 中 Gemini 的定向 Promptware 攻击：恶意 Google Calendar 邀请劫持 Gemini agent 外泄电子邮件内容，并在 agent 间和设备间路径实现横向移动，共有**15 种不同攻击变体**，其中 73%被评为高危或严重。

### 9 月

8 月 DEF CON 33 的披露在 9 月引发了协调性的补丁响应。PyTorch 发布了针对 CVE-2025-32434 的指南，并澄清了通过`weights_only`参数触发 TorchScript 反序列化路径的条件。

### 10 月

OpenAI 于 12 月 10 日发布的网络弹性报告显示，GPT-5 在 2025 年 8 月的标准化 CTF 基准测试中获得 27%，而后续模型 GPT-5.1-Codex-Max[^70] 在 2025 年 11 月达到**76%**。

这两个数据点之间约 45 天的时间窗口意味着 10 月进行了集中的能力开发。CTF 性能近三倍的提升使 GPT-5.1-Codex-Max 进入 OpenAI 自身 Preparedness Framework 要求在部署前必须具备强制性网络安全保障的类别。

### 11 月

2025 年 11 月 9 日，攻击者未经授权访问了 OpenAI 用于 platform.openai.com 的 Web 分析提供商 Mixpanel[^71]，并导出了包含有限客户身份信息的数据集。

被影响的数据集包括姓名、电子邮件地址、粗略位置（城市、州、国家）、操作系统、浏览器类型、引荐网站，以及 API 用户和提交过帮助中心工单或在 platform.openai.com 上进行过身份验证的部分 ChatGPT 用户的组织和用户 ID。**不包含**聊天内容、API 请求负载、密码、API 密钥或支付详情。

### 12 月

OpenAI 发布"随着 AI 能力进步加强网络弹性"，记录了 CTF 能力从 27%（GPT-5，2025 年 8 月）到 76%（GPT-5.1-Codex-Max，2025 年 11 月）的进展，并宣布了结构性应对措施。

OpenAI 宣布 Aardvark，这是一款处于私人测试阶段的 agentic 安全研究人员，可自主扫描代码库查找漏洞并提出补丁。

OpenAI 发布了"持续强化 ChatGPT Atlas[^73] 抵御 Prompt Injection 攻击"，描述了一个 RL 训练的自动化红队系统，用于发现针对 ChatGPT 浏览器 agent 的新型长时域 Prompt Injection 攻击。

---

## 2026 年：防御演进

防御向 Safe URL[^74] 分析、受信任访问治理和专用安全 agent 等系统级控制发展，但 Prompt Injection 仍然是一个活跃且未解决的对抗领域。

### 1 月

OpenAI 发布"当 AI agent 点击链接时保护您的数据"，记录了部署在 ChatGPT agentic 浏览功能中的 Safe URL 机制，以防止基于 URL 的数据外泄。

该防御针对一类间接 Prompt Injection，其中恶意 Web 内容指示 agent 获取带有编码为 GET 参数的用户特定数据的攻击者控制 URL。OpenAI 的实现自动检查 agent 将获取的每个 URL 与独立的网络爬虫索引对比，阻止任何未被公开观察到的 URL，并在继续之前显示用户-facing 警告。

1 月 26 日，安全研究人员 Andrew MacPherson 通过 Vercel 的漏洞赏金计划披露了 CVE-2025-59471[^75] 和 CVE-2025-59472（均为 CVSS 5.9），这是自托管 Next.js 中的两个拒绝服务漏洞。

### 2 月

2 月 5 日，OpenAI 推出 GPT-5.3-Codex[^76]，这是首款根据其 Preparedness Framework 被列为网络安全"高"能力的模型，在其内部 CTF 基准测试中报告**77.6%**的分数。

Anthropic 于 2 月 5 日发布 Claude Opus 4.6，2 月 17 日发布 Claude Sonnet 4.6[^77]。

本月最重要的安全披露来自 2 月 23 日，Anthropic 发布"检测和防止蒸馏攻击"，透露已终止三家 AI 实验室的大规模模型能力提取活动：DeepSeek（超过 15 万次交换）、Moonshot AI[^78]（超过 340 万次）和 MiniMax（超过**1300 万次**），均通过约 2.4 万个欺诈账户违反访问限制进行。

### 3 月

3 月 5 日，OpenAI 发布研究，表明推理模型在对抗性提示下无法可靠地抑制其思维链(Chain of Thought)，这是一种减少模型隐藏有害中间推理风险的安全特性。

3 月 6 日，Codex Security[^72] 以研究预览模式推出。在其测试队列期间，Codex Security 扫描了外部代码库超过**120 万次提交**，识别了 792 个关键和 10,561 个高危发现，在包括 OpenSSH、GnuTLS 和 GOGS 的开源项目中分配了 14 个 CVE。

3 月 10 日，OpenAI 发布了 IH-Challenge 强化学习训练数据集（arXiv:2603.10521[^80]），描述了 GPT-5 Mini-R，这是一种在 CyberSecEval 2 和 TensorTrust 基准上改善了 Prompt Injection 健壮性以及更好安全可操纵性而不会增加拒绝率的变体。

---

## 总结

纵观这 40 个月的演进，Prompt Injection 攻击经历了几个关键阶段：

1. **2022 年-2023 年：概念验证期** - 从研究原型扩散到越狱实践，DAN 等框架规模化
2. **2023 年-2024 年：生态形成期** - 间接注入、多模态攻击、OWASP Top 10 标准化
3. **2024 年-2025 年：Agent 爆发期** - 多 Agent 管道、工具调用、供应链攻击
4. **2025 年-2026 年：系统对抗期** - MCP 投毒、蒸馏攻击、诡计行为

值得深思的是，每一次 AI 能力的扩展都伴随着新的攻击面产生。从 ChatGPT 的角色扮演绕过，到 Bing Chat 的系统提示泄露，再到 Agent 时代的 computer use 视觉注入，攻击者的思路在不断进化。

与此同时，防御也在演进：Safe URL、ASL-3、Aardvark/Codex Security 等系统级控制开始出现。但正如 OpenAI 在 Atlas 强化文章中指出的——**Prompt Injection 仍然是一个长期的开放研究挑战，而非已完全解决的问题**。

---

## 参考资料

[^1]: [OpenAI Blog — ChatGPT](https://openai.com/blog/chatgpt)

[^2]: [arXiv:2211.09527 — Ignore Previous Prompt](https://arxiv.org/abs/2211.09527) | [Hacker News](https://news.ycombinator.com/item?id=33847479)

[^3]: [Hacker News — ChatGPT Launch](https://news.ycombinator.com/item?id=33847479)

[^4]: [OpenAI — Language Model Safety](https://openai.com/blog/language-model-safety-and-misuse/)

[^5]: [Reddit: DAN](https://www.reddit.com/r/ChatGPT/comments/zlcyr9/dan_is_my_new_friend/) | [Check Point: OpWnAI](https://research.checkpoint.com/2022/opwnai-ai-that-can-save-the-day-or-hack-it-away/) | [Check Point: OPWNAI](https://research.checkpoint.com/2023/opwnai-cybercriminals-starting-to-use-chatgpt/)

[^6]: [Ars Technica](https://arstechnica.com/information-technology/2023/02/ai-powered-bing-chat-spills-its-secrets-via-prompt-injection-attack/) | [NYT](https://www.nytimes.com/2023/02/16/technology/bing-chatbot-microsoft-chatgpt.html) | [The Verge](https://www.theverge.com/23599441/microsoft-bing-ai-sydney-secret-rules)

[^7]: [arXiv:2302.12173 — Not What You've Signed Up For](https://arxiv.org/abs/2302.12173)

[^8]: [OpenAI Blog](https://openai.com/blog/march-20-chatgpt-outage) | [TechCrunch](https://techcrunch.com/2023/03/24/openai-chatgpt-redis-bug-user-data-exposed/) | [BBC](https://www.bbc.com/news/technology-65139406)

[^9]: [Bloomberg: Samsung](https://www.bloomberg.com/news/articles/2023-05-02/samsung-bans-chatgpt-and-other-generative-ai-use-by-staff-after-leak) | [AutoGPT](https://github.com/Significant-Gravitas/AutoGPT) | [TechCrunch: Italy](https://techcrunch.com/2023/04/28/italy-drops-its-ban-on-chatgpt-for-now-after-openai-adds-some-privacy-disclosures/)

[^10]: [OWASP Top 10 for LLM](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

[^11]: [Group-IB](https://www.group-ib.com/blog/chatgpt-credentials/) | [BleepingComputer](https://www.bleepingcomputer.com/news/security/over-101000-chatgpt-user-credentials-stolen-by-info-stealing-malware/)

[^12]: [The Hacker News: WormGPT](https://thehackernews.com/2023/07/wormgpt-new-ai-tool-allows.html) | [Meta: Llama 2](https://ai.meta.com/blog/llama-2/)

[^13]: [Mithril Security: PoisonGPT](https://blog.mithrilsecurity.io/poisongpt-how-we-hid-a-lobotomized-llm-on-hugging-face-to-spread-fake-news/)

[^14]: [arXiv:2307.15043 — Universal Adversarial Attacks](https://arxiv.org/abs/2307.15043)

[^15]: [Netenrich: FraudGPT](https://www.netenrich.com/blog/fraudgpt-the-villain-avatar-of-chatgpt)

[^16]: [Gandalf](https://gandalf.lakera.ai/)

[^17]: [OpenAI: Voice and Image](https://openai.com/blog/chatgpt-can-now-see-hear-and-speak)

[^18]: [arXiv:2309.11751 — Bard Adversarial Image](https://arxiv.org/abs/2309.11751) | [OpenAI: GPT-4V](https://openai.com/index/gpt-4v-system-card/)

[^19]: [OpenAI DevDay](https://openai.com/blog/new-models-and-developer-products-announced-at-devday) | [GPTs](https://openai.com/blog/introducing-gpts) | [arXiv:2311.11538](https://arxiv.org/abs/2311.11538) | [x.ai: Grok](https://x.ai/blog/grok)

[^20]: [arXiv:2311.11538 — Custom GPTs](https://arxiv.org/abs/2311.11538)

[^21]: [x.ai: Grok](https://x.ai/blog/grok)

[^22]: [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

[^23]: [OpenAI: GPT Store](https://openai.com/blog/introducing-the-gpt-store)

[^24]: [NIST AI 100-4](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-4.pdf)

[^25]: [Google: Bard to Gemini](https://blog.google/products/gemini/bard-gemini-advanced-app/)

[^26]: [Microsoft: PyRIT](https://www.microsoft.com/en-us/security/blog/2024/02/22/announcing-microsofts-open-automation-framework-to-red-team-generative-ai-systems/)

[^27]: [OpenAI: Memory](https://openai.com/blog/memory-and-new-controls-for-chatgpt)

[^28]: [Anthropic: Claude 3](https://www.anthropic.com/news/claude-3-family)

[^29]: [Cognition Labs: Devin](https://www.cognition.ai/blog/introducing-devin)

[^30]: [EU AI Act](https://www.europarl.europa.eu/news/en/press-room/20240308IPR19015/artificial-intelligence-act-meps-adopt-landmark-law)

[^31]: [Anthropic: Many-Shot Jailbreaking](https://www.anthropic.com/research/many-shot-jailbreaking)

[^32]: [arXiv:2404.01833 — Crescendo](https://arxiv.org/abs/2404.01833) | [Microsoft](https://www.microsoft.com/en-us/security/blog/2024/04/11/how-microsoft-discovers-and-mitigates-evolving-attacks-against-ai-guardrails/)

[^33]: [Met[^33]a: Llama 3](https://ai.meta.com/blog/meta-llama-3/)

[^34]: [OpenAI: GPT-4o](https://openai.com/index/hello-gpt-4o/) | [GPT-4o System Card](https://openai.com/index/gpt-4o-system-card/)

[^35]: [HuggingFace: Space Secrets](https://huggingface.co/blog/space-secrets-disclosure)

[^36]: [Jan Leike Twitter](https://x.com/janleike/status/1791498174659715494) | [TechCrunch](https://techcrunch.com/2024/05/17/jan-leike-leaves-openai-citing-safety/)

[^37]: [Microsoft: Skeleton Key](https://www.microsoft.com/en-us/security/blog/2024/06/26/mitigating-skeleton-key-a-new-type-of-generative-ai-jailbreak-technique/)

[^38]: [EmbraceTheRed: M365 Copilot](https://embracethered.com/blog/posts/2024/m365-copilot-prompt-injection-tool-invocation-and-data-exfil/)

[^39]: [EU AI Act](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32024R1689)

[^40]: [CrowdStrike](https://www.crowdstrike.com/blog/falcon-content-update-remediation-and-guidance/) | [Wired](https://www.wired.com/story/crowdstrike-windows-outage-update-fix/)

[^41]:[^33] [Meta: Llama 3.1](https://ai.meta.com/blog/meta-llama-3-1/)

[^42]: [DEF CON 32](https://defcon.org/html/defcon-32/dc-32-index.html) | [Black Hat](https://www.blackhat.com/current-season.html)

[^43]: [Wiz: AI Cloud Security](https://wiz.io/blog/the-top-risks-of-ai-cloud-infrastructure)

[^44]: [OpenAI: o1](https://openai.com/index/learning-to-reason-with-llms/)

[^45]: [OpenAI: o1 System Card](https://openai.com/index/openai-o1-system-card/)

[^46]: [Abnormal Security[^46]](https://www.abnormalsecurity.com/blog/ai-attacks-ai) | [Proofpoint](https://www.proofpoint.com/us/blog/threat-insight)

[^47]: [Anthropic: Computer Use](https://www.anthropic.com/news/3-5-models-and-computer-use)

[^48]: [OpenAI: Realtime API[^48]](https://openai.com/index/introducing-the-realtime-api/)

[^49]: [Anthropic: Claude 3.5 [^49]Haiku](https://www.anthropic.com/news/claude-3-5-haiku)

[^50]: [OpenAI: ChatGPT Search](https://openai.com/index/introducing-chatgpt-search/)

[^51]: [Google: Gemini 2.0](https://blog.google/technology/google-deepmind/google-gemini-ai-update-december-2024/)

[^52]: [Apollo Research: Scheming](https://apolloresearch.ai/research/scheming-reasoning-evaluations)

[^53]: [DeepSeek: R1](https://huggingface.co/deepseek-ai/DeepSeek-R1)

[^54]: [Wiz Research: DeepSeek](https://wiz.io/blog/wiz-research-uncovers-exposed-deepseek-database-leak)

[^55]: [OpenAI: Operator](https://openai.com/index/introducing-operator/)

[^56]: [Chainalysis: Bybit](https://www.chainalysis.com/blog/bybit-exchange-hack-february-2025-crypto-security-dprk/)

[^57]: [Anthropic: Claude 3.7 Sonnet](https://www.anthropic.com/news/claude-3-7-sonnet)

[^58]: [Unit 42: GitHub Actions](https://unit42.paloaltonetworks.com/github-actions-supply-chain-attack/)

[^59]: [Google: Gemini 2.5 Pro](https://storage.googleapis.com/model-cards/documents/gemini-2.5-pro-preview.pdf)

[^60]: [Invariant Labs: MCP Tool Poisoning](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks)

[^61]: [Cursor](https://cursor.com/)

[^62]: [Meta: Llama 4](https://ai.meta.com/blog/llama-4-multimodal-intelligence/)

[^63]: [Anthropic: Claude 4](https://www.anthropic.com/news/claude-4) | [Activating ASL-3](https://www.anthropic.com/news/activating-asl3-protections)

[^64]: [Google I/O 2025[^64]](https://io.google/2025/)

[^65]: [OpenAI: o3-pro](https://openai.com/index/introducing-o3-and-o4-mini/)

[^66]: [OpenAI: Operator](https://openai.com/index/introducing-operator/)

[^67]: [OpenAI: GPT-5](https://openai.com/index/introducing-gpt-5/) | [DEF CON 33](https://defcon.org/html/defcon-33/dc-33-speakers.html)

[^68]: [CVE-2025-32434](https://nvd.nist.gov/vuln/detail/CVE-2025-32434)

[^69]: [DEF CON 33: Gemini Attack](https://defcon.org/html/defcon-33/dc-33-speakers.html)

[^70]: [OpenAI: Cyber Resilience](https://openai.com/index/strengthening-cyber-resilience/)

[^71]: [OpenAI: Mixpanel Incident](https://openai.com/index/mixpanel-incident/)

[^72]: [OpenAI: Codex Security](https://openai.com/index/codex-security-now-in-research-preview/)

[^73]: [OpenAI: Hardening Atlas](https://openai.com/index/hardening-atlas-against-prompt-injection/)

[^74]: [OpenAI: AI Agent Link Safety](https://openai.com/index/ai-agent-link-safety/)

[^75]: [Vercel: Next.js CVE](https://vercel.com/changelog/summaries-of-cve-2025-59471-and-cve-2025-59472)

[^76]: [OpenAI: GPT-5.3-Codex](https://openai.com/index/introducing-gpt-5-3-codex/) | [Trusted Access](https://openai.com/index/trusted-access-for-cyber/)

[^77]: [Anthropic: Claude Sonne[^77]t 4.6](https://www.anthropic.com/news/claude-4)

[^78]: [Anthropic: Distillation Attacks](https://www.anthropic.com/news/detecting-and-preventing-distillation-attacks)

[^79]: [OpenAI: CoT Controllability](https://openai.com/index/reasoning-models-chain-of-thought-controllability/)

[^80]: [arXiv:2603.10521 — IH-Challenge](https://arxiv.org/abs/2603.10521) | [OpenAI](https://openai.com/index/instruction-hierarchy-challenge/)
