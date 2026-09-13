# 调试与排障指南（clash_rules × Hako Clash）

> 目的：改规则 / 改配置时一次做对，别再踩三个坑——「改了但客户端不生效」「手改缓存把规则集搞空」「换端就连不上」。
> 适用客户端：Hako Clash（macOS / iOS / iPadOS，mihomo 内核）；同一份配置亦用于任意 mihomo 客户端（Android，如 CMFA）。
> **同一份 `config.yaml` 三端通用**（纯 mihomo 标准 YAML，不含 Hako 专属字段）。本文只描述机制与操作，不含订阅链接、密钥、节点或账号。

---

## 0. TL;DR：改规则的正确闭环

1. 改本仓库的 `*.list` 文件（唯一源头，见 §8 落点速查）→ **commit & push**（§10：报 SSL 错时先复核，别只看 stderr）。
2. 客户端 → 「配置 Profiles」→ 当前订阅 → **刷新全部 provider**（或只刷新改动的那一条）。
   **1.0.8 起不再需要「删除 profile → 重新添加」**（见 §2、§6）。
3. **VPN 开关关一次再开一次** → 若节点选择被重置则**重选** → **按 §4 验证**。

> ⚠️ 「**刷新 provider**」与「**更新 / 同步订阅**」是**两个不同的动作**：后者只重拉 config 文本，**会跳过 provider**（`x-hako-side-update-safe: true`）。**改了规则列表，必须用前者。**

---

## 1. 架构与当前基线

### 1.1 谁是谁的源头

| 层 | 位置 | 性质 |
|---|---|---|
| **策略源头** | 本仓库的 `*.list` 文件 | 手工维护，**唯一真相** |
| 主配置 | 私有仓库 `clash-config/config.yaml` | 含订阅 token，靠 PAT 导入 URL |
| 客户端规则数据 | provider 本地缓存 | **`type: file`，带 sha256 校验** |

**关键事实（实测）**：**内核运行时**把**全部** rule-provider 都当 **`type: file`** 加载，**没有一条是 `type: http`** —— **内核自己不联网去拉规则仓库**。远程抓取改由 **App 层**负责：App 在应用内下载 HTTPS provider → 校验 → 落成本地文件后交给内核（§2.1）。

### 1.2 当前配置基线（2026-09-13）

- **客户端**：Hako Clash **1.0.8**（macOS / 内核 mihomo `1.19.30`）。**自该版本起支持应用内刷新 provider（整体或单条）**，规则列表更新**不再需要删除 profile 重加**（机制见 §2）。
- **29 个 rule-provider**，URL 全部走 **GitHub raw**（`raw.githubusercontent.com/wyk111wyk/clash_rules/main/...`），不依赖任何第三方 CDN。
- **广告表** = `AdBlockCore`（`AdRules/AdBlock_core.list`，**2944 条**精简强语义广告/追踪条目）；`AdBlock_reject.list`（**77748 条**全量）**未被任何配置引用**，仅作筛选素材保留。
- **已删 5 张与 `GEOIP,CN` 冗余的 DIRECT 表**：`WeChat_dom/cls`、`ChinaASN_cls`、`SteamCN_dom`、`SpeedtestCN_dom`。
- **`geox-url` 只留实际用到的 2 条**：`mmdb` + `geoip`（`asn` / `geosite` 已删）。
- **TUN 协议栈固定 `gvisor`**（三端唯一兼容，见 §11）。
- 分组：`proxy-providers` 淘气兔168G（`interval` 48h 自动换节点）；`rule-provider interval` 统一 **90 天**；12 个策略组（1 个 hidden）。

---

## 2. 核心机制：为什么「push 了不生效」

客户端解析订阅时，把每个 rule-provider **下载成本地文件**（`<Hako 容器>/working/store/profiles/<profileID>/revisions/<revID>/providers/provider-*.txt`），并在同目录 `manifest.json` 记录 **sha256 + bytes**；内核加载时**只读本地文件**并做 **sha256 校验**。这些 provider 被打了 **`x-hako-side-update-safe: true`** 标记 → 「同步 / 更新订阅」**会跳过**重新下载。

