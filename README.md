# MiSub / Mihomo 分流模板

模板地址：

```text
https://raw.githubusercontent.com/dr1np/misub-ruleset/main/misub-metacubex-mihomo.yaml
```

在 MiSub 的订阅组中选择“内置转换引擎 + 自定义模板 URL”。节点由 MiSub 注入，不要将模板本身当作节点订阅导入。

## 出口策略

- 私有域名、局域网 IP 直连。
- 服务分类优先于 `cn_domain` 国内域名与国内 IP 规则；其余国内流量默认直连。
- 其余流量以 `MATCH,通用代理` 兜底，TCP、UDP 使用该组的当前选择。
- 所有分组都有 `REJECT`；除 Claude、OpenAI、Google 外，所有分组都有 `DIRECT`。Microsoft、Apple 等服务组默认选择“通用代理”，国内服务默认 `DIRECT`。
- Claude、OpenAI、Google 默认跟随“通用代理”，也可分别选择 MiSub 注入的代理节点或 `REJECT`。三组不提供独立的 `DIRECT` 选项，并排除 `Direct`、`Compatible`、`Pass` 类型的注入节点；跟随“通用代理”时，若该组切到 `DIRECT`，流量也会随之直连。
- “通用代理”允许手动选择 `DIRECT`。选中节点连接失败时不会自动改走直连或其他节点。
- 节点不支持 UDP 时，海外 UDP 由显式拒绝规则阻断，避免 Mihomo 跳过代理规则后隐式直连。
- 广告拦截的选项顺序为 `REJECT`、通用代理、`DIRECT`，默认拦截。Claude 已确认的服务依赖优先进入 Claude，其余广告优先于其他服务和国内分类。

规则先识别 Claude 原生进程和服务域名，再匹配带 `no-resolve` 的 IP 规则。未知域名不会为了判断是否属于国内 IP 而额外解析后直连；即使它最终解析到国内 CDN，也仍由“通用代理”兜底。直接访问国内 IP 时默认直连。域名归属以 MetaCubeX 的规则集为准。

## Steam 分流

| 分组 | 默认出口 | 覆盖范围 |
| --- | --- | --- |
| Steam 登录与好友 | `DIRECT` | 客户端登录、好友聊天、Steam API 和客户端配置；也匹配直接连接的 Valve 网络 IP，例如 CM 或语音中继 |
| Steam 下载与更新 | `DIRECT` | SteamPipe 游戏下载、客户端更新、已知区域下载 CDN 和部分 Steam Cloud 传输端点 |
| Steam 商店 | 通用代理 | 商店、结算、通用网页和静态资源，包括商店与社区共用的 `shared.*.steamstatic.com` |
| Steam 社区 | 通用代理 | 社区页面、市场、创意工坊页面、头像和用户内容 |

四组均可选择“通用代理”、`DIRECT`、`REJECT` 或订阅中的独立节点。客户端登录和好友连接默认直连；商店或社区页面中的网页登录仍随该页面分组。创意工坊页面走社区组，通过 SteamPipe 下载的工坊文件走下载组。HTTPS 共用域名无法按路径拆分；若商店和社区选择不同节点，共享静态资源仍随商店组。非 Valve 的游戏服务器继续按原规则匹配。

