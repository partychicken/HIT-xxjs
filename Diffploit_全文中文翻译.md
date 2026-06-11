# Diffploit：促进开源库漏洞的跨版本利用迁移

Zirui Chen  
区块链与数据安全全国重点实验室，浙江大学  
中国杭州  
chenzirui@zju.edu.cn

Zhipeng Xue  
区块链与数据安全全国重点实验室，浙江大学  
中国杭州  
zhipengxue@zju.edu.cn

Jiayuan Zhou  
皇后大学  
加拿大金斯顿  
jiayuan.zhou@queensu.ca

Xing Hu*  
区块链与数据安全全国重点实验室，浙江大学  
中国杭州  
xinghu@zju.edu.cn

Xin Xia*  
区块链与数据安全全国重点实验室，浙江大学  
中国杭州  
xin.xia@acm.org

Xiaohu Yang  
区块链与数据安全全国重点实验室，浙江大学  
中国杭州  
yangxh@zju.edu.cn

## 摘要

利用代码（exploit）通常用于证明库漏洞的存在，并验证其在不同版本中的影响。然而，由于软件演化过程中引入的破坏性变更，直接将利用代码应用到其他版本时往往会失败。这些失败既源于触发条件的变化（例如 API 重构），也源于动态环境被破坏（例如构建或运行时错误）；而这两类问题都很难被人工解释并适配。现有技术主要通过模糊测试对代码级执行轨迹进行对齐，但这种方法既耗时，也不足以处理环境层面的失败。此外，在应对跨版本中复杂的触发条件变化时，它们也常常力不从心。为解决这一问题，我们提出 Diffploit，一种迭代式、由差异驱动的利用迁移方法，其围绕两个关键模块构建：上下文模块（Context Module）和迁移模块（Migration Module）。上下文模块通过分析目标版本与参考版本之间的行为差异来构造上下文，这些上下文捕获失败症状及其相关的 diff 代码块。借助这些上下文，迁移模块通过一个基于大语言模型（LLM）的迭代反馈循环引导适配过程，在 diff 候选的探索与逐步细化之间取得平衡，从而有效解决复现失败。我们在一个大规模数据集上评估了 Diffploit，该数据集包含 79 个库中的 102 个 Java CVE 和 689 个版本迁移任务。Diffploit 成功迁移了 84.2% 的利用代码，较感知变更的测试修复工具 TaRGET 提高了 52.0%，较 IDEA 中基于规则的工具提高了 61.6%。除技术效果外，Diffploit 还识别出 5 份受影响版本范围存在错误的 CVE 报告，其中 3 份已被确认。我们还在 GitHub Advisory Database 中发现了 111 个未报告版本。

## CCS 概念

- 安全与隐私 → 软件安全工程。

## 关键词

库漏洞，利用迁移，受影响版本

\* 通讯作者

# 1 引言

开源库是现代软件开发中的关键基础设施，它使开发者能够避免重复实现，并加速开发过程 [32, 51, 57, 65, 71]。然而，开源库的广泛采用也引发了人们对这些库中漏洞所带来风险的担忧 [10, 23, 37, 42, 43, 45, 64, 68–70]，因为这些漏洞可能会从上游库传播到下游项目 [5, 33, 35]。考虑到库通常演化迅速，而下游项目又可能依赖广泛的版本范围，因此有必要评估漏洞在不同版本中的影响 [4, 49]。为此，开发者常常使用公开披露的利用代码来评估漏洞行为是否以及如何在不同版本中表现出来 [66]，例如识别受影响的库版本 [13, 29]，以及评估在具有不同受影响版本的下游项目中的可利用性 [10, 17, 20, 31, 70]。

然而，若不做适配，简单地将原始利用代码复用到其他受影响版本上，往往无法成功复现 [13, 66]，因为公开披露的利用代码通常是针对最初报告该漏洞的版本专门编写的，无法泛化到其他受影响版本。这些失败通常源于：(1) 动态环境被破坏 [26, 61, 66]；以及 (2) 底层触发条件发生变化 [67]。人工理解并解决这些问题通常既耗时又需要大量专业知识，这凸显了自动化利用迁移的必要性。现有研究已经证明了跨版本迁移利用代码的可行性 [13, 29]，其通常通过 API 匹配来对齐执行轨迹。尽管这些方法在处理轻微 API 变化方面取得了一定成功，但它们往往依赖模糊测试，因此在需要迁移多个版本时会十分耗时。更重要的是，它们忽视了实践中经常出现的两个关键挑战：

❶ **动态环境被破坏（Broken Dynamic Environment）**：以往方法主要处理代码级变化，却忽略了依赖升级或运行时配置等环境变化所引入的不兼容性。例如，CVE-2020-5245 的利用代码 [55] 在 1.3.8 版本中会因所依赖库（javassist）更新而触发运行时错误，尽管利用代码本身完全相同。这类由环境引起的失败可能出现在多个阶段，包括构建过程和运行时 [63]，并且在不同库之间具有很强的多样性，从而给缓解带来显著挑战。

*arXiv:2511.12950v1 [cs.SE]，2025 年 11 月 17 日*

❷ **复杂的触发条件演化（Complicated Triggering Condition Evolution）**：当 API 接口在不同版本之间发生较大变化、重构 [56] 甚至被删除时，现有的轨迹对齐方法——例如识别函数重命名或函数合并/拆分 [13]——往往难以找到合适的替代项。一个代表性例子是图 1 所示的 CVE-2024-22257 [16]：关键方法（createList 和 createAuthorityList）在 2.0.8.RELEASE 中并不存在，因此需要使用签名和语义都显著不同的替代 API，而这无法通过现有 API 匹配策略有效解决。在这类场景中，不正确或不充分的匹配会直接导致迁移无效。

应对这些挑战需要一种更全面的方法，不仅要考虑环境调整，还要具备智能 API 适配机制，而不能仅依赖简单的轨迹或命名对齐。为有效解决跨版本利用迁移问题，我们提出了一个迭代式、由 diff 驱动的 LLM（大语言模型）框架 Diffploit。其核心思想是：利用带有版本感知能力的上下文和反馈，迭代地适配失败的利用代码。当某次迁移尝试失败时，Diffploit 会比较目标版本与一个成功参考版本的行为，以识别失败指示器（failure indicators）。这些指示器会触发上下文模块构造结构化的迁移上下文：既包括很可能导致失败的致因差异（Causing Diffs），也包括可能帮助解决失败的支持性差异（Supporting Diffs）。该上下文随后用于指导迁移模块对利用代码进行适配。我们设计了一种模拟退火策略，在多轮迭代中探索并细化适配方案。每次尝试后，系统都会重新执行更新后的利用代码，并收集新的反馈以指导下一轮循环。通过这一闭环过程，Diffploit 能够逐步缩小差异，并实现有效迁移，即使面对显著的 API 或环境变化也是如此。

我们在一个大规模 Java 开源库漏洞利用数据集 [57] 上评估了 Diffploit。该数据集包含 79 个库中的 102 个 CVE 及其明确受影响版本。其中，我们识别出 30 个需要跨 689 个版本进行迁移的利用代码，这一规模与先前 C/C++ 领域研究 [13] 可比，后者涉及 30 个 CVE 和 470 个版本。具体而言，Diffploit 成功将 23 个 CVE 中的 580 个利用代码迁移到目标版本，在 689 个任务中达到了 84.2% 的成功率。与近期面向变更感知的测试修复研究 [48] 相比，Diffploit 相比 TaRGET 提升了 52.0%，并较 IDEA 中基于规则的方法高出 61.6%，凸显了 Diffploit 利用 LLM 能力进行 exploit 迁移的有效性。我们还通过消融实验验证了 Diffploit 设计的合理性：相较于基础模型，它带来了 46.1% 的提升。为了验证 Diffploit 所迁移利用代码的实用性与正确性，我们基于迁移后的利用代码，在 5 个 CVE 漏洞中识别出了先前未披露的易受攻击版本，并联系了 CNA（CVE 编号机构，CVE Numbering Authority）[11] 进行确认。其中，3 个案例已被确认，我们提交的利用链接也被纳入参考链接。同时，我们向 GitHub Advisory Database 提交了 6 个拉取请求，以纳入 Diffploit 检测出的共 111 个缺失版本，其中 82 个版本已被接受。我们进一步证明，Diffploit 能以较低成本有效克服前述挑战。

本文的主要贡献如下：

- 我们提出了 Diffploit，这是一种新颖的、由差异驱动的方法，能够自动将利用代码从目标版本迁移到参考版本。Diffploit 的源代码和数据集已在我们的网站上公开 [9]。
- 我们在 689 个版本上进行了评估，结果表明 Diffploit 成功为其中 580 个版本迁移了利用代码，性能优于我们的基线方法。
- 我们迁移得到的利用代码揭示了 5 个受影响版本范围不准确的 CVE（其中 3 个已获 CNA 确认），并发现 GitHub Advisory Database 漏报了 111 个受影响版本。

# 2 动机

本节介绍使用场景和一个动机示例，以说明 Diffploit 所要解决的挑战。

## 2.1 使用场景

在现实世界的漏洞管理中，我们的方法旨在支持两个实际使用场景：

1. **将无效利用代码适配到已确认的受影响版本。** 公开披露的利用代码对于验证某个特定项目是否受到已知上游漏洞影响至关重要。然而，这些利用代码通常是为某个特定漏洞库版本编写的，因此由于代码或环境变化，它们往往无法在其他受影响版本上运行。例如，假设某份安全公告确认库 netty-codec-http 的 4.0.56 版本受到 CVE-2021-43797 影响，但公开利用代码只能在高于 4.1.0.CR7 的版本上工作 [55]。使用 4.0.56 的开发者便无法获得一个可用的利用代码来验证其自身场景中的漏洞。Diffploit 通过自动将已有利用代码适配到这些已确认但与原 exploit 不兼容的版本，填补了这一空白，从而使现实环境中的实际验证成为可能。

2. **发现先前未报告的受影响版本。** 人工整理的漏洞报告往往包含错误和遗漏 [7, 12, 18, 30, 41]。当迁移后的利用代码能够在某个未被公共安全公告列出的版本上成功触发漏洞时，这表明受影响范围可能比已披露的更广。因此，Diffploit 可以帮助识别缺失的受影响版本，并提升漏洞数据库的完整性。但值得注意的是，如果 Diffploit 未能将某个利用代码迁移到特定版本，这并不一定意味着该版本不受影响，仍然需要进一步分析 [25]，例如引入提交分析（commit analysis）[2, 8, 60]。

## 2.2 动机示例

图 1 展示了一个来自 spring-security-core 的动机示例，用于说明 Diffploit 如何促进跨版本利用迁移。通过这一过程，我们识别出 5.7.0.RELEASE 之前的版本也受到 CVE-2024-22257 影响，尽管这些版本并未被纳入 CVE 报告。对于高于 3.0.0.RELEASE 的版本，已有一个公开披露的 CVE-2024-22257 利用代码可以用于复现该漏洞，它作为参考利用代码。该参考利用代码无法直接在更早的版本上执行，例如 2.0.8.RELEASE，因为它依赖于后续版本才引入的 API（如第 16 行、第 23 行），并且还受到更新过程中引入的破坏性变更影响（例如将现有 API 重构到不同 package 中）[52]，因此需要进行利用迁移。

