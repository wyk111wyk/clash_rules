# 调试与排障指南（clash_rules × Hako Clash）

> 目的：改规则时一次做对，别再踩「push 了但客户端不生效」「手改缓存把规则集搞空」这两个坑。
> 适用客户端：Hako Clash（macOS，Clash Meta / mihomo 内核）。
> 本文只描述机制与操作，不含任何订阅链接、密钥、节点或账号信息。

---

## 0. TL;DR：改规则的正确闭环（照做即可）

1. **改本仓库的 list 文件**（唯一源头，见 §8 落点速查）。
2. **本地 commit**；**push 最多试一次**，失败就停手，等网络恢复后手动推。
3. Hako → 侧边栏 **「配置 Profiles」** → **右键当前订阅 → 删除**。
4. 再 **「添加配置」** → **粘贴同一个订阅链接**（全量重新下载）。
5. **把 VPN 开关关一次再开一次**。
6. **重选被重置的节点**（profile 内手动选过的节点会恢复默认）。
7. **按 §4 验证**（看后端实际条数 + 新连接命中，而不是只看面板数字）。

**为什么非要「删除重加」**：见 §2、§6。光点「同步 / 更新订阅」不重建规则数据缓存。

---

## 1. 架构：谁是谁的源头

| 层 | 位置 | 性质 |
|---|---|---|
| **策略源头** | 本仓库的 `*.list` 文件 | 手工维护，**唯一真相** |
| 生成产物 | 本地 `config.yaml`（由转换脚本产出） | 现已废弃转换链，只需维护 list 文件 |
| 客户端运行配置 | Hako 的已解析 config（本地） | Hako 解析订阅后落盘 |
| 客户端规则数据 | Hako 的 `provider-*.txt` 缓存 | **`type: file`，带 sha256 校验** |

关键事实（实测）：Hako 运行时把**全部** rule-provider 都当 **`type: file`** 加载，**没有任何一条是 `type: http`**。也就是说——**客户端运行时根本不联网去拉你的规则仓库**。

---

## 2. 核心机制（不理解必踩坑）

1. Hako 解析订阅时，把每个 rule-provider **下载成本地文件**，存进
   `~/Library/Group Containers/<Hako 容器>/working/store/profiles/<profileID>/revisions/<revID>/providers/provider-*.txt`，
   并在同目录 `manifest.json` 记录每个文件的 **sha256 + bytes**。
2. 内核加载时**只读这些本地文件**，并做 **sha256 完整性校验**。
3. 这些 provider 上被打了 **`x-hako-side-update-safe: true`** 标记 → 后续「同步 / 更新订阅」**会跳过**这些文件的重新下载。

**推论（最重要的一条）**：
> **往 GitHub push 成功 ≠ 客户端生效。** 中间隔着一层本地 file 缓存 + 独立刷新周期。

---

## 3. 三条绝对不要做的事（血泪教训）

1. ❌ **绝不要直接编辑** Hako 缓存里的 `provider-*.txt`
   → 内容一变、与 manifest 的 sha256 对不上 → Hako 判定该规则集损坏 → **该规则集 `ruleCount` 归零（规则全失效）**，且**无法用 API 恢复**（只能靠 GUI 重载或重加 profile）。
2. ❌ **不要改 `manifest.json` 里的 sha256** 去绕过校验 → 同为 hack，且 Hako 对 profile 上层可能还有校验，风险更高。
3. ❌ **不要指望 Clash API 刷新 file 类型 provider**
   → `PUT /providers/rules/<name>` 会返回 **HTTP 204**，但那是**假成功**：`file` 类型没有 url 可重拉，它只会重读本地文件；若文件已被手动改过，反而直接把规则集清空。

> 正确做法只有一个：**让 Hako 自己重新下载**（§0 第 3–4 步）。

---

## 4. 验证改动是否真的生效

**不要只看面板上的规则集条数**——它可能是 UI 缓存。要以后端为准。

### 4.1 看后端实际加载的条数（推荐）
Clash 控制端口默认 `127.0.0.1:9090`（本机配置无 secret）：

```bash
python3 - <<'PY'
import json, subprocess
out = subprocess.run(["curl","-s","--compressed","http://127.0.0.1:9090/providers/rules"],
                     capture_output=True, text=True).stdout
d = json.loads(out)["providers"]
for name, p in d.items():
    print(f'{name:24} ruleCount={len(p.get("rules", []))}  vehicle={"file" if p.get("vehicle") is None else "http"}')
PY
```