> **最重要的一条**：**往 GitHub push 成功 ≠ 客户端生效。** 中间隔着本地 file 缓存；必须在客户端**显式刷新 provider**。

### 2.1 2026-09-13 变更：支持应用内刷新 provider（Hako 1.0.8）

新版本把 provider 的抓取与落地交给 App 自己做。本地化文案原文：

- 「Clash 会在**应用内下载 HTTPS provider、校验后发布一份不可变的本地副本**，再交给内核加载。」
- 「Provider 无法下载、解密或校验，**已保留已发布的修订**。」

于是多出第三个动作 —— **刷新 provider**（`刷新全部 provider` / 单条 provider 刷新，另有 `同步 provider`）。它按 provider 自身的 URL 重新下载 → 校验 → **发布新的 revision**（`providers/` 下写新文件 + 新 sha256）。**这取代了旧版唯一的办法「删除 profile → 从同一链接重新添加」。**

两个要点：

- **刷新失败不等于规则没了**：失败时保留**上一份已发布的 revision**（不会像手改缓存那样 `ruleCount` 归零）。副作用是「点了刷新却仍在跑旧规则」也可能发生 → 用 §4① 对条数，别凭感觉。
- **`x-hako-side-update-safe` 标记仍在**（`config.resolved.yaml` 里每条 provider 都带），所以「更新 / 同步订阅」**依旧跳过 provider**，两者不可互相替代（见 §6）。

---

## 3. 三条绝对不要做的事（血泪教训）

1. ❌ **绝不要直接编辑** 缓存里的 `provider-*.txt` → 内容与 manifest 的 sha256 对不上 → 该规则集判定损坏 → **`ruleCount` 归零（规则全失效）**，且 **API 无法恢复**，只能在 GUI 里**刷新该 provider** 重建（§2.1）。
2. ❌ **不要改 `manifest.json` 的 sha256** 绕过校验 → 同为 hack，上层可能还有校验，风险更高。
3. ❌ **不要指望 Clash API 刷新 file 类型 provider** → `PUT /providers/rules/<name>` 返回 **HTTP 204** 是**假成功**：`file` 类型没有 url 可重拉，只会重读本地文件；若文件已被手改，反而直接把规则集清空。**刷新要走 App GUI，不是 API。**

> 正确做法只有一个：**让 App 自己重新下载 provider**（§0 第 2 步 / §2.1）。

---

## 4. 验证改动是否真的生效

**不要只看面板条数**（可能是 UI 缓存），以后端为准。

**① 看后端实际条数**（控制端口 `127.0.0.1:9090`，本机无 secret；该版本**单条 provider 不支持 GET**，只能拉列表）：

```bash
curl -s --compressed http://127.0.0.1:9090/providers/rules \
  | python3 -c 'import sys,json;[print(f"{k:24} ruleCount={v.get(\"ruleCount\")} vehicle={\"file\" if v.get(\"vehicle\") is None else \"http\"}") for k,v in json.load(sys.stdin)["providers"].items()]'
```

- 把某条的 **`ruleCount`** 与 **GitHub 上该 list 的有效行数**对比，一致才算生效。
- ⚠️ 条数字段是 **`ruleCount`**（误用 `rules` 数组会全显示 0，虚惊一场）。

**② 看「新连接」的真实命中（最终判据）**：在 **Connections** 页新开请求，看域名实际命中的**策略组**。
> **已有长连接不会迁移到新路线**——刷新网页可能复用旧连接，要**断开重连 / 新开**才看得到。不要只判断「按钮点过了」。

---

## 5. Clash 控制端口（API）能力边界（实测）

| 端点 | 可用性 | 说明 |
|---|---|---|
| `GET /providers/rules` | ✅ | 列出所有 provider 的 `ruleCount` / `vehicle` |
| `PUT /providers/rules/<name>` | ⚠️ | 返回 204，但对 `type: file` 是**假成功** |
| `GET /providers/rules/<name>` | ❌ | 单条端点**不支持 GET** |
| `POST /restart` | ❌ 404 | 不支持 |
| `PUT /configs` | ❌ 405 | 不支持 |

