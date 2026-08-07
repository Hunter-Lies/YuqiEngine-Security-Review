# 宇奇引擎（YuqiEngine）安全评估与综合评估报告

> 评估对象：YuqiEngine v0.1.2（个人开发的 Windows 游戏/系统优化工具）
> 评估日期：2026-08-08 ｜ 方式：反编译源码审查 + 运行时日志取证 + 官方 README 对照
> 声明：本报告立场中立，欢迎作者提供原始代码与完整测试环境以供复核；所有"未发现/未证实"结论均保持相应措辞。

---

## 太长不看（普通用户版）

**一句话结论：这软件不是病毒，但对普通用户来说风险大于收益——它能把你的电脑搞到开不了机，而且它承诺的"还原点兜底"经常靠不住。**

### 你需要知道的 6 件事

1. **它会让你"一键关闭系统防护"**——UAC、防火墙、Defender、Windows 更新、DEP、CFG、HVCI、VBS、Spectre 缓解等，确认一下就全关。关了之后你的电脑等于裸奔。
2. **它会改启动配置（BCD）**——包括 `tscsyncpolicy`、`useplatformtick` 这类高风险计时器项，改完要重启才生效。它自己的代码警告写着：**"错误值可能导致无法启动、回滚困难"**。
3. **它说会先创建"系统还原点"兜底，但创建失败时会"只提醒、继续执行"**——你以为有还原点，实际经常没有。
4. **实测它会疯狂拉起系统命令**——一次运行 41 分钟内拉了 341 次 `powercfg` 子进程，同一秒最多 41 次。这就是很多人反馈"CPU 看着满载、软件卡住"的原因，低配电脑可能直接死机。
5. **软件本体没有代码签名**——Windows 提示"未知发布者"；而且它的更新系统要求签名校验，未签名的本体永远过不了自己的更新，等于收不到更新。
6. **已有真实事故**——B站评论区："优化了一波，电脑顺利死机了，重置也死机""已经完全打不开了，无限重启"。

### 普通用户应该怎么做

- ✅ **只用它的"检测/诊断"功能**，不要点"应用/优化"
- ❌ **不要碰**任何"关闭安全防护 / 修改启动配置 / 关闭缓解"类选项
- ❌ **不要用"一键优化"**
- ✅ 如果一定要试：先手动创建系统还原点 + `bcdedit /export` 备份，再逐项小范围测试
- ✅ 出现启动异常：用 Windows 自带启动修复/还原点，别指望这软件救你

---

## 详细分析（技术版）

## 评估对象与方法

- 对象：YuqiEngine v0.1.2（.NET 8 + WebView2，安装包约 530 MB，含自包含运行时与离线 WebView2）
- 方法：反编译源码静态审查（`GameOptimizer.Core` 约 1308 个类）、运行时进程与日志观察、更新链路与网络行为核查、官方 README 对照
- 参考：作者官方 README、B站评论区用户反馈

## 核心安全发现

### 1.（最重要）安全兜底是软性的：还原点创建失败"只提醒、继续执行"

软件对高风险操作宣称"必须先创建系统还原点"，但三条失败路径均**继续执行**：

- 内置文案（`GameOptimizer.Core.decompiled.cs` 资源表 98489-98491）：
  - "系统还原点当前不可创建，**已作为风险提醒继续执行**"
  - "系统还原点创建失败，**已作为风险提醒继续执行**"
  - "系统还原点创建异常，**已作为风险提醒继续执行**"
- 调用代码（`YuqiEngine.decompiled.cs` 79525-79585，`TryCreateOneClickBackupReminderAsync`）：三种情况均返回 `RestorePointCreated:false` 并附提示，随后**继续执行优化**
- 门禁策略 `RestorePointCreationGuardPolicy.Evaluate`（行 140443）仅检查"当前是否可创建"，失败不阻断
- 还原点通过 PowerShell `Checkpoint-Computer` 创建（行 148191，含 `-ExecutionPolicy Bypass`），该方式在部分系统上本就容易失败；且存在"复用窗口"（具体时长未核实），一段时间内跳过重复创建

**含义**：用户以为有还原点兜底，实际经常没有。这是软件所有"安全承诺"中最核心的一条，但它是**软门禁**。

### 2. 修改启动配置（BCD）无导出备份，官方警告明说"可致无法启动、回滚困难"