- 关注某条时，把它的 `ruleCount` 和 **GitHub 上该 list 的有效行数**对比，一致才算生效。
- ⚠️ 该版本**单条 provider 端点不支持 GET**，只能拉上面这个列表端点。

### 4.2 看「新连接」的真实命中（最终判据）
在 Hako 的 **Connections / 连接** 页，新开一个请求，看该域名实际命中的**策略组**。
> 官方文档明确：**不要只判断「按钮点过了」**；而且**已有长连接不会迁移到新路线**——刷新网页可能继续复用旧连接。要**断开重连 / 新开**才看得到新策略。

---

## 5. Clash 控制端口（API）能力边界（实测）

| 端点 | 可用性 | 说明 |
|---|---|---|
| `GET /providers/rules` | ✅ | 列出所有 provider 的 `ruleCount` / `vehicle` |
| `PUT /providers/rules/<name>` | ⚠️ | 返回 204，但对 `type: file` 是**假成功** |
| `GET /providers/rules/<name>` | ❌ | 该版本单条端点**不支持 GET** |
| `POST /restart` | ❌ 404 | 不支持 |
| `PUT /configs` | ❌ 405 | 不支持 |

**结论**：这套 API 只能用来**读诊断信息**，不能用来修复/刷新 file 类型 provider。

---

## 6. 为什么「更新订阅 / 同步」不生效

两个原因叠加：

1. **独立刷新周期**：每个 provider 有 `interval`（本仓库统一 **90 天** = `7776000` 秒），客户端不会自动拉。
2. **`x-hako-side-update-safe` 标记**：让「同步 / 更新订阅」**跳过** provider 的重新下载 → 只 reload 了配置，数据还是旧的。

### 6.1 只开关一下 VPN 开关，会重新拉取吗？
**不会。** 机制上：VPN 开关 = 启停内核隧道 + 重新加载**当前已解析的本地配置与本地 provider 缓存**，**不触发订阅重解析、也不重新下载 provider**。
（旁证：之前多次「同步 + 开关切换」之后，缓存 txt 的 mtime 一直没变，条数也一直是旧的。）

**想确认？做一个 30 秒的安全实验（只读，不改任何文件）**：
1. 先记录现状（本文档最后附的「快照命令」跑一次，记下两个 provider 的 `ruleCount` 和缓存 `mtime`）。
2. 只把 VPN 开关 **关 → 开**。
3. 再跑一次快照。
4. `ruleCount` 与 `mtime` **都没变** → 证实「开关不重新拉取」；**变了** → 说明该版本会顺带刷新，也要记下来。

### 6.2 为什么「备份恢复」也不行？
「恢复配置」只还原 config 文本，**不重建 provider 数据缓存**（那是独立资源）。所以只恢复配置、不恢复全量数据，缓存依旧是旧的 → 依旧不生效。必须走「删除 profile 重新从同一链接添加」。

---

## 7. 改动后自检清单

- [ ] list 文件行格式正确（domain 行为：`+.example.com` / `example.com`；classical 行为：`DOMAIN-SUFFIX,example.com`）
- [ ] 本仓库已 commit（`git log --oneline -3`），必要时已 push，且 `git status` 干净
- [ ] GitHub 侧可直连验证条数：
      `curl -s https://raw.githubusercontent.com/<owner>/<repo>/main/<path> | grep -vcE '^\s*#|^\s*$'`
- [ ] Hako 里**删除并重新添加** profile（不是只点「同步」）
- [ ] VPN 开关关/开各一次
- [ ] 后端 `ruleCount` 与 GitHub 条数**一致**（§4.1）
- [ ] 被重置的节点已**重选**
- [ ] 新开连接在 Connections 里命中**正确的策略组**（§4.2）

---

## 8. 规则落点速查（改某个域名该动哪个文件）