**结论**：这套 API 只能**读诊断信息**，不能修复/刷新 file 类型 provider —— **刷新的唯一入口是 App GUI（§2.1）**。

---

## 6. 别用错动作：「刷新 provider」≠「更新 / 同步订阅」

| 动作 | 实际行为 | 什么时候用 |
|---|---|---|
| **刷新 provider**（全部 / 单条） | 按 provider 的 URL 重新下载 → 校验 → **发布新 revision** | **改了本仓库 `*.list` 时用这个**（§0 第 2 步） |
| 更新 / 同步订阅 | 重拉 **config 文本**；因 `x-hako-side-update-safe` **跳过 provider** | 只改了 config（策略组、规则顺序）时用 |
| 开关 VPN 开关 | 启停隧道 + 重载本地已解析配置与 provider 缓存 | **不重拉任何东西**，不能当刷新用 |
| 备份恢复 | 只还原 config 文本，不重建 provider 缓存 | 同上，**不能**当刷新用 |

为什么「更新订阅」会跳过 provider：① 每个 provider 有自己的 `interval`（本仓库统一 **90 天** = `7776000` 秒），客户端不自动拉；② `x-hako-side-update-safe: true` 标记让 profile 更新时**不动** provider 缓存。

> **历史（已废）**：1.0.8 之前**没有** provider 刷新入口，唯一办法是「**删除 profile → 从同一链接重新添加**」，代价是 profile 内手动选的节点被重置。**现在不需要了。**

---

## 7. 改动后自检清单

- [ ] list 行格式正确（domain 行为：`+.example.com` / `example.com`；classical 行为：`DOMAIN-SUFFIX,example.com`）
- [ ] 本仓库已 commit（`git log --oneline -3`），必要时已 push，且 `git status` 干净
- [ ] GitHub 侧可直连验证条数：`curl -s https://raw.githubusercontent.com/<owner>/<repo>/main/<path> | grep -vcE '^\s*#|^\s*$'`
- [ ] 客户端里**刷新 provider**（改了哪条就刷哪条，或「刷新全部 provider」；不是只点「更新订阅」）→ VPN 开关关/开各一次
- [ ] 后端 `ruleCount` 与 GitHub 条数**一致**（§4①）
- [ ] 被重置的节点已**重选**；新开连接在 Connections 命中**正确的策略组**（§4②）

---

## 8. 规则落点速查（改某个域名该动哪个文件）

| 想要的效果 | 改哪个文件 | 备注 |
|---|---|---|
| 某域名**直连**（不走代理） | `Custom/custom_direct.list` | 映射为 `RULE-SET,CustomDirect,DIRECT`，**排在很前**，优先级高 |
| 境外 **AI / LLM 工具**走 `🤖境外LLM`（JP/SG/US） | `Policy/AI/llm_ai_domain.list` | 规避「香港节点被境外 LLM 拒绝」 |
| 广告/跟踪**拦截** | `AdRules/AdBlock_core.list` | 排在很前，先于其它规则命中 |
| 广告拦截**白名单** | `AdRules/AdBlock_direct.list` | 需放行的域加这里 |
| 其它分类 | `Policy/{AI,Apple,Game,Tool,CN,Global}/` | 按业务分目录 |

**优先级口径**：策略由引用方 config 的 `RULE-SET` 行决定，**按 rules 段顺序自上而下匹配、先命中先决**；list 只装域名、不带策略。「想直连」→ 放进排在前面的 DIRECT 列表。

**两个容易漏的形态**：

- **裸 IP 连接**：工具自行解析 DNS 后直连 IP（如 `172.217.117.4:443`），**没有域名**，域名规则看不到 → 必须由 `Policy/AI/llm_ai_classical.list` 的 `IP-CIDR,...,no-resolve` 兜底。
- **Google 自有顶级域 `.goog`**：`googleapis.com` 通配**覆盖不到** `.goog`（如 `antigravity-unleash.goog`）。查缺口见 §12。

---

## 9. `geox-url` 为什么还是第三方 CDN（不是漏改）