提供以下 BCD 修改（`BcdSettingOperation`，行 68299；操作目录 60563-60575、70485-70553），均要求重启生效：

- `bcdedit /set {current} tscsyncpolicy Legacy`
- `bcdedit /set {current} useplatformtick yes`
- `bcdedit /set {current} disabledynamictick yes`
- 删除 `useplatformclock` 绑定

操作自带警告："BCD boot-entry changes directly affect startup and timer behavior. **Incorrect values can cause boot failure, timing instability, or difficult rollback.**"

修改前**未发现 `bcdedit /export` 备份**（全库唯一导出在"修复中心"路径，行 28903）；恢复依赖"修改前值快照"重新写回，**系统起不来时软件无法运行，快照即不可用**。用户"优化后无限重启"与该路径吻合（相关性判断，非因果断言）。

### 3. 安全缓解"仅引导恢复"，关闭后拒绝自动回滚

- `cpu.disable_spectre_meltdown` 写入 `FeatureSettingsOverride/Mask=3`（行 28835、50103）
- 恢复项为"仅引导恢复"（`CreateGuidedOnlyRevert`，行 35370）："Spectre/Meltdown 缓解策略属于安全边界，**不能按性能优化的反向值盲目恢复**"

用户关闭缓解后系统不稳定时，软件不提供自动回滚。

### 4. 可一键关闭全部系统安全防护

操作目录确认存在：`basic.disable_uac`（写 `EnableLUA=0`）、`basic.disable_firewall`、`basic.disable_security_center`、`basic.disable_windows_update`、`security.disable_defender_realtime`（写 `DisableRealtimeMonitoring=1`，行 87549）、`security.disable_dep/cfg/sehop/hvci/vbs/smartscreen`、`security.prevent_bitlocker_device_encryption` 等十余项。门禁为"确认 + 还原点"（行 50671-50673），但结合发现 1，**还原点这一层可被跳过**。

### 5.（实测）powercfg 子进程风暴

- 代码：`PowerCfgUtility.GetActiveSchemeAsync` 每次查询都启动 `powercfg /GETACTIVESCHEME` 子进程（行 84216），多页面探针高频调用
- 实测日志（`%LOCALAPPDATA%\YuqiEngine\Logs\yuqiengine-20260808.log`）：41 分钟内 **341 次** powercfg 子进程，**单秒峰值 41 次**（03:35:06），多次 35 次/秒
- 影响：CPU 占用飙升、界面卡死，低配机可直接表现为死机

### 6. 分发包未签名，且与自身更新系统自相矛盾

- 实测：`YuqiEngine.exe`（288MB）与 `Guard\YuqiGuard.exe`（80MB）Authenticode 均为 **NotSigned**
- 更新系统要求"当前进程与更新包必须通过 Authenticode 校验且发布者一致"（`VerifyPackagePublisherTrust`）→ **未签名本体永远无法通过自身更新校验**，用户实际收不到更新
- 作者 README 已承认未购买受信任证书，并明确"SHA-256 只确认完整性，不能代替签名"，但未解决该矛盾

## "既要又要"矛盾清单

> 证据列格式：`文件：行号（代码含义）`。行号对应评估所用的反编译产物（见附录 B），供复核使用；结论本身以行为描述与附录索引为准。