| 想要的效果 | 改哪个文件 | 备注 |
|---|---|---|
| 某域名**直连**（不走代理） | `Custom/custom_direct.list` | 映射为 `RULE-SET,CustomDirect,DIRECT`，**排在很前**，优先级高，最省事 |
| 境外 **AI / LLM 工具**走 `🤖境外LLM`（JP/SG/US） | `Policy/AI/llm_ai_domain.list` | 规避「香港节点被境外 LLM 拒绝」的问题 |
| 广告/跟踪**拦截** | `AdRules/AdBlock_reject.list` | 排在很前，会先于其它规则命中 |
| 广告拦截**白名单** | `AdRules/AdBlock_direct.list` | 需要放行的域加这里 |
| 其它分类 | `Policy/{AI,Apple,Game,Tool,CN,Global}/` | 按业务分目录 |

**优先级口径**：**策略由引用方 config 的 `RULE-SET` 行决定，且按 rules 段顺序自上而下匹配、先命中先决**。list 文件本身只装域名、不带策略。所以「想直连」→ 放进排在前面的 DIRECT 列表（如 `CustomDirect`）。

**两个容易漏的形态**：
- **裸 IP 连接**：某些工具（如 Anti-Gravity）自己解析 DNS 后直连 IP（例如连接列表里出现 `172.217.117.4:443`），这种连接**没有域名**，域名规则根本看不到它 → 必须由 `Policy/AI/llm_ai_classical.list` 里的 `IP-CIDR` 兜底。加 IP 规则时用 `,no-resolve`（与 `CustomDirect` 写法一致），只匹配真实 IP、不做域名反查。
- **Google 自有顶级域 `.goog`**：不要只用 `.google.com` 思路去猜；Google 有很多服务跑在 `*.goog` 下（如 `antigravity-unleash.goog`），`googleapis.com` 的通配**覆盖不到**它。查缺口的权威办法是拿 `MetaCubeX/meta-rules-dat` 的 `geo/geosite/<分类>.yaml` 与本仓库列表做**后缀语义**比对（见 §11）。

---

## 9. 关于 `geox-url` 为什么还是第三方 CDN（不是漏改）

`geox-url`（`mmdb` / `asn` / `geoip` / `geosite`）指向的是**内核启动时下载的公共地理数据二进制文件**，来源是第三方数据集仓库，**与本规则仓库无关**。它保持第三方 CDN 是**有意保留**，不是遗漏：

- **生命周期完全独立**：它跟规则列表的刷新机制是两回事。
- **几乎不联网**：本机 `geo-auto-update: false` → 只在**文件缺失或手动更新时**下载一次，之后走本地缓存（`working/geoip.metadb` 等，实测已缓存、日期很久没动）。
- **改成 raw 无收益**：换成 GitHub raw 不会带来任何「更新更快」的好处，反而 raw 对**大二进制文件**的直连可达性更差（国内尤其）。
- **万一真要改**：4 条都改成
  `https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/release/...`，
  并且必须**删除本地已缓存的 geo 文件**才会触发重新下载。**不建议动。**

---

## 10. 附带：本机网络注意事项

- 本机 GitHub 偶发 `SSL_ERROR_SYSCALL` / 连不上：**本地 commit 必做，push 试一次，失败即停手**，不要反复重试、不改 remote、不换 SSH、不加跳过校验的环境变量。
- 快速判断是「网络问题」还是「仓库问题」：
  `curl -sS -o /dev/null -w "%{http_code}" https://www.baidu.com` 通（200）而
  `https://api.github.com` 不通（000）→ 属于**本机代理/上游故障**，与仓库无关。
- 只读获取 GitHub 仓库信息可走网页抓取通道（直连 API 可能被 SSL 拦）。

---

## 11. 如何系统性查规则缺口（与官方 geosite 比对）

不要靠记忆猜缺哪些域名。用 `MetaCubeX/meta-rules-dat`（内核 geoip 数据的同一来源）的官方分类做**后缀语义**比对——注意 `+.googleapis.com` 这类通配已覆盖其全部子域，直接字符串比分会误报一堆"缺失"：

```bash
# 1) 取官方分类（raw 直连可达）
curl -s https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/category-ai-!cn.yaml -o /tmp/ai.yaml   # 境外 AI 工具总集
curl -s https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/google-gemini.yaml -o /tmp/gem.yaml     # Google 系 AI
# 2) 用 Python 做后缀语义比对（照抄仓库里的比对脚本，只算"真正未覆盖"的）
# 3) 加之前先做冲突检查：确认新域名没有被排在前面的 AdBlock(REJECT) 命中，否则加了也不生效
```