近年来，人工智能，尤其是 LLM，在多种任务中展现出强大能力 [21, 34, 44, 48, 54, 62, 72]，例如理解复杂文档 [72] 和解决错误 [21, 54]。然而，如果缺乏足够的上下文信息，LLM 往往难以使用目标版本中的恰当 API 来更新利用代码。此外，在更早版本中，由于训练数据中缺乏相应的使用模式 [50]，LLM 经常无法生成有效的 API 调用。这些局限削弱了 LLM 为目标版本生成可靠利用代码的能力。我们观察到，如果向 LLM 提供目标版本与参考版本之间的 diff，LLM 便能够推断失败的根本原因，并识别适配策略，例如定位替代 API。

基于这一观察，我们识别出两类有助于利用迁移的 diff：致因差异（Causing Diff）和支持性差异（Supporting Diff）。无论是由于触发条件变化还是环境破坏导致的 exploit 复现失败，本质上都源于库演化过程中引入的变更（causing diff）。与此同时，库自身通常也会包含针对这些变化的内部适配，而这些适配可以为 LLM 的迁移提供指导（supporting diff）。以 CVE-2024-22257 为例：

- **致因差异（Causing Diff）**：第 5 行到第 11 行中出现的 `Package does not exist` 错误，其根源在于类文件的重构或修改。具体来说，对应的 diff 代码块是那些涉及目标 package 名称变更的部分，例如 `ConfigAttribute` 类中 `package org.springframework.security` 这一 package 声明行被删除。这些变化为 LLM 提供了目标版本准确的 package 结构信息，从而减少生成过时或不存在 import 路径之类的幻觉。
- **支持性差异（Supporting Diff）**：对于第 16 行的修改，`UnanimousBasedTests` 中的一处变更反映了对 `ConfigAttributeDefinition` 重构的响应，并为解决 `createList` 方法的 `Cannot find symbol` 错误提供了修改指引。此外，`AuthorityUtils` 中的 diff 还明确指导如何将第 23 行不可用的 `createAuthorityList` 方法替换为目标版本中引入的替代 API。这些 diff 能帮助 LLM 应对触发条件的演化，通过识别已删除 API 的合适替代项来完成迁移。

基于所识别出的 diff，我们提出了 Diffploit，它通过一个反馈循环迭代地利用这些 diff 来促进 exploit 迁移。尽管在两个版本之间存在 2,555 处 diff、从中识别相关信息极具挑战，Diffploit 仍然只需九个步骤就完成了对 CVE-2024-22257 利用代码的迁移。

