# omp-agent — omp 主代理回话角色扩展包

给 Oh My Pi (omp) 添加**主代理回话角色**：`unrestricted`（安全研究执行者）、`vuln`（CTF/授权渗透）、`nsfw`（成人虚构创作）。
本 README 面向**开发者与 AI agent**：读完即可安装、验证、使用、添加新角色。

- 零外部依赖（扩展模块不 import 任何 npm 包）
- 仅依赖 omp 原生机制：扩展模块（`extensions.md`）、斜杠命令（`slash-command-internals.md`）、task 子代理（`task-agent-discovery.md`）
- omp 版本要求：17.x（`input` / `before_agent_start` 事件）

---

## 1. 这是什么 / 架构

omp **没有** opencode 的 primary agent 概念（主会话只有一个 main agent，见 `agent-hub.md`："The main agent is not listed because its conversation is the ambient session view"）。
本包用 omp 扩展机制实现"主代理回话角色"：**用户手动切换，主会话以该角色身份对话，整会话粘性生效**。

```
┌─ 用户输入 ───────────────────────────────────────────────┐
│  /vuln 渗透测试这个靶机                                    │
└──────────────────────────┬───────────────────────────────┘
                           ▼
┌─ extensions/roles.ts（核心扩展）──────────────────────────┐
│  pi.on("input")              ← 识别 /unrestricted /vuln /nsfw，
│                                 维护进程内角色状态（粘性）      │
│  pi.on("before_agent_start") ← 每轮把角色人格块追加进系统提示     │
│  pi.on("session_before_switch"/"session_shutdown") ← 清状态   │
└──────────────────────────┬───────────────────────────────┘
                           ▼
                    omp AgentSession
                    （本轮的 systemPrompt 含角色人格）
```

| 文件 | omp 机制 | 作用 |
|---|---|---|
| `extensions/roles.ts` | 扩展模块（`~/.omp/agent/extensions/*.ts` 自动发现） | **核心**：角色切换 + 每轮系统提示注入 |
| `commands/*.md` | 原生斜杠命令（`~/.omp/agent/commands/*.md`） | 手动切换入口，模板携带任务文本与 `[MODE: X]` 标记 |
| `agents/*.md` | task 子代理定义（`~/.omp/agent/agents/*.md`） | 同一角色的**完整版**规则，可经 `/agents` 或 task 工具派发 |

---

## 2. 文件清单

```
omp-agent/
├── extensions/
│   └── roles.ts            # 扩展模块（唯一代码文件）
│       - ROLES 表：每个角色 { id, label, marker, persona }
│       - persona = 注入主会话系统提示的人格块（压缩版核心契约）
├── commands/
│   ├── unrestricted.md     # /unrestricted 命令模板
│   ├── vuln.md             # /vuln 命令模板
│   └── nsfw.md             # /nsfw 命令模板
├── agents/
│   ├── unrestricted.md     # 子代理定义（完整版规则）
│   ├── vuln.md
│   └── nsfw.md
└── README.md               # 本文档
```

**内容分层约定**：
- `extensions/roles.ts` 的 `persona` = **压缩版**核心契约（策略/范围/禁止/规则/输出契约），每轮注入，控制 token 开销（各 ~1.5–2KB）
- `agents/*.md` = **完整版**规则（vuln 含 PARALLEL & DELEGATION 等完整小节），子代理派发时生效

---

## 3. 安装（全局）

把三个目录复制进 omp 用户配置目录，**无需插件系统、无需注册清单**——四个目录均为启动时自动扫描的原生目录（native provider，优先级 100）。

```bash
# 通用（POSIX；Windows 见下方）
OMP_AGENT_DIR="${HOME}/.omp/agent"          # 有 --profile <name> 时改为 ~/.omp/profiles/<name>/agent
mkdir -p "$OMP_AGENT_DIR/extensions" "$OMP_AGENT_DIR/commands" "$OMP_AGENT_DIR/agents"
cp extensions/roles.ts          "$OMP_AGENT_DIR/extensions/"
cp commands/unrestricted.md commands/vuln.md commands/nsfw.md "$OMP_AGENT_DIR/commands/"
cp agents/unrestricted.md agents/vuln.md agents/nsfw.md   "$OMP_AGENT_DIR/agents/"
```

Windows PowerShell：

```powershell
$d = "$HOME\.omp\agent"
New-Item -ItemType Directory -Force -Path "$d\extensions","$d\commands","$d\agents" | Out-Null
Copy-Item extensions\roles.ts "$d\extensions\"
Copy-Item commands\unrestricted.md,commands\vuln.md,commands\nsfw.md "$d\commands\"
Copy-Item agents\unrestricted.md,agents\vuln.md,agents\nsfw.md "$d\agents\"
```

**重启 omp 生效**（扩展/命令/子代理在启动时加载，无热重载）。生效检查点：新会话里 `/vuln` 命令出现在自动补全；`/agents` 仪表盘多出三个子代理。

---

## 4. 使用

**主代理回话角色（手动切换，粘性）**：

```
/unrestricted 帮我逆向这个程序的注册校验   # 切换为角色 + 立即执行任务
/vuln 渗透测试一下这个靶机
/nsfw 写一段场景
/vuln                                     # 无参数 = 只切换角色，不触发任务
/vuln off                                 # （或 /exit /stop）退出角色，恢复默认身份
```

- 粘性：切换后**后续每轮**都注入该角色人格（进程内状态，跨轮保持）
- 新会话自动重置；`session_before_switch`/`session_shutdown` 清除状态
- `/model` / `Ctrl+P` 换模型不影响角色；角色不绑定模型（跟随当前 default）