`geox-url`（`mmdb` / `geoip`）指向**内核启动时下载的公共地理数据二进制**，来源是第三方数据集仓库，**与本规则仓库无关**。保留第三方 CDN 是**有意保留**：生命周期与规则列表独立；`geo-auto-update: false` → 只在**文件缺失或手动更新时**下载一次，之后走本地缓存；改 raw 对**大二进制文件**直连可达性更差、**无收益**。已删掉用不到的 `asn` / `geosite` 两条，只留 `mmdb` + `geoip`，**不建议再动**。

---

## 10. 本机网络与 push 注意事项

- **网络现状（2026-09-12 起已稳定）**：GitHub 正常 `commit` / `push` 即可，**不再需要**早期「push 只试一次、失败即停手」的临时约定。
- 若偶发 `SSL_ERROR_SYSCALL` / 连不上：不反复重试、不改 remote、不换 SSH、不加跳过校验的环境变量；**先按下面一条复核是否其实已经推上去了**。
- ⚠️ **「push 失败」可能是误报**：报 SSL 错时**先复核** `git status` 与 `git log origin/main..HEAD`（为空即为已推送成功），**别只看 stderr**。
- 判定「网络问题」还是「仓库问题」：`curl -sS -o /dev/null -w "%{http_code}" https://www.baidu.com` 通（200）而 `https://api.github.com` 不通（000）→ 本机代理/上游故障，与仓库无关。只读获取仓库信息可走网页抓取通道。

---

## 11. 跨端通联：iOS / Android「无法连接」怎么查

同一份 `config.yaml` 三端通用，但**客户端运行时环境不同**，两个已确认的坑：

### 11.1 TUN 协议栈：iOS 只能用 `gvisor`（2026-09-12 实测修复）

- **必须 `tun.stack: gvisor`**。iOS Network Extension **不支持 `system` 栈**（TCP 在内核态建 tun）；`mixed` 包含 `system` 分支 → 启动失败：
  > `[bindif] no up interface carries fdfe:dcba:9876::1` → `Start TUN listening error` → 内核关闭 → 客户端显示**「无法连接」**。
- `gvisor` 是**纯用户态**栈，macOS / iOS / Android 三端通用且最兼容。
- **症状对照**：iOS 一开 VPN 就报「无法连接 / 内存不足」且节点为空 → 先查 `tun.stack` 是不是 `gvisor`，**不要**先怀疑设备内存。

### 11.2 规则/订阅源可达性（换网络就通 / 不通的根因）

- provider 与订阅都要联网拉。**某条网络到规则源（现走 `raw.githubusercontent.com`）不可达时，配置初始化失败**，客户端常给出**误导性报错**（如「内存不足」「无法连接」），且**节点列表为空**。
- **判定方法**：手机 Safari 直接打开订阅链接与规则源 URL，看是否返回内容；再换一条网络（如蜂窝 ↔ 热点）对比。
- **当前状态与边界**：已从 jsdelivr 改为 **GitHub raw**（便于统一在 GitHub 维护、推送后手动刷新即生效，无 CDN 缓存要 purge）。但 `raw.githubusercontent.com` 在**大陆部分网络同样会被墙**——若某端再现「规则加载失败 / 节点为空」，根因仍是它。**彻底解法**：把规则列表自托管到自己的服务器，provider URL 换成自有域名。

### 11.3 三端差异速查

| 项 | macOS | iOS / iPadOS | Android |
|---|---|---|---|
| TUN | 任意栈 | **只能 gvisor** | 任意栈，走 VpnService 授权 |
| 内存约束 | 宽松 | Network Extension 有硬上限，配置越重越危险 | 宽松 |
| 订阅 UA | 配置内已锁 `ClashMetaForAndroid/2.11.4` | 同 | 天然匹配 |

---

## 12. 如何系统性查规则缺口（与官方 geosite 比对）

不要靠记忆猜缺哪些域名。用 `MetaCubeX/meta-rules-dat`（内核 geoip 数据的同一来源）的官方分类做**后缀语义**比对——`+.googleapis.com` 这类通配已覆盖其全部子域，直接字符串比分会误报一堆「缺失」：

