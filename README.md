# clash_rules

手工维护的 mihomo 规则仓库；三端（macOS / iOS / Android）共用的 `config.yaml` 所引用的全部 rule-provider 列表都在这里。

- `AdRules/` 广告拦截
  - `AdBlock_direct.list` 白名单（DIRECT，优先级高于拦截；**误杀时往这里加一行**）
  - `AdBlock_core.list` 精简核心拦截表 —— **config.yaml 实际引用它，三端(桌面/iOS/Android)共用**
  - `AdBlock_reject.list` 全量拦截表（77.7k 条）—— 目前**未被任何配置引用**，仅作 AdBlock_core 的筛选素材保留
  - AdBlock_core 筛选规则：保留「广告/追踪强语义」条目（`ad*` 词元 / 已知追踪品牌 doubleclick·googlesyndication·umeng·cnzz 等 /
    `track*`·`analytic*`·`pixel`·`beacon`·`telemetry`·`impression`），若为 IM/支付/系统类平台端点则剔除；
    约 2.9k 条 vs 全量 77.7k 条
  - ⚠️ `AdBlock_core.list` **手工维护**（原转换器已下线）；如需重建，按上述筛选规则从 `AdBlock_reject.list` 重新裁剪
- `Custom/` 自定义临时规则（手工维护；custom_proxy 走优选 / custom_direct 直连）
- `Policy/` 分流列表，按类别分目录：AI / Apple / Game / Tool / CN / Global
  （*_domain.list 为 domain 行为，*_classical.list 为 classical 行为）
- 行格式为 mihomo 规则文件原生格式，策略由引用方 config 的 RULE-SET 行决定
- 维护方式：直接编辑对应 `*.list` 并推送；客户端**刷新 provider**（改了哪条刷哪条，或「刷新全部 provider」）即生效，
  **Hako 1.0.8 起无需再删除 profile 重新添加**（详见 `DEBUGGING.md` §2.1、§6）
