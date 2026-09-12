# clash_rules

wyk111wyk/qx_rules 的 Clash(mihomo) 转换版，由 convert_qx_rules.py 生成。

- `AdRules/` 广告拦截
  - `AdBlock_direct.list` 白名单（DIRECT，优先级高于拦截；**误杀时往这里加一行**）
  - `AdBlock_core.list` 精简核心拦截表 —— **config.yaml 实际引用它，三端(桌面/iOS/Android)共用**
  - `AdBlock_reject.list` 全量拦截表（77.7k 条）—— 目前**未被任何配置引用**，仅作 AdBlock_core 的筛选素材保留
  - AdBlock_core 筛选规则：保留「广告/追踪强语义」条目（`ad*` 词元 / 已知追踪品牌 doubleclick·googlesyndication·umeng·cnzz 等 /
    `track*`·`analytic*`·`pixel`·`beacon`·`telemetry`·`impression`），若为 IM/支付/系统类平台端点则剔除；
    约 2.9k 条 vs 全量 77.7k 条
  - ⚠️ `convert_qx_rules.py` 目前**只生成 `AdBlock_reject.list`，不生成 `AdBlock_core.list`**；
    重新生成规则后需按上述规则手工重建核心表（待后续内建到转换器）
- `Custom/` 自定义临时规则（手工维护，转换器不覆盖；custom_proxy 走优选 / custom_direct 直连）
- `Policy/` 分流列表，按类别分目录：AI / Apple / Game / Tool / CN / Global
  （*_domain.list 为 domain 行为，*_classical.list 为 classical 行为）
- 行格式为 mihomo 规则文件原生格式，策略由引用方 config 的 RULE-SET 行决定（对应 QX force-policy）
- 再生成: `python3 convert_qx_rules.py <qx_lists_dir> <订阅URL> <repo_dir> <config_out>`