| # | 既要…… | 又要…… | 证据 |
|---|---|---|---|
| 1 | 安全兜底：要求用户先创建系统还原点 | 失败不阻断：还原点不可用/失败/异常时"提醒后继续执行" | `YuqiEngine.decompiled.cs` 79525-79585（还原点失败路径：失败仍返回并继续执行）；140443（门禁仅检查"能否创建"） |
| 2 | 只处理可自动恢复项（README/守护助手） | 提供"仅引导恢复"的安全缓解开关，关闭后拒绝自动回滚 | `GameOptimizer.Core.decompiled.cs` 35370（`CreateGuidedOnlyRevert`：缓解项仅引导恢复）；50103（`cpu.disable_spectre_meltdown` 操作定义） |
| 3 | 不要批量应用（README） | 提供一键优化入口，且还原点门禁同样非阻断 | `GameOptimizer.Core.decompiled.cs` 7342（"可按需执行一键优化"推荐）；`YuqiEngine.decompiled.cs` 79500 起（`[OneClickGuard]` 一键批量流程） |
| 4 | 可校验更新（README 提供 SHA-256） | 未签名本体导致自身更新校验永远无法通过 | 实测 `Get-AuthenticodeSignature`：YuqiEngine.exe / YuqiGuard.exe 均 NotSigned；`UpdateCheckService` 更新要求"进程与更新包 Authenticode 签名一致" |
| 5 | 专业诊断/恢复（事务回滚、快照、修复中心） | powercfg 子进程风暴拖垮系统稳定性 | 实测日志：41 分钟内 341 次 powercfg 子进程、单秒峰值 41 次（见"核心安全发现 5"） |
| 6 | 不是万能一键提速（README 定位） | 一键关闭全部安全防护、修改 BCD 启动项 | `GameOptimizer.Core.decompiled.cs` 操作目录（如 60563-60575 BCD 项、87549 Defender 实时防护） |
| 7 | 个人独立开发 + AI 辅助 | 部分实现达到专业级（签名校验、防 zip-slip、命令白名单） | `UpdateCheckService.cs`（`VerifyPackagePublisherTrust` 签名校验）；`UpdateApplyService.cs`（`IsSafeArchiveEntry` 防 zip-slip） |

**矛盾的本质**：当"显得专业安全"与"功能全、改动深"冲突时，让步的总是安全——提示降级为提醒、门禁降级为文案、恢复降级为"引导"。

## AI 痕迹与代码质量

### AI 痕迹（作者自认 + 代码强迹象）

- 作者 README 自述使用 AI 作为辅助工具
- 单一优化工具拆出 **1308 个类**；Services 目录 422 个文件、Views 148 个文件
- 每个优化项 = Operation + StatusProbe + RegistryTarget + Snapshot + ViewModel + 本地化条目，"六件套"反复复制
- `MainWindow.xaml.cs` 数千行巨型类，逻辑与 UI 混杂
- 专业度断层：更新签名校验（证书哈希 `FixedTimeEquals`、必需时间戳）写得规范，同库内电源查询每次新建子进程（性能灾难）——典型"关键函数来自成熟方案，整体拼装"
- 中英文案内嵌为代码字典而非独立资源文件

### 代码质量

| 维度 | 结论 | 证据 |
|---|---|---|
| 结构 | 差 | 视图巨型化、服务过度拆分、主程序与 Guard 逻辑重复 |
| 性能 | 差 | powercfg 每次查询拉子进程，实测单秒 41 次 |
| 可靠性 | 差 | 还原点兜底失败不阻断；BCD 修改无导出备份 |
| 安全实现 | 参差 | 签名/zip 防穿越/路径白名单专业；关防护/仅引导恢复激进 |
| 可维护性 | 差 | 巨型类 + 样板爆炸 + 字典式本地化 |
| 测试 | 未见证据 | 基于可见源码未发现测试工程/用例 |

## 与官方 README 的对照

### 作者已公开披露、与实测一致的部分（予以认可）

- 未购买受信任代码签名证书（与 NotSigned 实测一致）
- 自述"公开体验版，不是最终版本"，警告"若无法接受…重装系统的风险，请只使用检测功能"
- 明确备份边界："不是完整系统镜像，也不会备份个人文档、照片、视频或游戏存档"
- 提示仅从官方渠道下载并核对 SHA-256
- 声明 AI 辅助开发、由作者本人负责整合与测试
- 问题包仅在本地生成（与"无数据外传"结论一致）

### README 承诺与代码行为的出入

1. **还原点**：README 建议用户"先创建…系统还原点"作兜底 → 代码中创建失败只提醒、不阻断（软门禁）
2. **自动恢复范围**：README 称守护助手"只处理支持自动恢复的项目" → 软件仍提供"仅引导恢复"的安全缓解开关
3. **恢复方式**：README 称会告知恢复方式 → BCD 修改无导出备份，恢复依赖应用可运行；README 以"重装系统"收尾等于自认无离线恢复
4. **分批应用**：README 提醒"不要一次启用大量不了解的项目" → 应用内仍提供一键优化（该流程会跳过 Extreme 风险项，但"非安全操作"的还原点门禁依然非阻断）

## 方法与局限（请勿把"未发现"当成"不存在"）