裸 IP 仅匹配 [Steam 支持列出的 Valve 网络](https://help.steampowered.com/en/faqs/view/2EA8-4D75-DA21-31EB)，IPv4 和 IPv6 网段来自 Valve 链接的 [AS32590 IPv4](https://as32590.net/ipv4.txt) 与 [IPv6](https://as32590.net/ipv6.txt) 清单，核对于 2026-11-09。域名分类参考 [Steam 连接管理服务器目录](https://api.steampowered.com/ISteamDirectory/GetCMListForConnect/v1/?cellid=0)、[SteamPipe 下载服务器目录](https://api.steampowered.com/IContentServerDirectoryService/GetServersForSteamPipe/v1/?cell_id=0)、[SteamTracking 客户端更新清单](https://github.com/SteamTracking/SteamTracking/blob/master/urls.txt) 和 [MetaCubeX Steam 域名清单](https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/steam.list)。选用不支持 UDP 的节点时，对应的 Steam UDP 连接会被拒绝；语音等 UDP 功能建议保持直连或使用支持 UDP 的节点。

Steam 的直连指连接出口，DNS 仍按下文配置。首次导入后请先为“通用代理”选好节点，让公网 DNS、Steam 商店和社区可用。

## Claude 覆盖范围与依据

核对日期：2026-11-07。采用 MetaCubeX `anthropic.mrs` 每日更新，叠加模板内置规则，避免只匹配 `claude.ai` 而遗漏登录、文件或新产品域名。

| 请求类型 | 分流方式 |
| --- | --- |
| 网站、API、Console、OAuth/SSO、下载与更新 | `anthropic.com`、`claude.ai`、`claude.com` 等完整后缀，包括所有子域名 |
| 文件、Artifacts、Chrome WebSocket、MCP 内容 | `claudeusercontent.com`、`claudemcpcontent.com`、`claudemcpclient.com` 等后缀 |
| 官网 CDN | 精确匹配 `servd-anthropic-website.b-cdn.net` |
| 插件元数据、旧版安装器 | 精确匹配 `storage.googleapis.com` |
| 当前可选遥测 | 官方文档列出的两个 Datadog 接收主机 |
| 旧版功能开关、统计 | 社区记录的 Statsig、GrowthBook、Fathom 主机 |
| Artifacts 外部字体、脚本 | 官方列出的 Google Fonts、cdnjs、jsDelivr、Tailwind、jQuery、unpkg 主机 |
| 原生 CLI、桌面端及其同名 Helper | 识别 `claude`、`claude.exe`、`Claude.exe`、`claude-desktop`、`Claude Helper` 及 Renderer/GPU/Plugin Helper；局域网规则优先，其他请求进入 Claude |
| 直接访问官方 API IP | 官方 Inbound 网段 `160.79.104.0/23`、`153.61.224.0/22`，预留 `2607:6bc0::/48`；模板仍关闭 IPv6 |

Claude 域名和原生进程规则优先于广告、Google、其他AI及国内服务；其选中节点不支持 UDP 时立即 `REJECT`，不会落入其他分组。Claude 的功能开关和遥测属于该组的完整请求范围；需要关闭可选遥测时，使用 Claude 自身设置。

共享域名有明确取舍：上述 GCS、字体、脚本 CDN 和遥测主机也可能被其他网站使用，它们的请求同样会进入 Claude。Mihomo 的域名规则不能根据 HTTPS URL 路径区分同一主机上的 Claude 对象和其他对象；没有把整个 `googleapis.com`、`gstatic.com`、`cloudflare.com`、`amazonaws.com` 或全部 CDN IP 加入 Claude。

GitHub、npm、Homebrew、用户配置的 MCP、Bedrock/Vertex/Foundry、自定义网关和工具访问的任意网站，不存在可穷举的 Claude 专属域名清单。原生 Claude 进程直接发出的公网请求由进程规则覆盖；它启动的 `git`、`node`、`curl` 等子进程、npm 版 CLI、浏览器、WSL/容器流量若无法识别为 Claude，则按各自域名分类。没有全局匹配 `node.exe`、浏览器或 `git.exe`。远端云会话发出的请求也不经过本地 Mihomo。因此这里覆盖已核对的已知请求，不能承诺未来所有请求百分之百识别。

参考来源：

- [Claude Code 官方网络要求](https://code.claude.com/docs/en/network-config)：登录、插件、Chrome 桥接、当前遥测和 Artifacts 依赖。
- [Anthropic 官方 IP 地址](https://platform.claude.com/docs/en/api/ip-addresses)：只采用 Inbound 网段，不混入 Outbound 网段。
- [MetaCubeX Anthropic 域名清单](https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/anthropic.list)：持续更新的厂商与产品域名。
- [blackmatrix7 社区补全讨论 #1671](https://github.com/blackmatrix7/ios_rule_script/pull/1671)：旧清单缺少 `claude.com`、用户内容域名和功能开关的经验；不照抄缺乏独立依据的域名。
- [xiaolai 社区规则登记表](https://github.com/xiaolai/anthropic-claude-surge-rules-set/blob/main/domains.yaml)：更新器的 GCS 依赖及 GrowthBook 记录。
- [Claude Code 社区故障 #8248](https://github.com/anthropics/claude-code/issues/8248)：功能开关请求可能导致启动长时间等待。
- [Anthropic 官方开发容器配置](https://github.com/anthropics/claude-code/blob/main/.devcontainer/init-firewall.sh)：历史 Statsig 依赖。
- [社区版本审计](https://gist.github.com/AdnaneKhan/7a2040bcdebdc923ef73a19f8831132d)：不同版本的遥测端点会变化，当前端点以官方文档为准。

实际验收时在 Mihomo 连接日志中检查网页登录、对话流式响应、文件上传/下载、Artifacts、Claude Code 登录与对话、插件列表、自动更新和 Chrome 桥接；确认目标请求命中 Claude 并使用选定节点。规则语法与离线分类检查不能代替带账号的端到端验证。

Claude 分流已使用 Mihomo 1.19.33 和本地模拟代理检查域名/IP 匹配、UDP 拒绝以及独立节点与通用代理的选择。另以测试用 `claude.exe` 验证未知域名的进程分流。三组选择独立节点时不受“通用代理”切换影响；选择“通用代理”时按其当前出口分流。

## DNS 与客户端设置

普通公网域名统一使用经过“通用代理”当前选择的 Cloudflare / Google DoH。直连网站的 DNS 查询也使用该选择，没有国内 DNS 分流策略或自动直连回退。代理节点自身的域名使用独立的阿里 DoH 直连解析，避免在建立代理前依赖代理查询自身。这项引导解析不能提供与代理内查询相同的隐私边界。

“通用代理”选中代理节点时，公网 DNS 经该节点；手动选中 `DIRECT` 时，DNS 也随之直连。Claude、OpenAI、Google 选择独立节点时，连接仍使用各自节点；选择“通用代理”时，连接也会跟随它直连。三组的独立分流不表示 DNS 查询也分别使用它们的出口。

客户端需满足以下条件：

1. 使用 Mihomo 的**规则模式**，关闭会覆盖模板的客户端 DNS 覆写。
2. 开启 TUN、自动路由、严格路由及 UDP/TCP DNS 劫持；Windows 需服务模式或相应权限。模板已包含这些设置，Clash Verge 的界面设置仍可能覆盖它们。
3. 保持模板的 IPv6 关闭设置，除非已另外验证 IPv6 的完整接管。
4. 首次导入时，Claude、OpenAI、Google 默认选择“通用代理”；在“通用代理”中选好节点，三组即可共用该出口。“通用代理”本身首次导入仍默认 `REJECT`。
5. `profile.store-selected: true` 会保留已有选择。若更新后 Claude、OpenAI、Google 仍选中旧的 `REJECT`，请手动切到“通用代理”；若广告组保留了旧的代理选项，请手动切到 `REJECT`。进程识别依赖客户端权限和运行环境。

“通用代理”的节点不可用时，使用该组的连接和公网 DNS 查询会失败；未缓存的国内域名、独立服务域名也可能因 DNS 无法经该组查询而暂时无法访问。已有国内域名/IP 直连规则不会因为海外节点掉线被改成代理。

此模板保证的是 **Mihomo 运行且流量已被接管时的分流和代理失败行为**。它不能代替操作系统级断网开关：退出内核、关闭 TUN、应用自行绕过代理，或使用独立加密 DNS 时，需另行检查实际路径。DNS 检测应结合解析器归属和实际请求路径判断，DNS 服务器 IP 无须与代理出口 IP 相同。