**IP 维度**的权威来源是 Google 官方公开段：`https://www.gstatic.com/ipranges/goog.json`（自有服务）/ `cloud.json`（GCP 客户段）。只取 `goog.json` 的核心服务段即可，避免误伤托管在 GCP 上的其它服务。

常用分类速查：`category-ai-!cn`（境外 AI 工具）、`google-gemini`（Google 系 AI）、`openai`、`anthropic`、`category-ai-chat-!cn`。

---

## 12. 路由「正确」但连接被掐断（EOF / 随机超时）怎么查

典型现象：AI 工具（Antigravity / Codex / Claude Code 等）报 `request failed ... EOF`，或连接随机失败；**但规则本身是对的**。此时不要去改规则，按下面顺序定位：

**第 1 步：先证明路由是对的**（抓内核日志）

```bash
B=http://127.0.0.1:9090
curl -s --compressed -N "$B/logs?level=debug" --max-time 20 > /tmp/l.log 2>&1 &
# 同时触发一次请求，然后看命中
grep -a "match RuleSet" /tmp/l.log | tail -20
# 形如：[TCP] ... -> daily-cloudcode-pa.googleapis.com:443 match RuleSet(LLM_AI_dom) using 🤖境外LLM[🇺🇸 US 02]
```

**第 2 步：区分「规则问题」还是「链路/节点问题」** —— 用代理端口做端到端 A/B（同一节点下，对比"Google 系"与"非 Google 系"目标）：

```bash
curl -sS -o /dev/null -x http://127.0.0.1:7890 --max-time 8 -w "%{http_code}\n" https://daily-cloudcode-pa.googleapis.com/
curl -sS -o /dev/null -x http://127.0.0.1:7890 --max-time 8 -w "%{http_code}\n" https://openrouter.ai/
# 一个通一个不通 → 是"节点到该目标的链路"问题，不是规则问题
```

**第 3 步：节点 A/B**（`🤖境外LLM` 是 select 组，可用 API 切换后实测，测完还原）：

```bash
# 切换：PUT /proxies/<组名> {"name":"<子组>"}（组名需 URL 编码，emoji 亦然）
# 然后重复第 2 步若干次，统计成功率
```

**实战结论（本仓库实测，2026-09）**：同一个机场，不同地区出口 IP 对 Google 的可用性差异极大 —— 目标 `daily-cloudcode-pa.googleapis.com`，累计成功率 US 3/10→12/12 剧烈波动、JP 2/6、**SG 43/46 最稳**、`🛰家宽兜底` **0/8（该组节点是死的）**。**遇到 Google 系间歇性 EOF，第一动作是换节点，而不是改规则。**

**两个坑（都实测踩过）**：

- **无代理的 `curl` 仍会被 TUN 捕获**（日志里源地址是 `198.18.0.1`），所以"直连测试成功"是假象，不能据此判断"直连可用"。
- **TUN + `fake-ip`** 是社区已知会影响 `*.googleapis.com` 访问的组合（表现为 Antigravity 登录/流式请求 EOF）。备选缓解：把相关域加进 `dns.fake-ip-filter`，或把 `dns.enhanced-mode` 改为 `redir-host`。改动属 **config.yaml 的 `dns:` 段**（不是规则列表），且注意国内 `nameserver` 解析 Google 域可能给出劣质 IP，需配合 `nameserver-policy` 才稳妥。

---

## 附录：快照命令（§6.1 实验与日常体检通用）

```bash
# 打印两个常改 provider 的条数 + 缓存文件 mtime（只读，不改任何文件）
GC="$HOME/Library/Group Containers/<Hako 容器目录>"
AC="$GC/working/active/config.yaml"
for n in LLM_AI_dom CustomDirect; do
  p=$(grep -A5 "^    $n:" "$AC" | grep "path:" | sed 's/.*path:[[:space:]]*//')
  printf '%-14s 有效行=%s  mtime=%s\n' "$n" \
    "$(grep -vE '^[[:space:]]*#|^[[:space:]]*$' "$p" | wc -l | tr -d ' ')" \
    "$(stat -f '%Sm' "$p")"
done
```

> 提示：Hako 的容器目录名以实际安装为准，用 `ls "$HOME/Library/Group Containers" | grep -i hako` 查得；profile/revision 的 UUID 会随「删除重加」变化，属正常。