- 评估基于反编译产物与本地运行观察，未做完整动态沙箱测试
- "未发现"类结论（BCD 无导出备份、无持久化、无外传）基于可见代码范围，`YuqiGuard.exe` 未完全审计
- `YuqiGuard.exe` 命名管道服务端是否校验客户端身份（潜在本地提权）**未证实**
- powercfg 数据来自一次运行实测，不代表所有环境
- "无限重启"与 BCD 修改为相关性判断，非因果证明
- 1308 个类仅抽样审查高危面

## 建议

**对用户**：只用检测功能；不要使用"关闭安全防护/修改启动配置/关闭缓解"类选项；修改前手动创建还原点并 `bcdedit /export` 备份。

**对作者**：① 移除或永久禁用关闭安全防护类功能；② BCD 修改前强制导出备份并提供离线恢复指引；③ 还原点创建失败必须阻断；④ 高危项提供一键自动回滚，取消"仅引导恢复"；⑤ 以系统 API 替代 powercfg 子进程轮询；⑥ 代码签名并建立安全厂商白名单流程。

**对平台**：上述问题修复前，建议将本软件标注为高风险工具。

---

## 我的感想

写这份报告的过程中，我最大的感受是：**这个软件的代码质量差到让我没有动力做完全分析。**

1308 个类、422 个服务文件、几千行的 MainWindow——不是"功能多所以代码多"，而是同一套逻辑被复制粘贴了几百遍。每个优化项都是一套"操作类 + 探针类 + 目标提供者 + 快照 + ViewModel + 文案"的六件套，像流水线一样堆出来的。我本来想逐类审查，但很快就发现：**高危问题根本不需要看完所有代码——它们就摆在最显眼的地方**（还原点失败不阻断、BCD 无备份、一键关防护）。与其花几天把每一行都读完，不如把这几条最致命的问题查实。这也是这份报告没有"完全分析"的原因：不是做不到，是性价比太低——对用户来说，知道"别碰哪些按钮"远比知道"第 1307 个类写了什么"重要。

另一个感受是"既要又要"：作者想要"专业、安全、负责任"的名声，又想要"功能全、一键化、改动深"的效果，于是文档里写满了风险提示，代码里却把风险提示降级成一句提醒。**文档诚实是加分项，但文档不能替代实现。** README 里写着"请先创建还原点"，代码里却写着"还原点创建失败，已作为风险提醒继续执行"——这两句话放在一起，就是对普通用户最大的误导。

我也客观说几句：作者确实做对了一些事——更新签名校验写得规范、ZIP 解压有防穿越、清理路径全写死、没有偷偷上传数据、README 主动承认未签名和体验版身份。这些说明作者不是恶意，只是**能力、测试资源和工程经验撑不起这个功能规模**。AI 帮他写出了 1308 个类的"量"，但"质"——哪些功能不该做、哪些开关不能给、哪些兜底必须强制——恰恰是 AI 给不了、需要人来判断的东西。

最后给普通用户一句实在话：**这类"系统优化工具"真正的优化空间，往往没有它制造的麻烦大。** 别为省几个启动项，把电脑的保命开关都关了。

---

## 附录 A：证据索引

- 还原点降级文案与调用代码：`GameOptimizer.Core.decompiled.cs` 98489-98491、98497；`YuqiEngine.decompiled.cs` 79525-79585、140443、148191
- BCD 操作与警告：`GameOptimizer.Core.decompiled.cs` 60563-60575、68299、70485-70553、28903
- 缓解仅引导恢复：`GameOptimizer.Core.decompiled.cs` 35370、28835、50103
- 关闭防护类操作：`GameOptimizer.Core.decompiled.cs` 87549 等；一键优化入口：7342
- powercfg 轮询：`GameOptimizer.Core.decompiled.cs` 84216、30108、30189、35401、52873
- 签名状态：`Get-AuthenticodeSignature`（YuqiEngine.exe / YuqiGuard.exe 均 NotSigned）
- 运行日志：`%LOCALAPPDATA%\YuqiEngine\Logs\yuqiengine-20260808.log`
- 官方 README：作者 GitHub 仓库 `README.md`

## 附录 B：反编译产物说明

- 评估使用的反编译产物：`GameOptimizer.Core.decompiled.cs`、`YuqiEngine.decompiled.cs`（由软件运行时内存 dump 提取的托管 DLL 反编译生成，为评估用临时产物）
- 文中行号均为上述反编译产物中的行号，供复核；行号可能随反编译工具版本略有差异，结论不依赖具体行号
