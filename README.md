# Clash 订阅转换配置

## 文件分类

| 目录 | 内容 |
| --- | --- |
| [clash/](clash/) | Clash / Mihomo 转换配置及基础模板，包括地区分组、中转和 TikTok 配置 |
| [stash/](stash/) | Stash 专用转换配置及基础模板 |
| [rules/](rules/) | 可独立引用的个人域名、italki 和 Zoom 规则集 |

配置按客户端分类，INI 与对应的基础 YAML 放在同一目录。调整目录后，客户端中的订阅转换地址需要改为下文的新地址；新路径在提交并推送到 `main` 后生效。

## Clash / Mihomo

Clash（Mihomo 内核）配置由两个文件组成：

| 文件 | 用途 |
| --- | --- |
| [clash.ini](clash/clash.ini) | subconverter 外部配置：生成策略组和流量规则 |
| [clash-base.yaml](clash/clash-base.yaml) | Mihomo 基础模板：redir-host、DNS 分流和嗅探 |

其他 Clash 转换配置：[clash-country.ini](clash/clash-country.ini)、[clash-relay.ini](clash/clash-relay.ini)、[tiktok.ini](clash/tiktok.ini)。这套配置需要 Mihomo 内核，也可用于 OpenClash。

### 使用方式

在 OpenClash 的订阅转换设置中，填写以下自定义外部配置地址：

```text
https://raw.githubusercontent.com/haoqianwei/my-clash-ini/main/clash/clash.ini
```

然后选择支持 Mihomo 的 Clash 转换目标，通常是 `target=clash&new_name=true`。转换服务须能访问 GitHub Raw，并允许读取 `clash_rule_base` 指定的远程 YAML 基础模板。只支持本地基础模板的自建转换服务，可将 `clash/clash-base.yaml` 放到服务的 `base/` 目录，并把 INI 中的 `clash_rule_base` 改为 `base/clash-base.yaml`。

更新订阅后，在 OpenClash 中选择 redir-host、启用 DNS 劫持和 DNS 遵循规则。基础模板默认 DNS 端口为 `7874`，实际以 OpenClash 设置和运行配置为准。不要再通过“自定义上游 DNS”“追加运营商 DNS”等覆写选项加入另一组上游。

进程匹配由 OpenClash 的覆写设置管理，路由器上可选择关闭（`off`）。模板不写入 `find-process-mode`：转换器可能把字符串 `"off"` 输出为无引号的 `off`，随后被 OpenClash 的 YAML 1.1 解析器读成布尔值 `false`，导致内核报 `invalid find process mode`。

模板将地理数据库定时更新交给 OpenClash：请在 OpenClash 中启用 GeoIP / GeoSite 数据更新。订阅应提供对应地区节点；“优选节点”按名称中的“落地”筛选，延续 `clash/clash-country.ini` 的分组方式。

本地 Cord 节点通过 OpenClash 的本地节点配置或自定义覆写单独维护，节点凭据不放在仓库模板中。

### DNS 路径

| 查询类型 | 解析方式 |
| --- | --- |
| 国内、私有域名 | 阿里 DoH `223.5.5.5` / `223.6.6.6`，指定 `#DIRECT` |
| 既有业务例外：`local.rightcapital.dev`、`italki.com`、`haoqianwei.com`、`haoqianwei.cn` | 同上，保留原有解析需求 |
| 其他域名 | Cloudflare DoH `1.1.1.1` / `1.0.0.1`，指定 `#🚀 国外流量` |
| 代理节点域名 | 独立的直连加密解析，避免依赖尚未建立的代理 |
| DoH 服务的引导解析 | IP 地址形式的阿里 DoH，无需先查询 DoH 域名 |

没有 `fallback` 和 `fallback-filter`：未知域名默认只使用代理侧解析器。微软、苹果、Zoom 等流量仍按原业务规则直连，但其 DNS 按上述域名分类处理。

`nameserver-policy` 的 YAML 引用仅用于复用国内解析器列表。`proxy-server-nameserver` 使用独立列表，订阅需要专用节点 DoH 时可单独修改，不会影响业务域名的 DNS 策略。

`🚀 国外流量` 必须保持为代理出口。如果重命名策略组，需要同步修改 YAML 中的 `#策略组` 后缀。浏览器或应用自行指定的 DoH 不属于 53 端口劫持范围；希望使用这套 DNS 分流时，应让应用使用系统 DNS。

### 与现有 OpenClash 覆写共存

`/etc/openclash/custom/openclash_custom_overwrite.sh` 中的自定义修改在模板之后执行。如果路由器仍在调用之前安装的 `dns_split_policy.rb`，最终 DNS 设置由该脚本决定；本模板与该脚本采用相同的 DNS 分流原则。以后修改 DNS 策略时，需要同步更新脚本，或确认模板生效后移除该脚本的调用。

OpenClash 也可能覆写端口、控制器、嗅探或 DNS 配置。排查时应查看实际运行配置 `/etc/openclash/config.yaml`，而非只看订阅原文件。

### 验证

已使用 subconverter-extended v1.9.13 将包含五个地区及落地节点的模拟订阅转换为完整配置，并通过 Mihomo v1.19.32 的 `-t` 校验；同时检查了 DNS 出口、策略组引用和规则目标。