![图 1：Diffploit 将 CVE-2024-22257 的利用代码从版本 3.0.0.RELEASE 迁移到 2.0.8.RELEASE。](#)

**参考利用代码（3.0.0.RELEASE）**

**迁移后的利用代码（2.0.8.RELEASE）**

**[失败指示器]**
- Package does not exist

**[相关 Diff]**
- {Classname}.java

**3.0.0.RELEASE**

**2.0.8.RELEASE**

**支持性 Diff**

`{Classname}.java:`
- `package org.springframework.security.vote;`
- `…`
- `package org.springframework.security;`
- `…`
- `package org.springframework.security.util;`
- `…`

`UnanimousBasedTests.java:`
- `ConfigAttributeDefinition config = new ConfigAttributeDefinition(new String[]{"ROLE_1", "DENY_FOR_SURE"});`
+ `List<ConfigAttribute> config = SecurityConfig.createList(new String[]{"ROLE_1", "DENY_FOR_SURE"});`

`AuthorityUtils.java:`
- `public static GrantedAuthority[] commaSeparatedStringToAuthorityArray(String authorityString) {`
- `…`
- `return authorities;`
- `}`

**[失败指示器]**
- Cannot find symbol

**[相关 Diff]**
- UnanimousBasedTests.java
- AuthorityUtils.java

**CVE-2024-22257**

**库：**  
spring-security-core

**需要迁移：**  
2.0.0 - 2.0.8.RELEASE

**NVD：**  
5.7.0 to 5.7.12  
5.8.0 to 5.8.11  
6.0.0 to 6.0.9  
6.1.0 to 6.1.8  
6.2.0 to 6.2.3

参考版本  
已报告  
遗漏！  
目标版本  
致因差异  
支持性差异

# 3 提出的方法

本节介绍我们的方法 Diffploit，包括整体工作流程以及其两个核心组成部分——上下文模块（Context Module）和迁移模块（Migration Module）的设计。

## 3.1 概览

给定一个需要进行 exploit 迁移的目标版本，我们的方法通过自适应过程迭代修复其与参考 exploit 之间的行为差异。参考版本从已成功复现的版本集合中选取，用于提供上下文指导。整体过程遵循一个由差异驱动的循环。在每次迭代中，我们比较目标版本和参考版本的执行输出，以识别复现失败。目标版本中检测到的每个失败指示器，都会触发上下文模块构建一个专门的迁移上下文。

该上下文封装了失败症状、与之对应的诊断键（diagnostic key），以及可能构成失败根源或有助于解决失败的代码级差异。

对于每个检测到的失败，相应的迁移上下文都会传递给迁移模块，由后者结合该上下文尝试对 exploit 进行适配。迁移过程由一种模拟退火策略引导，在上下文中的 diff 上进行探索。

## 3.2 上下文模块

上下文模块通过分析目标版本 $v_t$ 与参考版本 $v_r$ 之间的行为差异，从目标版本 $v_t$ 中提取一组迁移上下文。每个迁移上下文对应一个特定的失败单元，表示复现失败的一个原因（触发条件变化或动态环境失效），并包含可能导致或解决该失败的相关 diff。这些迁移上下文提供了结构化且有针对性的信息，以指导后续迁移过程。

**定义 3.1（迁移上下文）.** 迁移上下文是一种结构化表示，它将 $v_t$ 中观察到的复现失败指示器 $f_i$ 与其对应的失败标识符 $k_i$ 以及相关 diff 块关联起来。形式化地定义为：

$$
C_i = (f_i, k_i, D^{(i)}_{\text{cause}}, D^{(i)}_{\text{support}})
$$

其中：

- $D^{(i)}_{\text{cause}}$ 是与 $f_i$ 潜在根因相关的 diff 块；
- $D^{(i)}_{\text{support}}$ 是为迁移提供额外帮助的 diff 块，例如被修改的测试用例或替代函数。

所有 diff 块均指由 `git diff` 生成的 Git 风格 diff hunk。

### 3.2.1 失败指示器提取

对于每个需要进行 exploit 迁移的 $v_t$，我们确定一个对应的 $v_r$ 作为上下文指导。参考版本从成功复现的版本列表 $R = \{v_1, v_2, ..., v_n\}$ 中选取，使得

$$
v_r = \arg\min_{v \in R} \mathrm{dist}(v, v_t)
$$

其中，$\mathrm{dist}(\cdot)$ 表示版本距离度量。列表 $R$ 是通过在 Maven Central 仓库中的所有历史版本上执行已披露的 exploit 构建得到的。

在识别出 $v_r$ 后，我们分别在 $v_r$ 和 $v_t$ 上执行参考 exploit，并收集各自的输出。随后，我们比较这两个输出以检测行为差异。每个差异都被抽象为一个失败指示器 $f_i$，表示 $v_t$ 中失败的一个具体症状。失败指示器涵盖多种失败类型，包括构建时错误、运行时异常和断言失败。提取出的指示器集合 $\{f_1, f_2, ..., f_n\}$ 作为构建迁移上下文的切入点。

### 3.2.2 失败标识符提取

给定一个失败指示器 $f_i$，该模块的目标是推导出一组对应的失败标识符 $k_i$，以简洁地捕捉失败的语义核心，从而支持下游的 diff 检索。某些指示器（如构建时错误）具有结构化格式，可以用简单规则解析；而另一些指示器（如运行时异常或非预期执行结果）在语法和语义上变化更大，因此难以使用基于模式的启发式方法处理。

为解决这一问题，对于每个 $f_i$，我们提示 LLM 生成一组对应的标识符 $k_i$，其中 $k_i$ 中的每个元素都是一个显著 token，用于以便于与 diff 内容匹配的方式反映失败的核心内容。这些标识符 $k_i$ 通常包括方法名、异常类型或可能出现在相关 diff 中的符号化 token，后续将用于检索成因 diff 和支持 diff。

### 3.2.3 成因 Diff 提取

该模块的目标是识别可能构成给定失败 $f_i$ 根因的 diff 块。我们利用一组人工设计的启发式规则，这些规则针对常见失败类型进行了定制，以提取相关 diff。这些规则使我们能够高效地将符号化错误轨迹与相应代码修改关联起来，而无需依赖复杂的程序分析。

给定失败指示器 $f_i$ 及其对应的标识符集合 $k_i$，我们定义映射函数：

$$
D^{(i)}_{\text{cause}} = \mathrm{Rule}(\mathrm{type}(f_i), k_i)
$$

其中，$\mathrm{type}(f_i)$ 表示错误的高层类别（例如，`AssertionMismatch`、`MissingMethod`、`RuntimeError`）。我们实现了以下规则，这些规则不仅用于处理触发条件变化和环境层面的失效，也适用于迁移过程中遇到的其他非预期场景：

- **AssertionMismatch**：收集那些影响 $k_i$ 中方法调用的 diff 块，特别是 exploit 中直接调用且位于导致断言失败的调用栈上的方法。
- **RuntimeError**：识别那些修改了与异常类型或 $k_i$ 中消息 token 相关的方法、类或环境的 diff 块。这些符号通常反映了 API 使用失败或动态执行中的行为变化。
- **MissingClass / MissingPackage**：识别位于与 $k_i$ 中缺失类或包相对应文件下的 diff。
- **MissingMethod**：定位那些修改或删除了 $k_i$ 中引用的方法签名的 diff 块。
- **IncompatibleType / WrongReturn**：检索那些影响由 $k_i$ 标识的方法附近的方法返回类型、参数类型或泛型用法的 diff 块。
- **Other**：对于所有其他失败类型，我们检索那些修改了名称与 $k_i$ 中任一标识符匹配的元素（如类、方法或字段）的 diff 块。该回退规则确保在没有更具体启发式方法时仍具有通用适用性。

我们应用一组规则来判断某个 diff hunk 是否满足这些条件（例如，模式 `\b\w[\w\s<>,]*\s+{cause}\s*\(` 用于检测 `MissingMethod` 情况）。随后，每个由这些启发式方法匹配到的 diff hunk 都会被赋予一个相关性分数（前五类为 10 分，`Other` 中每个 $k_i$ 为 2 分），以指导后续迁移过程。

### 3.2.4 支持 Diff 提取

该模块旨在识别库针对导致 $f_i$ 的破坏性变更所引入的 diff 块，这些 diff 块可以辅助 exploit 迁移过程。我们观察到，诸如 API 删除或重构这类会导致 exploit 失败的破坏性变更，也会影响库内部对 API 的使用。为了维持功能与兼容性，库通常会引入相应的代码更新，包括修改内部方法调用、增加包装函数，或调整测试用例。这些响应式变更为指导迁移提供了有价值的支持。

此类响应往往表现为新添加的 diff 块，并且在结构上与同一 hunk 中被删除或修改的代码相似。为了检测这种模式，我们首先在 diff 中定位失败标识符集合 $k_i$ 中 token 的出现位置。然后，在每个出现位置周围，我们提取由具有相同 diff 前缀的行组成的连续 diff 块。具体而言，我们收集：

- $B^+$：在 $k_i$ 中 token 出现位置附近，由连续新增行（以前缀 `+` 标记）组成的块；
- $B^-$：在 $k_i$ 中 token 出现位置附近，由连续删除行（以前缀 `-` 标记）组成的块；

我们使用最长公共子序列（LCS）长度的归一化值，计算每个新增块 $b^+ \in B^+$ 与对应删除块 $b^- \in B^-$ 之间的 token 级相似度：

$$
\mathrm{sim}(b^+, b^-) = \frac{2 \times |\mathrm{LCS}(b^+, b^-)|}{|b^+| + |b^-|}
$$

其中，$|\mathrm{LCS}(\cdot)|$ 表示 LCS 中匹配 token 的数量，$|\cdot|$ 表示一个块的 token 数量。如果相似度超过阈值 $\tau$，则包含该块对的 diff hunk 会被视为候选 hunk，并被选入支持 diff 集 $D^{(i)}_{\text{support}}$。

我们根据这种相似度为每个选中的 hunk 赋分（范围为 0–10），并将其用于指导后续的 diff 退火过程。

最后，通过将失败指示器及其标识符 $k_i$ 与提取出的成因 diff $D^{(i)}_{\text{cause}}$ 和支持 diff $D^{(i)}_{\text{support}}$ 相关联，生成针对失败 $f_i$ 的迁移上下文 $C_i$，形成一种用于指导迁移过程的结构化表示。

## 3.3 迁移模块

为了支持有效迁移，上下文模块提供了从迁移上下文 $C_i$ 推导出的候选 diff hunk。这些候选项构成了迁移模块的搜索空间，并在引导成功的 exploit 迁移中发挥核心作用。在此基础上，迁移模块通过我们的退火策略选择 diff，并将其与 $f_i$ 一同呈现给 LLM；LLM 隐式决定 exploit 中合适的迁移位置。

### 3.3.1 Diff 退火

为了扩展用于迁移的候选 diff 的多样性与复杂性，我们设计了一种基于退火的机制，以受控随机性探索更广泛的 diff 空间。这里的复杂性不仅指针对给定失败指示器 $f_i$ 同时应用的 diff 数量，也指在 $f_i$ 的迁移过程中当出现新的失败指示器时所允许的修复轮数。这样的设置使系统能够同时考虑更广的补丁组合与更深的修复动作链，从而丰富潜在解空间。

该机制作用于三类 diff：高分成因 diff $D_{\text{cause}}$、支持 diff $D_{\text{support}}$，以及新合成的复合 diff 集 $D_{\text{combo}}$。我们首先通过对 $D_{\text{cause}} \cup D_{\text{support}}$ 中排名靠前的 diff 进行两两组合来构建 $D_{\text{combo}}$。对于每个组合，新 diff 继承一个复合分数，该分数由其组成 diff 的分数平均值得到。于是，$D_{\text{cause}} \cup D_{\text{support}} \cup D_{\text{combo}}$ 共同定义了该退火过程的搜索空间 $D$。

为了启动退火，我们定义温度参数 $T$，它同时控制 diff 的选择概率以及解决当前失败指示器 $f_i$ 时的探索深度。diff $d \in D$ 的选择概率与其归一化分数和温度成正比：

$$
P(d) \propto \exp\left(\frac{s(d)}{T}\right),
$$

其中，$s(d)$ 表示 diff $d$ 的分数。较高的温度会使选择偏向于高分 diff。

此外，探索深度——即在迁移过程中出现新的失败指示器时所进行的重试次数——与温度成反相关。在较高温度下，系统倾向于快速评估有前景的 diff，而将更深入的探索推迟到温度降低后的后续阶段。这种设计确保系统在早期阶段优先高效评估高分 diff。

在每次迭代中：

- 采样一个 diff $d \in D$，并通过 LLM 执行一次以 $f_i$ 为目标的迁移。
- 如果迁移失败，则对 $d$ 的分数进行惩罚，降低全局温度 $T$，并基于更新后的分数刷新 diff 组合列表 $D_{\text{combo}}$。
- 然后选择一个新的 diff，重复该过程。

这种基于退火的策略能够对 diff 空间进行受控探索，在利用高分 diff 与探索更多样或更复杂的 diff 组合之间取得平衡。

### 3.3.2 Exploit 迁移

给定从目标版本执行输出中提取的失败指示器 $f_i$，Exploit 迁移模块的目标是调整原始 exploit，使预期的漏洞行为得以恢复。该过程以一个选定的相关 diff $d_i$ 为输入，该 diff 基于先前分析被认为有望解决该失败。

我们首先利用 LLM，在失败指示器 $f_i$、其关联的诊断键 $k_i$ 和选定 diff $d_i$ 的条件下，定位 exploit 中相关的修改位置。这些元素被编码进一个结构化提示中，如图 2 所示，引导 LLM 精确指出修改位置。

基于这一定位，Diffploit 随后指示 LLM 在所识别位置上，根据 $d_i$ 所反映的变更修改原始 exploit，生成一个适配后的 exploit $\hat{e}_i$。然后在目标版本上执行 $\hat{e}_i$。通过分析其运行时输出，判断 $f_i$ 是否已被解决。

### 3.3.3 迁移验证

在前一模块中针对特定失败指示器 $f_i$ 完成 exploit 适配后，Diffploit 会重新运行 exploit，以收集更新后的失败指示器，从而判断 $f_i$ 是否已经解决。

- 如果 $f_i$ 仍然出现在失败列表中，则认为该次迁移尝试不成功。随后，过程返回退火流程，选择下一个 diff 候选项。这种迭代搜索将持续进行，直到 $f_i$ 被解决，或者搜索因达到温度上限或超时而终止。
- 如果 $f_i$ 不再出现在失败列表中，表明已成功解决，我们会进一步分析更新后的失败指示器：
  - 如果不再存在任何失败指示器，则迁移过程进入最终复现验证，通过检查 exploit 中的断言是否按预期表现，以及观察到的行为是否与预期复现行为一致来完成验证。如果这两个条件都满足，则迁移过程成功终止。
  - 如果剩余的失败指示器全部为先前已知的失败指示器（即在处理 $f_i$ 之前就已识别），则我们认为 $f_i$ 已被成功解决，并继续处理其余失败。
  - 如果在尝试解决 $f_i$ 后出现了新的失败指示器，我们将启动有限次数的额外修复尝试，以处理这些新出现的失败。此类尝试的次数称为探索深度，它依据当前温度 $T$ 确定，较高的退火温度对应较浅的探索。如果修复尝试次数超过由 $T$ 导出的探索预算，则当前迁移路径终止，并选择新的 diff 候选项来解决 $f_i$。

# 4 实验设置

**研究问题。** 我们的实验评估旨在回答以下研究问题：

- **RQ1（有效性）**：Diffploit 在多大程度上能够有效地将 exploit 迁移到目标版本？
- **RQ2（消融研究）**：Diffploit 内部各个组件如何对整体迁移过程作出贡献？
- **RQ3（实际可行性）**：考虑到 exploit 的成本和质量，Diffploit 在真实场景中是否可被接受？

我们通过 RQ1 评估基于 diff 的 exploit 迁移方法的有效性及其相较现有方法的优越性。我们通过 RQ2 评估方法中关键组件的合理性，包括成因 diff 提取模块、支持 diff 提取模块以及我们提出的 diff 退火算法。我们通过 RQ3 验证 Diffploit 生成的 exploit 以及其识别出的报告不足的易受攻击版本是否能够被认可和接受。

## 4.1 数据集

为了尽量减少 exploit 收集过程中的偏差，我们使用目前公开可获得的最大 Java exploit 数据集 [57] 来评估 Diffploit 的性能。该数据集包含 102 个 CVE 漏洞，每个漏洞都对应一个针对特定版本的 exploit，以及一组经人工验证的受影响版本，因此非常适合用于评估 exploit 迁移方法的有效性。

为了识别需要进行 exploit 迁移的版本，我们在被标记为受影响的版本上执行 exploits，并检查执行结果。该数据集包含用于验证复现的断言，因此，未能满足断言的版本会被标记为需要迁移。我们共识别出 988 个满足这一条件的版本。

随后，我们进一步分析这些版本，以确认其中确实存在漏洞，最终得到 30 个漏洞对应的 689 个真实受影响版本。数量减少主要归因于：在 CVE-2023-51080 中有 176 个版本，以及在 CVE-2021-43795 中有 72 个版本，在漏洞被引入之前就被错误地标记为易受攻击版本 [38, 53]。

## 4.2 基线方法

据我们所知，此前尚无工作探索跨 Java 库版本的 exploit 迁移，尽管第三方库在 Java 生态系统中发挥着关键作用 [22]。现有研究 [13, 29] 依赖 AFLGo [6] 等模糊测试框架，而这些框架难以适配 Java。我们纳入以下四个基线：

1. **TaRGET [48]** 是一种基于预训练语言模型的自动化函数级测试修复方法，它将测试修复视为一种语言翻译任务，并利用从测试失效中提取的上下文信息。尽管它并非专为 exploit 迁移而设计，但由于其在修复损坏的 JUnit 测试用例方面表现强劲，而这类用例在结构上与我们数据集中的 exploits 相似，因此我们将 TaRGET 作为基线。
2. **IDEA [27]** 是 IntelliJ IDEA 提供的 Quick Fix 与 Auto-import 功能的组合。这些功能帮助开发者解决由 API 重构、缺失导入或过时方法签名引起的编译问题，因此可以部分缓解由触发条件变化导致的 exploit 失败。
3. **GPT-4o** 和
4. **DeepSeek-v3** 是两种最先进的专有 LLM。

## 4.3 迁移成功判定标准

为确保迁移后的 exploit 触发与原始版本相同的漏洞，我们从两个维度进行验证：断言一致性和行为验证。断言一致性是指，迁移后的 exploit 是否表现出与原始 exploit 相同的断言失败行为，这表明其在断言层面保持一致。行为验证则通过检查输出日志中的细粒度行为指标，判断预期的漏洞行为在迁移后是否得以保留。

当且仅当某个 exploit 在目标版本中触发了预期断言，且该断言的位置和表现形式与参考版本一致时，我们认为该 exploit 已成功迁移到目标版本。

## 4.4 实现

在实验设置中，我们部署了一个基于 Ubuntu 20.04 的 Docker 环境。按照 Wu 等人 [57] 在 exploit 收集中的配置，我们选择 Java 11 作为运行时环境，具体版本为 18.9（version: build 11+28）。我们使用 `mvn test` 命令执行 exploit。在 LLM 的选择上，我们采用了一个高性能闭源模型 GPT-4o（截至 2024-11-20 的快照），以及一个在代码任务上表现强劲的开源模型 DeepSeek-v3（快照 0324）开展实验。为提高可复现性，Diffploit 的基础模型为 DeepSeek-v3。实验中，我们对 Diffploit 和各基线方法在每个版本上都施加 5 分钟的时间限制。由于数据集中的所有库都托管在 GitHub/GitLab 上，我们使用 `git diff` 生成 diff 文件。

关于基线设置，我们使用复现实验包中提供的微调权重复现 TaRGET。尽管我们已尽最大努力，但仍无法在 CVE-2020-13956 和 CVE-2023-51075 上复现 TaRGET，因为它需要为这些漏洞构造有效的 exploit。对于涉及 IntelliJ IDEA 的实验，我们使用 2025.1.3 版本。我们应用 IntelliJ IDEA 的 Quick Fix 和 Auto-import 功能来修改 exploit，并在无法继续修改后使用 `mvn test` 运行测试。当可选建议很多时，包括存在多个重命名建议的情况，我们评估前五个候选项。

# 5 实验评估

我们从三个方面评估 Diffploit 的性能。首先，我们使用目前规模最大的 Java 库漏洞 exploit 数据集评估其有效性，并与基线方法进行比较，同时分析其优势与局限。其次，我们进行消融实验，以展示 diff 模块和迁移模块对整体性能的贡献。最后，我们基于成本以及 CNA 和开源维护者的反馈，分析 Diffploit 的实际可行性。

## 5.1 有效性

### 5.1.1 性能

我们在一个包含 689 个需要进行 exploit 迁移的版本对的数据集上评估 Diffploit 的有效性。Diffploit 成功迁移其中 580 个，整体成功率达到 84.2%。在 30 个代表性 CVE 的范围内，Diffploit 成功完成了其中 23 个案例的 exploit 迁移，覆盖了多种漏洞类型和受影响库。这表明 Diffploit 在漏洞类别和依赖生态两个层面都具有良好的通用性。值得注意的是，对于其中 20 个 CVE，所有需要迁移的相关版本都被成功修复，这凸显了 Diffploit 作为自动化漏洞验证流水线中可靠组件的潜力。然而，需要强调的是，迁移失败并不一定意味着漏洞不存在。

与基线方法 TaRGET 相比，Diffploit 取得了 51.0% 的相对提升，表明其在处理 exploit 迁移中的关键挑战时具有稳健性，例如适配非函数级编辑（如 import 调整和构建配置更新）。与结合 IntelliJ IDEA 的 Quick Fix 和 Auto Import 功能的 IDEA 相比，Diffploit 显著领先 61.6%。这一结果凸显了其处理超出预定义规则能力范围的迁移场景的能力。我们还将 Diffploit 与直接使用 LLM 的方式进行了比较。GPT-4o 和 DeepSeek-V3 分别能够迁移 243（35.2%）和 263（38.1%）个案例；它们在一些较为直接的场景中表现尚可，但由于缺乏迁移上下文而整体逊色。相比之下，Diffploit 利用面向迁移的上下文信息，因此能够维持更高的成功率，尤其是在复杂迁移场景中。

### 5.1.2 优势

与基线方法 TaRGET 相比，Diffploit 在处理不局限于函数级别的测试变更时展现出更强的适应性。TaRGET 侧重于识别有缺陷的函数，但当变更发生在函数体之外时，例如修改 import 语句，或修复 `pom.xml` 中损坏的运行时环境，它依赖开发者手动应用合法编辑。因此，在 16 个必须进行此类非功能性变更的 CVE 中，它无法修复测试用例。

Diffploit 还能够处理这样一种场景：某个 exploit 测试本应失败，却在目标版本中悄然通过。这类情况中，漏洞的存在被表面上成功的测试所掩盖，而 TaRGET 往往会忽略它们。通过利用丰富的迁移上下文和历史 diff，Diffploit 能够检测并适配这类具有误导性的测试用例，确保它们继续作为漏洞的有效指示器。

此外，Diffploit 利用一种结构化的迁移上下文，同时捕获测试用例的致因 diff 和支持性 diff。这一设计使其能够适应更广范围的、与 exploit 迁移相关的代码修改。在复杂 exploit 迁移任务中，这一优势更加突出，因为此类任务需要理解细微的语义变化。在 Discussion 一节中，我们进一步分析迁移前后的编辑距离，突出 Diffploit 应对复杂适配的能力。

### 5.1.3 局限性

尽管 Diffploit 在跨版本迁移 exploit 方面表现强劲，但它仍存在若干局限。首先，它难以处理漏洞在不同版本中呈现方式不同的情况。例如，在 CVE-2020-13956 和 CVE-2023-51075 中，exploit 触发的是 `NumberFormatException` 和 `IndexOutOfBoundsException` 等异常，而不是最初预期的断言失败。尽管这类运行时异常仍然表明存在与安全相关的行为，Diffploit 目前仍将其视为失败，因为它采用固定的、基于断言的验证策略。类似地，在 CVE-2023-49250 中，exploit 会导致与恶意服务器建立连接。

**图 3：Diffploit 在 CVE-2022-34662 上的一个失败案例。**

```java
ResourcesServiceImpl res = new ResourcesServiceImpl();
Method method_verifyFile = res.getClass().getDeclaredMethod("verifyFile", …
```

**参考版本**  
`v2.0.0-alpha`

```java
ResourcesService res = new ResourcesService();
Method method_verifyFile = res.getClass().getDeclaredMethod("updateResource", …
```

**有效版本**  
`v1.3.9`

- 迁移失败
- 迁移成功
- Diffploit

其次，Diffploit 在迁移依赖于版本特定 API 的 exploit 时会遇到困难，尤其是在目标版本既缺乏结构上相似的对应项，也缺少支持性信息时。如图 3 所示，在 CVE-2022-34662 中，较高版本中的原始 exploit 调用了 `ResourcesServiceImpl.verifyFile`，而较低版本中在另一个类 `ResourcesService` 下提供了一个语义相关的方法 `updateResource`。由于缺乏上下文对齐，Diffploit 未能完成该方法的迁移。这个例子表明，尽管 Diffploit 能在一定程度上适配此类情况，但仍需要外部 API 知识。

## 5.2 消融实验

我们的消融实验旨在达到两个目标：（1）证明我们设计中的每个组件都有助于提高 exploit 复现成功率；（2）说明我们的设计有助于降低以步骤数衡量的复现成本。我们构造了 Diffploit 的三个消融变体来评估各组件的贡献：(a) **Diffploit-Causing**，禁用致因 diff 的提取；(b) **Diffploit-Supporting**，禁用支持性 diff 的提取；(c) **Diffploit-Annealing**，移除 diff 退火过程。为了进一步评估我们的 diff 组合策略的有效性，我们额外设计了一个变体：(d) **Diffploit-Combining**，其中仅使用 diff 分数提供上下文信息，而不纳入 diff 之间的关系。不使用任何 diff 信息的基础模型性能已在表 1 中评估，因此我们不再为该设置单独列出一个变体。

我们使用表 2 中报告的三个指标来评估每个变体。**Average Step** 衡量成功迁移 exploit 所需适配步骤的平均数量，其中失败案例被赋予默认值 30 步，该值来源于在五分钟内完成 exploit 执行和 LLM 响应延迟的估计时间。**Success Rate** 报告成功迁移 exploit 的版本数量和百分比。**Overhead** 量化相对于原始 Diffploit 的平均步骤开销，从而提供成本效率的归一化视图。

**表 1：Diffploit 与基线方法在 Exploit 迁移上的性能**

| Library | CVE | Reference Version | Affected Versions Total | Need Mig. | Diffploit | TaRGET | IDEA | GPT-4o | DeepSeek-v3 |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| commons-fileupload | CVE-2016-1000031 | 1.1.1 | 8 | 3 | 3 | 0 | 0 | 3 | 3 |
| cxf-rt-rs-security-xml | CVE-2014-3584 | 2.6.10 | 33 | 14 | 0 | 0 | 0 | 0 | 0 |
| dolphinscheduler-api | CVE-2022-34662 | 2.0.0 | 20 | 12 | 0 | 0 | 0 | 0 | 0 |
| dolphinscheduler-common | CVE-2023-49250 | 2.0.0-alpha | 44 | 12 | 0 | 0 | 0 | 0 | 0 |
| dropwizard-validation | CVE-2020-5245 | 2.0.1 | 40 | 16 | 16 | 0 | 0 | 0 | 0 |
| hibernate-validator | CVE-2019-10219 | 6.0.5.Final | 31 | 12 | 12 | 0 | 0 | 1 | 12 |
| httpclient | CVE-2020-13956 | 4.5.3 | 40 | 6 | 0 | – | 0 | 0 | 0 |
| hutool-core | CVE-2023-51075 | 5.7.18 | 38 | 1 | 0 | – | 0 | 0 | 0 |
| jackson-databind | CVE-2022-42004 | 2.0.0-RC1 | 24 | 5 | 2 | 0 | 0 | 0 | 0 |
| jackson-dataformat-xml | CVE-2016-7051 | 2.7.7 | 75 | 69 | 69 | 0 | 0 | 0 | 0 |
| junrar | CVE-2022-23596 | 6.0.0 | 15 | 8 | 8 | 0 | 0 | 5 | 8 |
| kernel | CVE-2022-24197 | 7.2.0 | 24 | 5 | 0 | 0 | 0 | 0 | 5 |
| netty-codec-http | CVE-2021-43797 | 4.1.49.Final | 157 | 86 | 86 | 80 | 0 | 80 | 0 |
| netty-codec-http | CVE-2019-20444 | 4.1.43.Final | 131 | 71 | 71 | 65 | 65 | 1 | 65 |
| netty-codec-http | CVE-2019-16869 | 4.1.0.Beta1 | 131 | 71 | 71 | 65 | 65 | 1 | 65 |
| netty | CVE-2015-2156 | 3.3.0.Final | 65 | 45 | 0 | 0 | 0 | 0 | 0 |
| para-core | CVE-2022-1848 | 1.42.2 | 102 | 84 | 84 | 0 | 21 | 84 | 23 |
| plexus-utils | CVE-2017-1000487 | 1.4.2 | 50 | 2 | 2 | 2 | 0 | 2 | 2 |
| postgresql | CVE-2024-1597 | 9.4.1212 | 179 | 49 | 48 | 0 | 0 | 0 | 0 |
| protocols-imap | CVE-2021-40111 | 3.5.0 | 11 | 9 | 9* | 0 | 0 | 0 | 0 |
| socket.io-client | CVE-2022-25867 | 1.0.0 | 12 | 9 | 9 | 0 | 0 | 0 | 0 |
| spring-amqp | CVE-2017-8045 | 2.1.0.RELEASE | 49 | 2 | 2 | 0 | 0 | 2 | 2 |
| spring-actuator-logview | CVE-2021-21234 | 0.2.9 | 14 | 10 | 10 | 10 | 5 | 5 | 10 |
| spring-context | CVE-2022-22968 | 4.2.9.RELEASE | 194 | 19 | 19 | 0 | 0 | 19 | 19 |
| spring-security-core | CVE-2019-11272 | 3.0.0.RELEASE | 61 | 9 | 9 | 0 | 0 | 0 | 0 |
| spring-security-core | CVE-2024-22257 | 5.7.11 | 210 | 9 | 9 | 0 | 0 | 0 | 9 |
| spring-webmvc | CVE-2014-3625 | 3.0.4.RELEASE | 30 | 1 | 1 | 0 | 0 | 1 | 1 |
| spring-web | CVE-2013-6430 | 1.1.1 | 53 | 4 | 4 | 0 | 0 | 0 | 0 |
| spring-web | CVE-2020-5421 | 5.2.8.RELEASE | 124 | 39 | 39 | 0 | 0 | 39 | 39 |
| wicket-core | CVE-2013-2055 | 1.5.10 | 33 | 7 | 7 | 0 | 0 | 0 | 0 |
| SUM | 30 CVEs | – | 1,998 | 689 | 580 (84.2%) | 222 (32.2%) | 156 (22.6%) | 243 (35.2%) | 263 (38.1%) |

\* 该 exploit 是一个不稳定测试（flaky test）。

**表 2：Diffploit 的消融实验。**

| Method | Average Step* | Success Rate | Overhead |
|---|---:|---:|---:|
| Diffploit | 8.28 | 580/689 (84.2%) | - |
| Diffploit-Causing | 12.71 | 470/689 (68.2%) | 277.15% |
| Diffploit-Supporting | 11.15 | 489/689 (71.0%) | 159.09% |
| Diffploit-Annealing | 15.02 | 462/689 (67.1%) | 444.69% |
| Diffploit-Combining | 13.35 | 505/689 (73.3%) | 400.92% |

\* 失败案例被赋予默认步骤数。

从表 2 可以看出，Diffploit 达到了最高的成功率，比表现最好的消融变体高出 10.9% 以上。这证实了我们设计中的每个组件都对整体成功作出了贡献。在这些变体中，虽然移除任何单个模块都会导致成功率下降，但它们都明显高于基础模型（单独在表 1 中评估），表明上下文感知的 diff 和退火策略带来了显著收益。在成本方面，Diffploit 不仅成功率最高，而且平均所需步骤最少（8.28），说明 Diffploit 的每个组件都对整体性能作出了有效贡献。尽管这三个变体的成功率相近，但它们带来的复现成本不同，其中去掉 Annealing 的变体表现最差（平均 15.02 步，相对原始 Diffploit 的开销为 444.69%）。这凸显了退火过程在筛选和优先排序对迁移最有帮助的 diff 方面的重要性。尽管 Diffploit-Combining 在所有变体中取得了最高的成功率，但它所需步骤显著多于 Diffploit，说明我们的组合策略大幅提升了发现有效 exploit 的效率。

## 5.3 实际可行性

### 5.3.1 可接受性

我们希望通过评估迁移后的 exploit 是否会被现实世界的漏洞管理流程所接受，对 Diffploit 的实际可行性给出一种客观评估。一些适配后的 exploit 针对的是已不再维护的早期分支，因此无法评估库维护者对 Diffploit 的接受程度。为解决这一问题，我们通过提交那些此前未被记录、但由迁移后 exploit 所支持的受影响版本来评估 Diffploit 的接受度。我们将此前未记录的受影响版本识别出来并报告给 CVE——最权威的漏洞存储库——以及 GitHub Advisory Database，后者支持通过基于 pull request 的提交通信。在这些提交中，我们的迁移后 exploit 作为支持性证据。我们根据 CNA 和开源审查者的反馈来评估 Diffploit 在现实世界中的接受情况。

Diffploit 成功将 exploit 迁移到了 580 个受影响版本。我们进一步调查这些版本是否在 NVD 描述以及 GitHub Advisory Database 的受影响产品列表中被明确记录。如表 3 所示，我们的迁移后 exploit 发现了 5 条 NVD 记录，其受影响版本范围的说明缺失或不清晰；同时还发现 GitHub Advisory Database 遗漏了 111 个受影响版本。我们通过电子邮件联系了相应的 CNA，其中 3 家已将我们的迁移后 exploit 纳入 CVE 报告的参考链接中，而其余 CNA 在投稿时仍未回复。我们还向 GitHub Advisory Database 提交了 6 个 pull request，其中促成了 57 个受影响版本的更新。其余提交之所以未被合并，是因为 GitHub 在大规模验证 exploit 方面能力有限 [3]。我们的 exploit 被加入 CVE 参考链接，以及根据迁移后 exploit 识别出的受影响版本得到更新，这些都证明了我们的实际可行性。

**表 3：Diffploit 在现实世界中的反馈。**

| CNA | CVE | CNA Response | GitHub Missing |
|---|---|---|---|
| Mitre | CVE-2019-20444 | Confirmed | 14 (12 Accepted) |
| Mitre | CVE-2019-16869 | Confirmed | 2 (0 Accepted) |
| GitHub, Inc. | CVE-2021-43797 | Under Review | 13 (Under Review) |
| GitHub, Inc. | CVE-2020-5245 | - | 19 (7 Accepted) |
| VMWare | CVE-2024-22257 | Under Review | - |
| Red Hat, Inc. | CVE-2013-6430 | - | 38 (38 Accepted) |
| Red Hat, Inc. | CVE-2019-10219 | Confirmed | 25 (25 Accepted) |
| SUM | 7 CVEs | 3 Confirmed | 111 (82 Accepted) |

Diffploit 通过识别额外的受影响版本并提供可运行的 exploit，帮助提升了 CVE 报告的质量。正如 Red Hat 所确认的那样：“Thank you again for helping us improve the CVE records. The changes for version information were made, and the references you sent added.”

### 5.3.2 成本

我们估算了使用基础模型 DeepSeek-V3 运行 Diffploit 迁移一个 exploit 的经济成本。平均而言，每个成功迁移的 exploit 会消耗 7,029 个输入 token 和 831 个输出 token。根据 DeepSeek 截至 2025 年 7 月提供的定价细节，这对应于每次成功迁移平均 $0.0014 的成本。类似地，对于失败的迁移，平均 token 消耗为 12,977 个输入 token 和 997 个输出 token，导致每次失败尝试的平均成本为 $0.0020。这些结果表明，使用 Diffploit 的经济成本较低，适合在现实场景中部署。

### 5.3.3 数据泄漏

为评估 Diffploit 在未见过的漏洞上的表现，我们使用模型截止日期之后披露的 exploit 进行实验。我们所有实验使用的基础模型都是 Deepseek-V3-0324，因此我们选择了 5 个在 2025 年 3 月之后发布的 CVE。

如表 4 所示，Diffploit 在 4 个未见过的 CVE 上成功为其所有受影响版本迁移了 exploit，只有 CVE-2025-53864 的两个版本未能生成有效 exploit。这些结果表明，我们的方法在处理底层 LLM 之前从未见过的漏洞时仍然具有稳健性。

# 6 讨论

本节首先考察 Diffploit 在多大程度上解决了所识别的挑战。随后，我们讨论潜在的效度威胁，以及它们可能如何影响我们对实验结果的解释。

## 6.1 挑战应对

**表 4：对截止日期后披露漏洞的评估。**

| CVE-ID | 发布时间 | 受影响版本数 | 成功数 | 总数 | 需要迁移 |
|---|---|---:|---:|---:|---:|
| CVE-2025-59340 | 2025-09-17 | 92 | 19 | 19 | 19 |
| CVE-2025-58056 | 2025-09-03 | 227 | 71 | 71 | 71 |
| CVE-2025-53864 | 2025-07-10 | 268 | 192 | 190 | 190 |
| CVE-2025-52999 | 2025-06-25 | 183 | 28 | 28 | 28 |
| CVE-2025-48976 | 2025-06-16 | 12 | 2 | 2 | 2 |

我们进一步讨论 Diffploit 如何应对我们提出的挑战。我们考察 Diffploit 在解决两个主要挑战方面的有效性：❶ 由环境损坏导致的失败，以及 ❷ 漏洞利用迁移过程的复杂性。

**❶ 动态环境损坏。** 为了理解 Diffploit 是否解决了由动态环境损坏导致的失败，我们检查了迁移后漏洞利用中的修改位置。在 23 个成功迁移的漏洞利用中，有 7 个不需要对漏洞利用代码进行修改。这些迁移使 Diffploit 成功复现了 128 个此前失败的版本，这表明许多原始漏洞利用的失败并非源于漏洞利用逻辑本身，而是由外部或环境因素引起。我们进一步归类了 Diffploit 在不修改漏洞利用代码的情况下所处理的失败类型。其中包括：（1）由漏洞利用所需其他依赖引起的不兼容，例如 spring-web 与 spring-test 的版本不匹配；（2）由 Maven 配置中被阻断或已废弃的仓库引起的构建期问题，例如 repo.spring.io；（3）由于 Java 平台演化导致的运行时失败（例如 Java 11 中的 JAXB）；以及（4）与过时或 snapshot 构件有关的依赖解析失败（例如 commons-io、javax.faces:jsf-api），或第三方库中的冲突（例如 Javassist）。这些结果表明，Diffploit 通过在 pom 文件中重建必要的构建与运行时上下文，能够有效解决源于环境问题的失败。

**❷ 复杂迁移。** 我们使用平均编辑距离（Average Edit Distance, AED）来评估 Diffploit 的迁移复杂度；AED 是迁移任务中广泛采用的指标。AED 被定义为一组漏洞利用对上按 token 级别计算的 Levenshtein 距离的平均值。具体而言，我们计算迁移前后漏洞利用之间的 AED，以评估 Diffploit 所迁移漏洞利用的整体复杂度。此外，为了估计仅依赖 IDEA 进行迁移时所需的人工工作量，我们计算仅使用 IDEA 修改后的漏洞利用与 Diffploit 生成的相应有效漏洞利用之间的 AED。该比较量化了 Diffploit 实现的人工工作量减少。

在 Diffploit 成功完成漏洞利用迁移的 22 个 CVE 上，迁移后漏洞利用与原始漏洞利用之间的平均编辑距离为 176.00，这表明所需修改并非微不足道，而是体现出相当大的结构性或语义性变化。相比之下，IDEA 只能在其中 7 个案例中提供有用建议。即便应用了其建议，得到的漏洞利用仍然平均需要 59.69 的编辑距离，才能与 Diffploit 生成的有效漏洞利用相匹配。基于这些观察，我们估计，与 IDEA 相比，Diffploit 在漏洞利用迁移中按总编辑距离衡量大约减少了 80% 的人工工作量。这突出了 Diffploit 不仅能够处理复杂迁移场景，还能显著减轻与漏洞利用迁移相关的人工负担。

## 6.2 定性分析

如第 5.1.3 节所讨论，我们已经分析了方法的局限性。为补充该讨论，我们进一步对失败案例进行了定性分析，这些案例大体可归纳为三类根因。理解这些类别有助于解释当前仍存在的差距，并为未来改进提供指引。

**断言不匹配。** Diffploit 在迁移过程中会强制保留原始断言。一些漏洞在不同版本上会表现出不同的行为，从而导致原始漏洞利用断言在目标版本上失效。表 5 列出了观测行为与原始行为不一致的代表性示例。

**表 5：断言不匹配示例。**

| CVE | 期望 | 实际 |
|---|---|---|
| CVE-2020-13956 | 错误的返回值 | NumberFormatException |
| CVE-2023-51075 | 超时 | IndexOutOfBoundsException |
| CVE-2023-49250 | 控制台输出 | 远程服务器连接 |
| CVE-2022-42004 | StackOverflowError | 非预期异常 |
| CVE-2024-1597 | 错误的返回值 | VM 崩溃 |

**领域特定输入。** 某些漏洞利用需要高度专门化、特定格式的载荷，而这类载荷无法仅凭代码 diff 或浅层上下文被可靠地合成。例子包括精心构造的 PDF（如 CVE-2022-24197）以及结构化 token 或 cookie 名称（如 CVE-2014-3584、CVE-2015-2156）。这些构件通常需要精确到字节级或具备格式感知能力的修改，这超出了 Diffploit 当前的能力范围。

**大型 API 鸿沟。** 当目标版本引入大幅结构变化（多处同时编辑、API 重命名或组件替换）时，Diffploit 的搜索与编辑合成可能只能部分完成所需的转换。例如，CVE-2022-34662 需要替换一个服务类并找到适当的替代 API（从 `ResourcesServiceImpl.verifyFile` 到 `ResourcesService.updateResource`）；Diffploit 只能完成其中一部分迁移，导致漏洞利用仍然无法工作。

## 6.3 有效性威胁

我们的研究主要面临以下有效性威胁：

**外部有效性。** 主要威胁在于数据集的普适性。为缓解这一问题，我们使用当前公开可用的最大 Java 漏洞利用数据集来评估 Diffploit。不过，我们也承认，其在其他编程语言上的表现仍未得到验证。另一个威胁是，受影响版本范围是人工标注的，这可能引入人为错误。尽管我们进行了第二轮验证，并剔除了 22.7% 的错误标注，但仍可能存在被误分类的漏洞版本。Diffploit 依赖源代码，通过相应的 GitHub 仓库获取库版本之间的 diff。然而，对于某些项目，版本 diff 已不再公开可用，例如 spring-framework 3.0.0 之前的版本。在这种情况下，由于缺乏可访问的 diff，Diffploit 可能会失败。

**内部有效性。** 内部有效性的一个潜在威胁是，Diffploit 的性能可能来自基础模型此前见过已迁移漏洞利用，而非其真实的泛化能力。为检验这一点，我们进行了一个实验，仅使用基础模型来执行漏洞利用迁移。我们观察到，当完整方法被替换为仅使用基础模型时，成功率显著下降。该结果表明，Diffploit 的有效性并非源于潜在的数据泄漏，而是来自其基于 diff 的设计。

内部有效性的另一个威胁在于，Diffploit 依赖 `mvn` 命令进行构建（`mvn compile`）和管理库版本（`mvn versions:use-dep-version`），这限制了其在其他编程语言中的适用性。对此需要对编译和利用过程做进一步修改。Diffploit 依赖 `git diff` 生成目标版本与参考版本之间的 diff 文件，这也限制了其在其他版本控制系统中的适用性。

# 7 相关工作

我们的工作聚焦于漏洞利用迁移任务，即将现有漏洞利用适配到不同的软件版本。该任务对于现实场景中的准确漏洞评估和漏洞利用复现至关重要，因为环境差异和软件演化常常会使朴素的漏洞利用失效。近期研究已提出多种方法来处理漏洞利用迁移和跨版本可利用性评估：AEM [29] 提出了一种面向 Linux 内核的自动化漏洞利用迁移方法，该方法在不同内核版本之间对齐执行点，以复现利用行为。类似地，VulScope [13] 利用定向模糊测试在软件版本之间迁移漏洞利用，从而提升漏洞检测覆盖率。SyzBridge [73] 通过相应地调整漏洞利用，解决上游 Linux 内核与下游发行版之间的环境差异，从而提高可利用性评估的准确性。Evocatio [28] 则自动生成漏洞利用，以揭示此前未知的漏洞能力。这些工作凸显了漏洞利用迁移的复杂性及其实践重要性。上述方法依赖模糊测试，并需要显式的执行轨迹映射来支持漏洞利用迁移，这一过程既耗时又具有挑战性。

尽管漏洞利用迁移在漏洞分析中很重要，但在 Java 漏洞领域仍然研究不足，这正是我们方法的动机：在一个由 diff 驱动的框架中同时处理触发条件变化与环境损坏。

我们观察到，漏洞利用迁移与测试迁移和 API 迁移具有相似性。类似于自动化测试修复 [1, 14, 47]，随着软件系统快速演化、需要频繁更新以维持测试套件的可靠性与有效性 [15, 40]，自动化测试迁移正受到越来越多关注。已有工作包括 TaRGET，它利用预训练语言模型将测试修复视为一种语言翻译任务 [24, 48]。UTFix 专注于修复因焦点方法变化而受影响的 Python 单元测试，并使用上下文静态和动态代码切片以及失败消息 [46]。更早的框架如 TestCareAssistant 通过适应诸如方法参数增加之类的有限变化来修复或生成测试 [39]；而 TestFix 则使用遗传算法合成方法调用插入与删除，以修复损坏的 JUnit 测试，不过它仅支持单断言测试 [59]。为了更好地保留测试意图，TRIP 采用一种由动态符号执行引导的基于搜索的方法，优先选择那些能够维持原始测试语义的修复候选，并为被测用例生成修复 [36]。与测试迁移类似，API 迁移旨在更新下游项目中的 API 用法，以解决库更新带来的问题 [19, 58, 67]。这些方法可以部分解决库演化期间由 API 变化导致的漏洞利用失败。尽管已有这些进展，但由于导致漏洞利用失效的原因具有多样性，测试迁移和 API 迁移仍然充满挑战。此外，现有任务通常只关注函数级测试修复，忽视了环境因素带来的挑战。如何应对此类环境挑战仍是一个尚未被充分探索的领域。

# 8 结论

在本文中，我们提出了 Diffploit，这是一种新颖的漏洞利用迁移框架，它将 LLM 驱动的适配与基于特定版本代码 diff 的动态上下文构建结合起来。通过识别致因 diff 和支撑 diff，并采用模拟退火策略来指导迭代式适配，Diffploit 将失败症状转化为可执行的迁移步骤。这个由反馈驱动的过程使得跨越复杂版本差距进行可靠的漏洞利用迁移成为可能。通过在迄今为止最大的 Java 漏洞利用数据集上进行广泛实验，Diffploit 取得了 84.2% 的高成功率，并显著优于当前最先进的基线方法。我们的消融研究证实，每一个设计组件都对有效性和成本效率做出了有意义的贡献。此外，来自 CNA 和开源维护者的真实世界反馈也证实了我们方法在实践中的可行性：有 82 个受影响版本被 GitHub Advisory Database 接收，且 3 个 CVE 的 CNA 确认了我们基于迁移后漏洞利用识别出的受影响版本。尽管 Diffploit 展现出很强的适应性和通用性，但它在处理复现行为被修改的情况时仍存在局限。这些挑战指向了很有前景的未来研究方向。总体而言，我们的研究表明，Diffploit 不仅提升了漏洞利用在跨版本场景下的可用性，也有助于提高漏洞数据库的完整性和准确性。

## 致谢

本研究得到国家重点研发计划（No.2024YFB4506400）的资助。我们也感谢匿名评审提出的富有洞见的意见和建议。

## 参考文献

[1] Nadia Alshahwan, Jubin Chheda, Anastasia Finogenova, Beliz Gokkaya, Mark Harman, Inna Harper, Alexandru Marginean, Shubho Sengupta, and Eddy Wang. 2024. 在 Meta 使用大语言模型自动改进单元测试. In Companion Proceedings of the 32nd ACM International Conference on the Foundations of Software Engineering (Porto de Galinhas, Brazil) (FSE 2024). Association for Computing Machinery, New York, NY, USA, 185–196. doi:10.1145/3663529.3663839

[2] Gabin An, Jingun Hong, Naryeong Kim, and Shin Yoo. 2023. Fonte：从失败中发现引入缺陷的提交. In Proceedings of the 45th International Conference on Software Engineering (Melbourne, Victoria, Australia) (ICSE ’23). IEEE Press, 589–601. doi:10.1109/ICSE48619.2023.00059

[3] Anoymous. [n. d.]. 用于更新 CVE-2019-16868 的拉取请求. https://github.com/github/advisory-database/pull/5774

[4] Lingfeng Bao, Xin Xia, Ahmed E. Hassan, and Xiaohu Yang. 2022. V-SZZ：自动识别受 CVE 漏洞影响的版本范围. In Proceedings of the 44th International Conference on Software Engineering (Pittsburgh, Pennsylvania) (ICSE ’22). Association for Computing Machinery, New York, NY, USA, 2352–2364. doi:10.1145/3510003.3510113

[5] Gabriele Bavota, Gerardo Canfora, Massimiliano Di Penta, Rocco Oliveto, and Sebastiano Panichella. 2015. Apache 社区如何升级依赖：一项演化研究. Empirical Software Engineering 20 (2015), 1275–1317.

[6] Marcel Böhme, Van-Thuan Pham, Manh-Dung Nguyen, and Abhik Roychoudhury. 2017. 定向灰盒模糊测试. In Proceedings of the 2017 ACM SIGSAC Conference on Computer and Communications Security (Dallas, Texas, USA) (CCS ’17). Association for Computing Machinery, New York, NY, USA, 2329–2344. doi:10.1145/3133956.3134020

[7] Oscar Chaparro, Jing Lu, Fiorella Zampetti, Laura Moreno, Massimiliano Di Penta, Andrian Marcus, Gabriele Bavota, and Vincent Ng. 2017. 检测缺陷描述中的缺失信息. In Proceedings of the 2017 11th Joint Meeting on Foundations of Software Engineering (Paderborn, Germany) (ESEC/FSE 2017). Association for Computing Machinery, New York, NY, USA, 396–407. doi:10.1145/3106237.3106285

[8] Xiao Chen, Hengcheng Zhu, Jialun Cao, Ming Wen, and Shing-Chi Cheung. 2025. SemBIC：具备语义感知能力的引入缺陷提交识别. Proc. ACM Softw. Eng. 2, FSE, Article FSE062 (June 2025), 23 pages. doi:10.1145/3715781

[9] Zirui Chen. [n. d.]. 复现实验包. https://github.com/chen-zirui/PoCAdaptation

[10] Zirui Chen, Xing Hu, Xin Xia, Yi Gao, Tongtong Xu, David Lo, and Xiaohu Yang. 2024. 通过基于迁移的自动化测试生成利用库漏洞. In Proceedings of the IEEE/ACM 46th International Conference on Software Engineering (Lisbon, Portugal) (ICSE ’24). Association for Computing Machinery, New York, NY, USA, Article 228, 12 pages. doi:10.1145/3597503.3639583

[11] The MITRE Corporation. [n. d.]. CNA 详情. https://www.cve.org/PartnerInformation/ListofPartners

[12] Roland Croft, M. Ali Babar, and M. Mehdi Kholoosi. 2023. 软件漏洞数据集的数据质量. In Proceedings of the 45th International Conference on Software Engineering (Melbourne, Victoria, Australia) (ICSE ’23). IEEE Press, 121–133. doi:10.1109/ICSE48619.2023.00022

[13] Jiarun Dai, Yuan Zhang, Hailong Xu, Haiming Lyu, Zicheng Wu, Xinyu Xing, and Min Yang. 2021. 通过 PoC 迁移促进漏洞评估. In Proceedings of the 2021 ACM SIGSAC Conference on Computer and Communications Security (Virtual Event, Republic of Korea) (CCS ’21). Association for Computing Machinery, New York, NY, USA, 3300–3317. doi:10.1145/3460120.3484594

[14] Brett Daniel, Danny Dig, Tihomir Gvero, Vilas Jagannath, Johnston Jiaa, Damion Mitchell, Jurand Nogiec, Shin Hwei Tan, and Darko Marinov. 2011. ReAssert：一个修复损坏单元测试的工具. In Proceedings of the 33rd International Conference on Software Engineering (Waikiki, Honolulu, HI, USA) (ICSE ’11). Association for Computing Machinery, New York, NY, USA, 1010–1012. doi:10.1145/1985793.1985978

[15] Brett Daniel, Vilas Jagannath, Danny Dig, and Darko Marinov. 2009. ReAssert：为损坏的单元测试建议修复方案. In 2009 IEEE/ACM International Conference on Automated Software Engineering. 433–444. doi:10.1109/ASE.2009.17

[16] National Vulnerability Database. [n. d.]. CVE-2024-22257. https://nvd.nist.gov/vuln/detail/cve-2024-22257

[17] Peng Deng, Lei Zhang, Yuchuan Meng, Zhemin Yang, Yuan Zhang, and Min Yang. 2025. CHAINFUZZ：利用开源供应链中的上游漏洞. In Proceedings of the 34th USENIX Conference on Security Symposium (Seattle, WA, USA) (SEC ’25). USENIX Association, USA, Article 319, 20 pages.

[18] Ying Dong, Wenbo Guo, Yueqi Chen, Xinyu Xing, Yuqing Zhang, and Gang Wang. 2019. 面向公共安全漏洞报告中不一致性的检测. In Proceedings of the 28th USENIX Conference on Security Symposium (Santa Clara, CA, USA) (SEC’19). USENIX Association, USA, 869–885.

[19] Mattia Fazzini, Qi Xin, and Alessandro Orso. 2019. 面向 Android 应用的自动化 API 使用更新. In Proceedings of the 28th ACM SIGSOFT International Symposium on Software Testing and Analysis (Beijing, China) (ISSTA 2019). Association for Computing Machinery, New York, NY, USA, 204–215. doi:10.1145/3293882.3330571

[20] Yi Gao, Xing Hu, Zirui Chen, and Xiaohu Yang. 2025. 从第三方库生成漏洞触发测试用例. arXiv:2409.16701 [cs.SE] https://arxiv.org/abs/2409.16701

[21] Konstantin Grotov, Sergey Titov, Yaroslav Zharov, and Timofey Bryksin. 2024. 解开死结：利用 LLM 解决计算笔记本中的错误. arXiv:2405.01559 [cs.SE] https://arxiv.org/abs/2405.01559

[22] Hao He, Runzhi He, Haiqiao Gu, and Minghui Zhou. 2021. 关于 Java 库迁移的大规模实证研究：普遍性、趋势与动因. In Proceedings of the 29th ACM Joint Meeting on European Software Engineering Conference and Symposium on the Foundations of Software Engineering (Athens, Greece) (ESEC/FSE 2021). Association for Computing Machinery, New York, NY, USA, 478–490. doi:10.1145/3468264.3468571

[23] Runzhi He, Hao He, Yuxia Zhang, and Minghui Zhou. 2023. 实践中的依赖更新自动化：对 GitHub Dependabot 的探索性研究. IEEE Trans. Softw. Eng. 49, 8 (Aug. 2023), 4004–4022. doi:10.1109/TSE.2023.3278129

[24] Kevin J. Hoffman, Patrick Eugster, and Suresh Jagannathan. 2009. 语义感知的轨迹分析. SIGPLAN Not. 44, 6 (June 2009), 453–464. doi:10.1145/1543135.1542527

[25] Jiyong Jang, Abeer Agrawal, and David Brumley. 2012. ReDeBug：在整个操作系统发行版中发现未修补的代码克隆. In 2012 IEEE Symposium on Security and Privacy. 48–62. doi:10.1109/SP.2012.13

[26] Abbas Javan Jafari, Diego Elias Costa, Emad Shihab, and Rabe Abdalkareem. 2023. 依赖更新策略与软件包特征. ACM Trans. Softw. Eng. Methodol. 32, 6, Article 149 (Sept. 2023), 29 pages. doi:10.1145/3603110

[27] Jetbrains. [n. d.]. IDEA 官网. https://www.jetbrains.com/idea/

[28] Zhiyuan Jiang, Shuitao Gan, Adrian Herrera, Flavio Toffalini, Lucio Romerio, Chaojing Tang, Manuel Egele, Chao Zhang, and Mathias Payer. 2022. Evocatio：从单个 PoC 唤出漏洞能力. In Proceedings of the 2022 ACM SIGSAC Conference on Computer and Communications Security (Los Angeles, CA, USA) (CCS ’22). Association for Computing Machinery, New York, NY, USA, 1599–1613. doi:10.1145/3548606.3560575

[29] Zheyue Jiang, Yuan Zhang, Jun Xu, Xinqian Sun, Zhuang Liu, and Min Yang. 2023. AEM：促进 Linux 内核漏洞的跨版本可利用性评估. In 2023 IEEE Symposium on Security and Privacy (SP). 2122–2137. doi:10.1109/SP46215.2023.10179286

[30] Hyeonseong Jo, Jinwoo Kim, Phillip Porras, Vinod Yegneswaran, and Seungwon Shin. 2021. GapFinder：从非结构化文本中发现安全信息的不一致性. IEEE Transactions on Information Forensics and Security 16 (2021), 86–99. doi:10.1109/TIFS.2020.3003570

[31] Hong Jin Kang, Truong Giang Nguyen, Bach Le, Corina S. Păsăreanu, and David Lo. 2022. 通过测试拟态评估库漏洞的可利用性. In Proceedings of the 31st ACM SIGSOFT International Symposium on Software Testing and Analysis (Virtual, South Korea) (ISSTA 2022). Association for Computing Machinery, New York, NY, USA, 276–288. doi:10.1145/3533767.3534398

[32] Raula Gaikovina Kula, Daniel M German, Ali Ouni, Takashi Ishio, and Katsuro Inoue. 2018. 开发者会更新其库依赖吗？一项关于安全公告对库迁移影响的实证研究. Empirical Software Engineering 23 (2018), 384–417.

[33] Raula Gaikovina Kula, Daniel M German, Ali Ouni, Takashi Ishio, and Katsuro Inoue. 2018. 开发者会更新其库依赖吗？一项关于安全公告对库迁移影响的实证研究. Empirical Software Engineering 23 (2018), 384–417.

[34] Jia Li, Zhuo Li, Huangzhao Zhang, Ge Li, Zhi Jin, Xing Hu, and Xin Xia. 2024. 深度源代码处理模型上的投毒攻击与投毒检测. ACM Trans. Softw. Eng. Methodol. 33, 3, Article 62 (March 2024), 31 pages. doi:10.1145/3630008

[35] Siyuan Li, Yongpan Wang, Chaopeng Dong, Shouguo Yang, Hong Li, Hao Sun, Zhe Lang, Zuxin Chen, Weijie Wang, Hongsong Zhu, and Limin Sun. 2023. LibAM：一种用于在二进制文件中检测第三方库的区域匹配框架. ACM Trans. Softw. Eng. Methodol. 33, 2, Article 52 (Dec. 2023), 35 pages. doi:10.1145/3625294

[36] Xiangyu Li, Marcelo d’Amorim, and Alessandro Orso. 2019. 保留意图的测试修复. In 2019 12th IEEE Conference on Software Testing, Validation and Verification (ICST). IEEE Computer Society, Los Alamitos, CA, USA, 217–227. doi:10.1109/ICST.2019.00030

[37] Shuhan Liu, Jiayuan Zhou, Xing Hu, Filipe Roseiro Cogo, Xin Xia, and Xiaohu Yang. 2025. 关于开源软件系统漏洞披露管理的实证研究. ACM Trans. Softw. Eng. Methodol. 34, 7, Article 214 (Aug. 2025), 31 pages. doi:10.1145/3716822

[38] Looly. [n. d.]. CVE-2023-51080 的漏洞引入提交. https://github.com/chinabugotech/hutool/commit/c45b3f

[39] Mehdi Mirzaaghaei. 2011. 自动化测试套件演化. In Proceedings of the 19th ACM SIGSOFT Symposium and the 13th European Conference on Foundations of Software Engineering (Szeged, Hungary) (ESEC/FSE ’11). Association for Computing Machinery, New York, NY, USA, 396–399. doi:10.1145/2025113.2025172

[40] Mehdi Mirzaaghaei, Fabrizio Pastore, and Mauro Pezzè. 2014. 自动化测试用例演化. Softw. Test. Verif. Reliab. 24, 5 (Aug. 2014), 386–411. doi:10.1002/stvr.1527

[41] Dongliang Mu, Alejandro Cuevas, Limin Yang, Hang Hu, Xinyu Xing, Bing Mao, and Gang Wang. 2018. 理解众包报告安全漏洞的可复现性. In Proceedings of the 27th USENIX Conference on Security Symposium (Baltimore, MD, USA) (SEC’18). USENIX Association, USA, 919–936.

[42] Shengyi Pan, Lingfeng Bao, Xin Xia, David Lo, and Shanping Li. 2023. 基于 CWE 树结构的细粒度提交级漏洞类型预测. In 2023 IEEE/ACM 45th International Conference on Software Engineering (ICSE). 957–969. doi:10.1109/ICSE48619.2023.00088

[43] Shengyi Pan, Lingfeng Bao, Jiayuan Zhou, Xing Hu, Xin Xia, and Shanping Li. 2024. 迈向更实用的漏洞评估自动化. In Proceedings of the IEEE/ACM 46th International Conference on Software Engineering (Lisbon, Portugal) (ICSE ’24). Association for Computing Machinery, New York, NY, USA, Article 148, 13 pages. doi:10.1145/3597503.3639110

[44] Shengyi Pan, You Wang, Zhongxin Liu, Xing Hu, Xin Xia, and Shanping Li. 2024. 面向硬分叉的零样本补丁移植自动化. In Proceedings of the 33rd ACM SIGSOFT International Symposium on Software Testing and Analysis (Vienna, Austria) (ISSTA 2024). Association for Computing Machinery, New York, NY, USA, 363–375. doi:10.1145/3650212.3652134

[45] Shengyi Pan, Jiayuan Zhou, Filipe Roseiro Cogo, Xin Xia, Lingfeng Bao, Xing Hu, Shanping Li, and Ahmed E. Hassan. 2022. 自动挖掘危险的问题报告. In Proceedings of the 30th ACM Joint European Software Engineering Conference and Symposium on the Foundations of Software Engineering (Singapore, Singapore) (ESEC/FSE 2022). Association for Computing Machinery, New York, NY, USA, 834–846. doi:10.1145/3540250.3549156

[46] Shanto Rahman, Sachit Kuhar, Berk Cirisci, Pranav Garg, Shiqi Wang, Xiaofei Ma, Anoop Deoras, and Baishakhi Ray. 2025. UTFix：使用 LLM 的变更感知单元测试修复. Proc. ACM Program. Lang. 9, OOPSLA1, Article 85 (April 2025), 26 pages. doi:10.1145/3720419

[47] Shanto Rahman and August Shi. 2024. FlakeSync：自动修复异步不稳定测试. In Proceedings of the IEEE/ACM 46th International Conference on Software Engineering (Lisbon, Portugal) (ICSE ’24). Association for Computing Machinery, New York, NY, USA, Article 136, 12 pages. doi:10.1145/3597503.3639115

[48] Ahmadreza Saboor Yaraghi, Darren Holden, Nafiseh Kahani, and Lionel Briand. 2025. 使用语言模型自动修复测试用例. IEEE Trans. Softw. Eng. 51, 4 (Feb. 2025), 1104–1133. doi:10.1109/TSE.2025.3541166

[49] Youkun Shi, Yuan Zhang, Tianhan Luo, Xiangyu Mao, and Min Yang. 2023. 面向 Web 漏洞的精确受影响/未受影响版本分析. In Proceedings of the 37th IEEE/ACM International Conference on Automated Software Engineering (Rochester, MI, USA) (ASE ’22). Association for Computing Machinery, New York, NY, USA, Article 76, 13 pages. doi:10.1145/3551349.3556933

[50] Joseph Spracklen, Raveen Wijewickrama, A H M Nazmus Sakib, Anindya Maiti, Bimal Viswanath, and Murtuza Jadliwala. 2025. 我们有一个包给你！对代码生成 LLM 包幻觉的全面分析. arXiv:2406.10279 [cs.SE] https://arxiv.org/abs/2406.10279

[51] Synopsys. [n. d.]. 2023 年开源安全与风险分析报告. https://www.synopsys.com/software-integrity/resources/analyst-reports/open-source-security-risk-analysis.html

[52] Tekul. [n. d.]. Spring Security 提交. https://github.com/spring-projects/spring-security/commit/76438b

[53] Trustin. [n. d.]. CVE-2021-43795 的漏洞引入提交. https://github.com/line/armeria/commit/e2697a575e9df6692b423e02d731f293c1313284

[54] Gladys Tyen, Hassan Mansoor, Victor Cărbune, Peter Chen, and Tony Mak. 2024. LLM 无法发现推理错误，但在给出错误位置后可以纠正它们. arXiv:2311.08516 [cs.AI] https://arxiv.org/abs/2311.08516

[55] Vision-version. [n. d.]. CVE-2020-5245 的漏洞利用. https://github.com/vision-version/vision-version.github.io/blob/main/Vision/1.groundtruth/testcase-trigger/testcase-trigger/CVE-2020-5245/pom.xml

[56] Siqi Wang, Xing Hu, Bei Wang, Wenxin Yao, Xin Xia, and Xinyu Wang. 2025. 深度学习代码重构：实践与未满足工具需求研究. In International Conference on Software Maintenance and Evolution.

[57] Susheng Wu, Ruisi Wang, Kaifeng Huang, Yiheng Cao, Wenyan Song, Zhuotong Zhou, Yiheng Huang, Bihuan Chen, and Xin Peng. 2024. Vision：识别开源软件漏洞受影响的库版本. In Proceedings of the 39th IEEE/ACM International Conference on Automated Software Engineering (Sacramento, CA, USA) (ASE ’24). Association for Computing Machinery, New York, NY, USA, 1447–1459. doi:10.1145/3691620.3695516

[58] Shengzhe Xu, Ziqi Dong, and Na Meng. 2019. Meditor：API 迁移编辑的推断与应用. In 2019 IEEE/ACM 27th International Conference on Program Comprehension (ICPC). 335–346. doi:10.1109/ICPC.2019.00052

[59] Yong Xu, Bo Huang, Guoqing Wu, and Mengting Yuan. 2014. 使用遗传算法修复 JUnit 测试用例. In 2014 21st Asia-Pacific Software Engineering Conference, Vol. 1. 287–294. doi:10.1109/APSEC.2014.51

[60] Zhipeng Xue, Zhipeng Gao, Xing Hu, and Shanping Li. 2023. ACWRecommender：一种利用弱监督验证可执行警告的工具. In 2023 38th IEEE/ACM International Conference on Automated Software Engineering (ASE). IEEE, 1876–1880.

[61] Zhipeng Xue, Zhipeng Gao, Shaohua Wang, Xing Hu, Xin Xia, and Shanping Li. 2024. Selfpico：借助 LLM 的自引导部分代码执行. In Proceedings of the 33rd ACM SIGSOFT International Symposium on Software Testing and Analysis. 1389–1401.

[62] Yanming Yang, Xing Hu, Zhipeng Gao, Jinfu Chen, Chao Ni, Xin Xia, and David Lo. 2024. 面向软件工程的联邦学习：代码克隆检测与缺陷预测案例研究. IEEE Transactions on Software Engineering 50, 2 (2024), 296–321. doi:10.1109/TSE.2023.3347898

[63] Zhengmin Yu, Yuan Zhang, Ming Wen, Yinan Nie, Wenhui Zhang, and Min Yang. 2025. CXXCrafter：一个基于 LLM 的自动化 C/C++ 开源软件构建代理. Proc. ACM Softw. Eng. 2, FSE, Article FSE116 (June 2025), 23 pages. doi:10.1145/3729386

[64] Qi Zhan, Xing Hu, Zhiyang Li, Xin Xia, David Lo, and Shanping Li. 2024. PS3：基于语义符号签名的精确补丁存在性测试. In Proceedings of the IEEE/ACM 46th International Conference on Software Engineering (Lisbon, Portugal) (ICSE ’24). Association for Computing Machinery, New York, NY, USA, Article 167, 12 pages. doi:10.1145/3597503.3639134

[65] Qi Zhan, Xing Hu, Xin Xia, and Shanping Li. 2024. REACT：面向二进制文件的 IR 级补丁存在性测试. In Proceedings of the 39th IEEE/ACM International Conference on Automated Software Engineering (Sacramento, CA, USA) (ASE ’24). Association for Computing Machinery, New York, NY, USA, 381–392. doi:10.1145/3691620.3695012

[66] Zheng Zhang, Yu Hao, Weiteng Chen, Xiaochen Zou, Xingyu Li, Haonan Li, Yizhuo Zhai, Zhiyun Qian, and Billy Lau. 2024. SymBisect：面向模糊测试暴露漏洞的精确二分定位. In Proceedings of the 33rd USENIX Conference on Security Symposium (Philadelphia, PA, USA) (SEC ’24). USENIX Association, USA, Article 140, 18 pages.

[67] Hao Zhong and Na Meng. 2024. 编译器导向的客户端代码 API 调用点迁移. In Proceedings of the IEEE/ACM 46th International Conference on Software Engineering (Lisbon, Portugal) (ICSE ’24). Association for Computing Machinery, New York, NY, USA, Article 226, 12 pages. doi:10.1145/3597503.3639084

[68] Jiayuan Zhou, Michael Pacheco, Jinfu Chen, Xing Hu, Xin Xia, David Lo, and Ahmed E. Hassan. 2023. CoLeFunDa：可解释的静默漏洞修复识别. In Proceedings of the 45th International Conference on Software Engineering (Melbourne, Victoria, Australia) (ICSE ’23). IEEE Press, 2565–2577. doi:10.1109/ICSE48619.2023.00214

[69] Jiayuan Zhou, Michael Pacheco, Zhiyuan Wan, Xin Xia, David Lo, Yuan Wang, and Ahmed E. Hassan. 2022. 大海捞针：自动挖掘静默漏洞修复. In Proceedings of the 36th IEEE/ACM International Conference on Automated Software Engineering (Melbourne, Australia) (ASE ’21). IEEE Press, 705–716. doi:10.1109/ASE51524.2021.9678720

[70] Zhuotong Zhou, Yongzhuo Yang, Susheng Wu, Yiheng Huang, Bihuan Chen, and Xin Peng. 2024. Magneto：一种借助 LLM 赋能定向模糊测试来利用依赖库漏洞的分步方法. In Proceedings of the 39th IEEE/ACM International Conference on Automated Software Engineering (Sacramento, CA, USA) (ASE ’24). Association for Computing Machinery, New York, NY, USA, 1633–1644. doi:10.1145/3691620.3695531

[71] Markus Zimmermann, Cristian-Alexandru Staicu, Cam Tenny, and Michael Pradel. 2019. 小世界，高风险：对 npm 生态系统中安全威胁的研究. In 28th USENIX Security Symposium (USENIX Security 19). USENIX Association, Santa Clara, CA, 995–1010.

[72] Anni Zou, Wenhao Yu, Hongming Zhang, Kaixin Ma, Deng Cai, Zhuosheng Zhang, Hai Zhao, and Dong Yu. 2024. DOCBENCH：用于评估基于 LLM 的文档阅读系统的基准. arXiv:2407.10701 [cs.CL] https://arxiv.org/abs/2407.10701

[73] Xiaochen Zou, Yu Hao, Zheng Zhang, Juefei Pu, Weiteng Chen, and Zhiyun Qian. 2024. SyzBridge：弥合 Linux 生态系统中 Linux 内核缺陷可利用性评估的鸿沟. NDSS (2024). doi:10.14722/ndss.2024.24926