```bash
curl -s https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/category-ai-!cn.yaml -o /tmp/ai.yaml   # 境外 AI 工具总集
curl -s https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/google-gemini.yaml -o /tmp/gem.yaml     # Google 系 AI
# 再用 Python 做后缀语义比对，只算「真正未覆盖」的；加之前先做冲突检查（别被前面的 AdBlock 命中）
```

**IP 维度**的权威来源是 Google 官方公开段：`https://www.gstatic.com/ipranges/goog.json`（自有服务）/ `cloud.json`（GCP 客户段）。只取 `goog.json` 核心服务段即可，避免误伤托管在 GCP 上的其它服务。

常用分类：`category-ai-!cn`（境外 AI 工具）、`google-gemini`、`openai`、`anthropic`、`category-ai-chat-!cn`。

---

## 13. 路由「正确」但连接被掐断（EOF / 随机超时）怎么查

典型现象：AI 工具（Antigravity / Codex / Claude Code 等）报 `request failed ... EOF`，或连接随机失败；**但规则本身是对的**。此时不要去改规则：

**① 先证明路由是对的（抓内核日志）**

```bash
B=http://127.0.0.1:9090
curl -s --compressed -N "$B/logs?level=debug" --max-time 20 > /tmp/l.log 2>&1 &
grep -a "match RuleSet" /tmp/l.log | tail -20   # 形如 ... match RuleSet(LLM_AI_dom) using 🤖境外LLM[🇺🇸 US 02]
```

**② 区分「规则问题」还是「链路/节点问题」** —— 经代理端口做 A/B（同一节点下，对比「Google 系」与「非 Google 系」目标，一个通一个不通即链路问题）：

```bash
curl -sS -o /dev/null -x http://127.0.0.1:7890 --max-time 8 -w "%{http_code}\n" https://daily-cloudcode-pa.googleapis.com/
curl -sS -o /dev/null -x http://127.0.0.1:7890 --max-time 8 -w "%{http_code}\n" https://openrouter.ai/
```

**③ 节点 A/B**：`PUT /proxies/<组名(URL 编码，emoji 亦然)> {"name":"<子组>"}` 切换 select 子组，重复 ② 若干次统计成功率，测完还原。

**实战结论（2026-09 实测）**：同一机场不同出口 IP 对 Google 可用性差异极大 —— 目标 `daily-cloudcode-pa.googleapis.com`，累计成功率 US 3/10→12/12 剧烈波动、JP 2/6、**SG 43/46 最稳**、`🛰家宽兜底` **0/8（该组节点是死的）**。**遇到 Google 系间歇性 EOF，第一动作是换节点，不是改规则。**

**两个坑**：

- **无代理的 `curl` 仍会被 TUN 捕获**（日志源地址 `198.18.0.1`），「直连测试成功」是假象。
- **TUN + `fake-ip`** 是社区已知会影响 `*.googleapis.com`（Antigravity 登录/流式 EOF）的组合。缓解：把相关域加进 `dns.fake-ip-filter`，或把 `dns.enhanced-mode` 改为 `redir-host`（属 config.yaml `dns:` 段，非规则；国内 nameserver 解析 Google 域需配 `nameserver-policy` 才稳）。

---

## 附录：快照命令（日常体检通用）

```bash
# 打印两个常改 provider 的条数 + 缓存文件 mtime（只读）
GC="$HOME/Library/Group Containers/<Hako 容器目录>"
AC="$GC/working/active/config.yaml"
for n in LLM_AI_dom CustomDirect; do
  p=$(grep -A5 "^    $n:" "$AC" | grep "path:" | sed 's/.*path:[[:space:]]*//')
  printf '%-14s 有效行=%s  mtime=%s\n' "$n" \
    "$(grep -vE '^[[:space:]]*#|^[[:space:]]*$' "$p" | wc -l | tr -d ' ')" \
    "$(stat -f '%Sm' "$p")"
done
```

> Hako 容器目录名以实际安装为准（`ls "$HOME/Library/Group Containers" | grep -i hako`）；`revision` 的 UUID 会随**刷新 provider**（以及删除重加）变化，属正常；旧 revision 目录会保留，provider 文件按内容去重（`ls -l` 可见硬链接数为 2）。