更新订阅后，先检查 `/etc/openclash/config.yaml`：

- `dns.enhanced-mode` 为 `redir-host`。
- 默认 `nameserver` 包含 `#🚀 国外流量`，国内策略包含 `#DIRECT`。
- 没有额外的 `fallback`、`direct-nameserver` 或运营商 DNS。
- 节点域名解析不依赖代理，国内 DoH 和国外代理节点均能正常连接。

从 Wi-Fi 客户端分别测试国内域名、国外域名，以及手动指定 `8.8.8.8` 的 UDP / TCP 查询；结合 Mihomo 连接记录确认国外 DoH 的代理链。DNS 测试显示国内解析器时，应检查被测域名是否属于国内或业务例外，及浏览器是否启用了自定义 DoH。

## Stash

Stash 配置沿用 `clash/clash.ini` 的业务分流、策略组、地区筛选和默认选择，适用于 Stash iOS/tvOS 3.6+ 或 macOS 4.3+。

| 文件 | 用途 |
| --- | --- |
| [stash.ini](stash/stash.ini) | subconverter 外部配置：生成节点、策略组和 Stash 分流规则 |
| [stash-base.yaml](stash/stash-base.yaml) | Stash 基础模板：DNS 分流、Fake-IP 例外和 Telegram IP 规则集 |

### 使用方式

将节点订阅通过 subconverter 转换，使用 `target=clash&new_name=true`，自定义外部配置地址填写：

```text
https://raw.githubusercontent.com/haoqianwei/my-clash-ini/main/stash/stash.ini
```

转换服务需要支持 INI 模板变量，并能读取 `clash_rule_base` 指定的远程 YAML。只支持本地基础模板的自建服务，可将 `stash/stash-base.yaml` 放到服务的 `base/` 目录，并把 INI 中的 `clash_rule_base` 改为 `base/stash-base.yaml`。将转换得到的完整 YAML 订阅地址添加到 Stash；INI 和基础 YAML 本身都不包含节点，不能直接作为完整配置导入。

新增文件提交并推送到 `main` 后，上述 GitHub Raw 地址才会生效。首次启用时，Stash 需要下载 GeoSite 数据和 Telegram IP 规则集，请先确保所选代理可用；GeoSite 数据由 Stash 按需加载，不使用 Mihomo 的 `geox-url`。

### 与 Clash / Mihomo 配置的差异

- DNS 使用 Stash 的 `follow-rule: true`。INI 顶部将阿里 DoH 的 IP 设为直连、Cloudflare DoH 的 IP 设为 `🚀 国外流量`，不使用 Mihomo 的 `respect-rules` 或 DNS 地址中的 `#策略组` 后缀。
- 国内域名、私有域名和既有业务例外使用阿里 DoH；其他域名使用经代理访问的 Cloudflare DoH。节点域名由独立的 `proxy-server-nameserver` 直连解析。不配置 `fallback`，部分直连业务的 DNS 仍可能依赖代理，与 `clash/clash.ini` 配置的取舍一致。
- 保留 Stash 自带的 Fake-IP 接管方式，不配置路由器的 redir/tproxy 端口、DNS 监听、Mihomo 嗅探或地理数据库参数。私有域名和 `local.rightcapital.dev` 返回真实 IP；对应本地服务仍须能从运行 Stash 的设备访问。
- Stash 的 `GEOIP` 使用 MaxMind 数据库。Telegram 域名保留 `GEOSITE,telegram`，IP 则使用独立的 `telegram-ip` 规则集，来源与原 `geoip-lite.dat` 相同，每 24 小时更新。

地区组仍要求订阅提供名称匹配的节点，“优选节点”仍按“落地”筛选。空组回退、FCM 直连、苹果和微软默认直连等行为沿用原配置。

导入后，在 Stash 中确认国外策略组选择了有效代理，检查国内 DNS 直连、Cloudflare DoH 经代理，以及 Telegram IP 连接命中 `📲 Telegram`。Stash 专用参数参考[官方 DNS 文档](https://stash.wiki/features/dns-server)、[规则类型](https://stash.wiki/rules/rule-types)和[规则集合](https://stash.wiki/rules/rule-set)。

### 验证

已使用 MetaCubeX/subconverter v0.9.2 发布包，将包含五个地区、落地节点和流量说明条目的模拟订阅转换为完整 YAML，确认保留 6 个有效节点、16 个策略组和 29 条规则。检查了策略组引用与循环、例外规则顺序、四个 DoH IP 的出口，以及 Telegram IPv4/IPv6 规则集。尚未在 Stash 客户端进行连接实测。

## 共享规则集

以下文件均为 Clash classical 格式，规则内容位于 `payload` 下，可供转换配置或兼容客户端单独引用。

| 文件 | 内容 |
| --- | --- |
| [haoqianwei-rule-provider.yaml](rules/haoqianwei-rule-provider.yaml) | `haoqianwei.com`、`haoqianwei.cn` 域名 |
| [italki-rule-provider.yaml](rules/italki-rule-provider.yaml) | 域名关键词 `italki` |
| [zoom-rule-provider.yaml](rules/zoom-rule-provider.yaml) | Zoom 相关域名 |
