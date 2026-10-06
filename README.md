# codebuddy-cli-delegation

![License: MIT-0](https://img.shields.io/badge/license-MIT--0-blue)
![Skill](https://img.shields.io/badge/type-agent%20skill-8957e5)

**English summary** — A reusable Agent Skill for delegating large batches of coding work to a
standalone **headless CodeBuddy CLI** agent. It covers the clean-environment launcher (which
strips host-injected environment variables that otherwise cause silent zero-byte hangs),
instruction budgeting, parallelism admission rules (4-way isolation + disjoint write sets),
per-stage parameter allocation, and independent audit & acceptance through adversarial probes
and mutation controls — never trusting the agent's own report. All 21 supported models and every
other claim are empirically verified against **CodeBuddy CLI 2.156.0**.

*The skill body and the rest of this README are written in Chinese, the primary audience for
CodeBuddy CLI. This summary exists so non-Chinese visitors can judge relevance in 30 seconds.*

---

> 把大批量编码任务派给「**独立无头 CodeBuddy CLI**」执行的标准流程 —— 你只负责
> **写清指令 → 控预算 → 收产物 → 独立验收**。

**它解决什么**：前台同步做会超时 / 上下文会被撑爆 / 需要并行多批次的大活。
**它不解决什么**：需要人来拍板的决策、需要访问宿主私有状态的事、一次性的小改动。

这是一个 **Agent Skill**（技能包），核心是一份 866 行的 `SKILL.md` 加一个跨平台启动器脚本。
全部结论基于 **CodeBuddy CLI 2.156.0**（2026-10-06）实测，不含任何未验证的推测。

---

## ⚠️ 适用范围：这是 **CodeBuddy CLI 专用**技能

本技能的全部实测结论**只对 CodeBuddy CLI 成立**，包括：干净环境启动器要剥离的变量名
（`CODEBUDDY_*` / `ACC_PRODUCT_CONFIG_*` / `SERVER__*`）、`--setting-sources project`、
`--permission-mode bypassPermissions`、`--effort` 六档档位与厂商映射、21 个可用模型清单、
`--agents` 的 JSON schema、`-c` 恢复会话、`--tools` 白名单、会话日志路径 `~/.codebuddy/projects/`。

**换用别的 CLI（Codex / Gemini CLI / Claude Code 等）时不要照搬**：那些 CLI 的注入变量清单、
参数名、会话日志结构、权限机制都不同，必须各自重新实测。

其中**真正与 CLI 无关、可直接复用的方法论**（换 CLI 后仍成立）：
§二（并行准入与四隔离）、§3.5（派发前契约核对六动作）、§7.3–7.7（对抗探针与变异对照）、
§零铁律 3（不采信自述）。

---

## 安装

技能就是一个目录，拷进对应客户端的 skills 目录即可：

```bash
# CodeBuddy CLI
cp -r codebuddy-cli-delegation ~/.codebuddy/skills/

# WorkBuddy
cp -r codebuddy-cli-delegation ~/.workbuddy/skills/
```

Windows 下目标路径形如 `C:\Users\<你>\.codebuddy\skills\codebuddy-cli-delegation\`。
装好后用 Skill 工具按名 `codebuddy-cli-delegation` 加载（或直接说「用 CLI 派活」触发）。

---

## 目录结构

```
codebuddy-cli-delegation/
├── SKILL.md                      # 技能主体：十条正文 + 四条铁律 + 检查清单
├── scripts/
│   └── codebuddy-cli.js          # 干净环境启动器（单文件，Windows/macOS/Linux 通用）
└── skills-bootstrap.template.md  # 注入 CLI system prompt 的技能约定模板
```

`scripts/codebuddy-cli.js` 之所以用 **Node 而不是 shell**：跨平台单文件、装了 CodeBuddy CLI
就必然有 Node（它自己就是 Node 应用）、且不受 shell 方言与 UTF-8 BOM 这类编码陷阱影响
（前几轮在 `.sh` / `.ps1` 上踩过坑，见 SKILL.md「七轮翻车复盘」）。

---

## 四条铁律

1. **绝不直接启动裸 CLI** —— 宿主会把环境变量注入子进程，轻则跑错配置，重则**永久挂死**
   （0 字节输出、300s 无反应）。必须走干净环境启动器。
2. **默认串行；并行是「支持」的**，但要不要并行取决于「共享可变资源是否隔离」，与端口无关。
   要并行就必须先做**四隔离**（目录+git / 依赖 / 数据库 / 端口）并保证**写集不相交**。
3. **不采信它的自述** —— 它说「已通过 / 已标记 / 已加载」都要自己复跑一遍才能算完成。
4. **模型由派发方显式指定**；技能只负责在给定模型下分配参数，**永不替派发方选模型**。

---

## SKILL.md 章节导览

| 节 | 内容 |
|---|---|
| 零 | 四条铁律 |
| 一 | 为什么需要干净环境启动器 · 启动器实现 · 七轮翻车复盘 · **21 个模型可用性事实表** |
| 二 | 并行与串行：准入、四隔离、分工与合并审计纪律 |
| 三 | **写指令**（成败关键）：提技能 ≠ 加载技能、喂结论、硬性预算、失败标准动作、派发前契约核对六动作 |
| 四 | 无头模式下的用户级 hooks（优先级最高的坑） |
| 五 | 把技能约定注入 system prompt（SDD / TDD / ponytail） |
| 五之二 | 模型侧不可控项：哪些旋钮根本不存在 |
| 五之三 | 子代理分档 + 三条会让审计失真的行为事实 |
| 五之四 | 运行中途换档（`-c` 恢复会话 + 改 `--effort`） |
| 六 | 权限模式与运行 |
| 七 | **审计与验收**：会话记录位置、独立验收、**对抗性探针 + 变异对照**、三类自述的核实、预埋检查项、抽查映射表 |
| 八 | 失败模式速查 |
| 九 | 交付时的交代清单 |
| 十 | 检查清单（可复制走） |

---

## 设计取舍（写给改它的人）

这本技能有几处**刻意的「不写」**，别好心补回去：

- **不写「阶段 → 推荐模型」速查表**。那会让下一次派活悄悄滑向「技能替人做技术选型」，
  而选型责任与后果都在派发方。想按阶段换模型，必须由派发方显式拆成多批或多子代理。
- **不写「端口按工作区派生，第二个实例必 EADDRINUSE」**。该结论已于 2026-10-01 实测推翻。
- **不靠调 temperature 收敛**。那不是可控旋钮；要「保守」就翻译成改动范围白名单 + 工具白名单 + 禁止项。

---

## 实测基线

| 项 | 值 |
|---|---|
| CodeBuddy CLI | 2.156.0 |
| 验证日期 | 2026-10-06 |
| 覆盖平台 | Windows（Git Bash）、macOS、Linux |

> 技能里所有带 ⚠️ 的结论都对应一次具体的翻车记录，不是理论推导。

---

## 许可

**MIT-0（MIT No Attribution）** —— 全文见 [LICENSE](LICENSE)。

意即：可自由使用、修改、分发、商用，**连版权声明与许可声明都不需要保留**，
也不提供任何担保。全文与 SPDX 官方文本一致（仅替换了版权行）。