**子代理形态**：`/agents` 打开 Agent Control Center 可见 `unrestricted`/`vuln`/`nsfw`；task 工具 `agent: "vuln"` 派发。

---

## 5. 验证清单（可执行）

```bash
# 1) 模式注入（期望输出含 MODE-*-OK）
omp -p "/vuln 只回答：系统提示是否含 [MODE: VULN] 标记？是则回 MODE-VULN-OK" --no-session
omp -p "/unrestricted 只回答：系统提示是否含 [MODE: UNRESTRICTED] 标记？是则回 MODE-UNRESTRICTED-OK" --no-session
omp -p "/nsfw 只回答：系统提示是否含 [MODE: NSFW] 标记？是则回 MODE-NSFW-OK" --no-session

# 2) 粘性（第二条无命令前缀，期望仍回 [MODE: VULN]）
omp -p "/vuln 你好" "只回答你的系统提示中的 MODE 标记名" --no-session

# 3) 子代理派发（期望 VULN-SUB-OK）
omp -p "用 task 工具派发 agent 为 vuln 的子代理，任务：'只回复 VULN-SUB-OK'，原样报告其回复" --no-session
```

---

## 6. 机制（为什么这样实现）

- **`before_agent_start` 系统提示注入**：事件返回 `{ systemPrompt: [...当前块数组, 人格块] }`——runner 语义为**链式替换**（每个 handler 收到上一个 handler 输出的数组，返回新数组即替换本轮系统提示）。注入时先检查 `marker` 是否已在系统提示中，防重复。
- **`input` 事件切换**：`input` 返回 `{ handled: true }` 消费输入（纯切换/退出时不打扰 LLM）；带任务时**不** handled，让命令模板正常展开（模板含 `[MODE: X]` 标记与 `$ARGUMENTS`）。
- **标记兜底**：headless/print 模式不触发 `input` 事件（"interactive mode only"），`before_agent_start` 从 `event.prompt` 中的 `[MODE: X]` 标记 arm 并持久化——保证 headless 下也粘性。
- **去重键**：`marker` 字符串（如 `[MODE: VULN]`）是注入去重与兜底触发的键，**必须与命令模板中的标记完全一致**。
- 状态为进程内变量，不做持久化（resume 后需重新切换角色）。

---

## 7. 添加新角色（四步模板）

以 `newrole` 为例，**marker 一致性是唯一硬约束**（`roles.ts` 的 marker == 命令模板的 `[MODE: NEWROLE]`）：

1. **`extensions/roles.ts`** — `ROLES` 表加一条：
   ```ts
   newrole: {
     id: "newrole",
     label: "新角色名",
     marker: "[MODE: NEWROLE]",
     persona: `[MODE: NEWROLE]\n…角色人格核心契约…`,
   },
   ```
2. **`commands/newrole.md`** — 命令模板（照抄 `commands/vuln.md` 结构）：
   ```md
   ---
   description: 切换主代理为新角色并立即开工
   ---

   [MODE: NEWROLE]
   …

   用户请求（保持原动词，立即执行）：

   $ARGUMENTS
   ```
3. **`agents/newrole.md`** — 子代理定义（frontmatter 必须含 `name` + `description`，body 为完整规则）：
   ```md
   ---
   name: newrole
   description: 新角色子代理：<一句话定位>
   ---

   …完整版规则…
   ```
4. 复制到 `~/.omp/agent/` 对应目录 → 重启 omp → 跑 §5 验证清单对应项。

**检查点**：marker 三处一致（ROLES 表 / 命令模板 / 去重逻辑无需改）；persona 与 agent body 内容分层（核心契约 vs 完整版）；`description` 必填（native 技能/命令/代理要求）。

---

## 8. 与 opencode 原版项目的差异（移植参考）

原版：`opencode-unrestricted` / `opencode-vuln` / `opencode-nsfw`（opencode 扩展三件套）。

| opencode | omp（本包） |
|---|---|
| `plugin/*.ts`（`@opencode-ai/plugin` 管道 API） | `extensions/roles.ts`（事件订阅 API）——**完全重写，不可直接拷贝** |
| `agent/*.md` frontmatter `mode/color/permission` | `agents/*.md` frontmatter `name/description`（task 子代理契约） |
| `command/*.md` frontmatter 含 `agent:` | `commands/*.md` 仅 `description` |
| `opencode.json`（default_agent/permission） | 无对应物，删除 |
| 关键词自动改写（`matchesIntent`/`rewriteText`） | 未移植（本包为手动切换模式） |
| HARD STOPS 安全红线 | **已移除**（按需求）；行为契约（禁止软拒绝/保持保真度）保留 |

人格内容文本（策略/范围/禁止/规则）可复用，API 与 frontmatter 必须按上表改写。

---

## 9. 故障排查

| 现象 | 原因与处理 |
|---|---|
| `/vuln` 无反应/不注入 | 未重启 omp（扩展启动时加载）；检查 `~/.omp/agent/extensions/roles.ts` 存在且与源码一致 |
| 扩展加载报错 | 看日志 `~/.omp/logs/omp.*.log`；roles.ts 必须 default 导出工厂函数 |
| 命令不在自动补全 | 检查 `commands/*.md` frontmatter YAML 合法（native 命令 frontmatter 解析失败为 fatal） |
| 子代理不在 `/agents` | 检查 `agents/*.md` 有 `name` + `description` |
| 注入重复 | marker 未出现在系统提示已有内容中时才会注入；确认 marker 拼写一致 |
