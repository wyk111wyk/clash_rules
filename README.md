# clash_rules

wyk111wyk/qx_rules 的 Clash(mihomo) 转换版，由 convert_qx_rules.py 生成。

- `AdRules/` 广告拦截（_direct 白名单 / _reject 拦截，均 domain 行为）
- `Policy/` 分流列表（*_domain.list 为 domain 行为，*_classical.list 为 classical 行为）
- 行格式为 mihomo 规则文件原生格式，策略由引用方 config 的 RULE-SET 行决定（对应 QX force-policy）
- 再生成: `python3 convert_qx_rules.py <qx_lists_dir> <订阅URL> <repo_dir> <config_out>`
