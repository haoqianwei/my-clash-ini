# Clash 订阅转换配置

## OpenClash / Mihomo

OpenClash 专用配置由两个文件组成：

| 文件 | 用途 |
| --- | --- |
| [openclash.ini](openclash.ini) | subconverter 外部配置：生成策略组和流量规则 |
| [openclash-base.yaml](openclash-base.yaml) | Mihomo 基础模板：redir-host、DNS 分流和嗅探 |

现有 `clash-country.ini`、`clash-relay.ini`、`tiktok.ini` 保留原用途；这套配置只面向 OpenClash / Mihomo。

### 使用方式

在 OpenClash 的订阅转换设置中，填写以下自定义外部配置地址：

```text
https://raw.githubusercontent.com/haoqianwei/my-clash-ini/main/openclash.ini
```

然后选择支持 Mihomo 的 Clash 转换目标，通常是 `target=clash&new_name=true`。转换服务须能访问 GitHub Raw，并允许读取 `clash_rule_base` 指定的远程 YAML 基础模板。只支持本地基础模板的自建转换服务，可将 `openclash-base.yaml` 放到服务的 `base/` 目录，并把 INI 中的 `clash_rule_base` 改为该本地路径。

更新订阅后，在 OpenClash 中选择 redir-host、启用 DNS 劫持和 DNS 遵循规则。基础模板默认 DNS 端口为 `7874`，实际以 OpenClash 设置和运行配置为准。不要再通过“自定义上游 DNS”“追加运营商 DNS”等覆写选项加入另一组上游。

模板将地理数据库定时更新交给 OpenClash：请在 OpenClash 中启用 GeoIP / GeoSite 数据更新。订阅应提供对应地区节点；“优选节点”按名称中的“落地”筛选，延续 `clash-country.ini` 的分组方式。

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
