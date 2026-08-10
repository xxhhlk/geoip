# 简介

本项目是 [Loyalsoldier/geoip](https://github.com/Loyalsoldier/geoip) 的一个 fork，复用其 CLI 工具定制生成分类 GeoIP 数据文件。

每周通过 GitHub Actions 自动构建（**每周一 05:00 北京时间 / 每周日 21:00 UTC**），从多个上游数据源拉取最新 IP 列表，生成以下产物并发布到 [release 分支](https://github.com/xxhhlk/geoip/tree/release) 与 GitHub Release：

- `xxhhlk.dat`：V2Ray 格式 dat 文件
- `xxhhlk.txt`：纯文本 CIDR 列表（按分类分段）

也可在仓库页面手动触发 `workflow_dispatch` 立即构建。

## 分类与数据源

产物中的标签（entry name）以**大写**形式呈现（`AWS` / `AKAMAI` / `CHINAMOBILE`）。

| 标签 | 数据源 | 说明 |
|---|---|---|
| `AWS` | [axpwx/IP-Data](https://github.com/axpwx/IP-Data) 的 `provider/aws-cidr-ipv4.txt` 与 `provider/aws-cidr-ipv6.txt` | AWS 官方地址段（IPv4 + IPv6） |
| `AKAMAI` | [harrisonwang/cloud-ip-crawler](https://github.com/harrisonwang/cloud-ip-crawler) 的 `cloud-ip.csv`（Release `dataset-latest`），抽取 `akamai` 与 `linode` 两部分 | Akamai / Linode 地址段 |
| `CHINAMOBILE` | [china-operator-ip](https://china-operator-ip.yfgao.com/cmcc46.txt) | 中国移动地址段 |

> Akamai 上游只提供 CSV（非纯文本 CIDR），因此构建时在 workflow 中先下载 CSV 再 `awk` 抽取 `provider` 为 `akamai` / `linode` 的行，生成本地 `akamai.txt` 供 CLI 读取；其余数据源均为远程文本直链。

## 下载与使用

产物发布在 [release 分支](https://github.com/xxhhlk/geoip/tree/release)。若 `raw.githubusercontent.com` 无法访问，可改用 `cdn.jsdelivr.net` 或 `fastly.jsdelivr.net`。`*.sha256sum` 为校验文件。

### V2Ray dat 格式（`xxhhlk.dat`）

适用于 [V2Ray](https://github.com/v2fly/v2ray-core)、[Xray-core](https://github.com/XTLS/Xray-core)、[mihomo](https://github.com/MetaCubeX/mihomo)、[hysteria](https://github.com/apernet/hysteria)、[Trojan-Go](https://github.com/p4gefau1t/trojan-go) 等。

- **xxhhlk.dat**
  - [https://raw.githubusercontent.com/xxhhlk/geoip/release/xxhhlk.dat](https://raw.githubusercontent.com/xxhhlk/geoip/release/xxhhlk.dat)
  - [https://cdn.jsdelivr.net/gh/xxhhlk/geoip@release/xxhhlk.dat](https://cdn.jsdelivr.net/gh/xxhhlk/geoip@release/xxhhlk.dat)
- **xxhhlk.dat.sha256sum**
  - [https://raw.githubusercontent.com/xxhhlk/geoip/release/xxhhlk.dat.sha256sum](https://raw.githubusercontent.com/xxhhlk/geoip/release/xxhhlk.dat.sha256sum)
  - [https://cdn.jsdelivr.net/gh/xxhhlk/geoip@release/xxhhlk.dat.sha256sum](https://cdn.jsdelivr.net/gh/xxhhlk/geoip@release/xxhhlk.dat.sha256sum)

Xray / V2Ray routing 用法示例：

```json
"routing": {
  "rules": [
    { "type": "field", "outboundTag": "Proxy",  "ip": ["ext:xxhhlk.dat:aws"] },
    { "type": "field", "outboundTag": "Proxy",  "ip": ["ext:xxhhlk.dat:akamai"] },
    { "type": "field", "outboundTag": "Direct", "ip": ["ext:xxhhlk.dat:chinamobile"] }
  ]
}
```

mihomo 用法示例：

```yaml
geodata-mode: true
geox-url:
  geoip: "https://cdn.jsdelivr.net/gh/xxhhlk/geoip@release/xxhhlk.dat"
```

### 纯文本格式（`xxhhlk.txt`）

按分类分段的 CIDR 文本，可直接用于 Clash / mihomo 的 `ipcidr` rule-provider 等。

- **xxhhlk.txt**
  - [https://raw.githubusercontent.com/xxhhlk/geoip/release/xxhhlk.txt](https://raw.githubusercontent.com/xxhhlk/geoip/release/xxhhlk.txt)
  - [https://cdn.jsdelivr.net/gh/xxhhlk/geoip@release/xxhhlk.txt](https://cdn.jsdelivr.net/gh/xxhhlk/geoip@release/xxhhlk.txt)
- **xxhhlk.txt.sha256sum**
  - [https://raw.githubusercontent.com/xxhhlk/geoip/release/xxhhlk.txt.sha256sum](https://raw.githubusercontent.com/xxhhlk/geoip/release/xxhhlk.txt.sha256sum)
  - [https://cdn.jsdelivr.net/gh/xxhhlk/geoip@release/xxhhlk.txt.sha256sum](https://cdn.jsdelivr.net/gh/xxhhlk/geoip@release/xxhhlk.txt.sha256sum)

## 自行构建

本地生成需安装 [Golang](https://go.dev/dl/) 与 [Git](https://git-scm.com)：

```bash
git clone https://github.com/xxhhlk/geoip.git
cd geoip
go run ./ convert -c ./config.json
```

产物输出到 `./output` 目录。

> **注意**：Akamai 数据源为 CSV，需先生成本地 `akamai.txt`。GitHub Actions 中由 workflow 自动完成；本地构建时手动执行：
>
> ```bash
> curl -fsSL -o cloud-ip.csv.gz \
>   https://github.com/harrisonwang/cloud-ip-crawler/releases/download/dataset-latest/cloud-ip.csv.gz
> gunzip -c cloud-ip.csv.gz \
>   | awk -F',' '$1=="akamai" || $1=="linode"{print $2}' > akamai.txt
> ```
>
> 其余数据源（AWS、China Mobile）由 CLI 直接通过 `config.json` 中的远程 URL 拉取。

## CLI 简介

本项目的核心 CLI 工具 `geoip` 来自上游 [Loyalsoldier/geoip](https://github.com/Loyalsoldier/geoip)，作用是通过配置文件聚合多个数据源、去重、转换为目标格式并输出。常用子命令：

- `convert`：按 `config.json` 转换并生成产物
- `list`：列出支持的 input / output 格式
- `lookup`：查找某个 IP / CIDR 所属分类
- `merge`：合并并去重标准输入中的 IP 与 CIDR

```bash
$ ./geoip convert -c config.json
2026/xx/xx xx:xx:xx ✅ [v2rayGeoIPDat] xxhhlk.dat --> output
2026/xx/xx xx:xx:xx ✅ [text] xxhhlk.txt --> output
```

## 配置说明

`config.json` 关注两个概念：`input`（数据源及输入格式）与 `output`（产物去向及输出格式）。本仓库实际配置：

- **input**
  - `aws`：axpwx/IP-Data 的 IPv4 + IPv6 远程文本
  - `akamai`：本地 `./akamai.txt`（由 workflow 从 harrisonwang CSV 抽取生成）
  - `chinamobile`：yfgao 的远程文本
- **output**
  - `v2rayGeoIPDat` → `./output/xxhhlk.dat`
  - `text` → `./output/xxhhlk.txt`

更多格式与配置选项可参考上游 [`configuration.md`](https://github.com/Loyalsoldier/geoip/blob/HEAD/configuration.md)。

## License

本仓库基于 [Loyalsoldier/geoip](https://github.com/Loyalsoldier/geoip) fork，沿用其 [CC-BY-SA-4.0](https://creativecommons.org/licenses/by-sa/4.0/) 与 [GPL-3.0](https://github.com/Loyalsoldier/geoip/blob/master/LICENSE-GPL) 双许可。
