# HomeProxy 智能合并上游与保留 Sing-box 1.15+ 纯净自定义规范指南

本文档专门用于指导如何优雅、安全地将 ImmortalWrt 官方上游（`immortalwrt/homeproxy:master`）的最新代码合并到个人专属仓库（`sjnhnp/homeproxy:master`），同时**完整保留 Sing-box 1.14 / 1.15+ 纯净现代化架构、TUN 完整依赖声明与网络分流最佳实践**，确保未来无论由 AI 还是人工执行合并，都不会发生语法退化或致命崩溃。

---

## 目录
1. [仓库拓扑与远端定义](#1-仓库拓扑与远端定义)
2. [为什么不能直接盲目 pull 官方上游？](#2-为什么不能直接盲目-pull-官方上游)
3. [核心自定义资产清单（绝不可被覆盖）](#3-核心自定义资产清单绝不可被覆盖)
4. [核心网络与分流排错黄金准则（诊断经验库）](#4-核心网络与分流排错黄金准则诊断经验库)
5. [未来指挥 AI 进行智能合并的提示词模版（一键复制）](#5-未来指挥-ai-进行智能合并的提示词模版一键复制)
6. [智能合并标准执行流程（Step-by-Step）](#6-智能合并标准执行流程step-by-step)
7. [常见代码冲突排查与官方解法](#7-常见代码冲突排查与官方解法)
8. [合并后必查清单（Checklist）](#8-合并后必查清单checklist)
9. [热更新到真机路由器调试验证](#9-热更新到真机路由器调试验证)

---

## 1. 仓库拓扑与远端定义

当前本地工作区路径：`D:\ProgramData\Antigravity\homeproxy`

| 远端名称 | 仓库地址 (URL) | 跟踪分支 | 说明 |
| :--- | :--- | :--- | :--- |
| **`origin`** | `https://github.com/sjnhnp/homeproxy.git` | `master` | **我的专属主线仓库（Fork 改造版）**，固件 CI 编译时均直接引用此仓库。 |
| **`upstream`** | `https://github.com/immortalwrt/homeproxy.git` | `master` | **ImmortalWrt 官方上游主仓库**，提供官方新功能、新协议与 bug 修复。 |

---

## 2. 为什么不能直接盲目 pull 官方上游？

* **风险 1：Sing-box 1.15+ 启动死循环崩溃（Fatal Crash Loop）**  
  官方上游代码至今仍使用已在 Sing-box 1.14 标记废弃、1.15 完全移除的旧字段（如 `download_detour`）。若直接合并上游覆盖，会导致生成的 `config.json` 触发 Sing-box 致命报错并以退出码 0/1 闪退：
  ```text
  FATAL[0000] to continuing using this feature, set environment variable ENABLE_DEPRECATED_LEGACY_RULE_SET_DOWNLOAD_DETOUR=true
  ```
* **风险 2：TUN 模式被禁用或报错**  
  官方 `Makefile` 依赖项中缺少 `+ip-full` 与 `+kmod-tun`，若被覆盖，纯净编译或初次安装时会导致内核缺少 `tun.ko` 模块，或无法使用 `/usr/libexec/ip-full` 命令配置路由，使得 UI 界面无法勾选 TUN 模式。
* **风险 3：废弃参数导致语法警告与潜在冲突**  
  官方代码生成的配置包含废弃的 `store_rdrc`、`stack: 'system'` 以及 DNS 规则中的 `strategy`，严重影响 Sing-box 运行效率与健壮性。

因此，合并必须遵循 **“分支隔离 -> 比对差异 -> 强制保留现代化字段 -> 语义校验”** 的标准流程。

---

## 3. 核心自定义资产清单（绝不可被覆盖）

在合并任何上游提交时，必须严格保留以下核心改造：

| 文件路径 | 资产属性 | 核心改动内容与保留要求 |
| :--- | :--- | :--- |
| **`Makefile`** | **依赖声明** | **保留 `+kmod-tun` 与 `+ip-full`**：在 `LUCI_DEPENDS` 中明确声明依赖，保证固件编译或安装 `luci-app-homeproxy` 时自动补全内核 TUN 驱动与完整版 iproute2 工具。 |
| **`root/etc/homeproxy/scripts/generate_client.uc`** | **核心配置生成引擎** | **1. `http_client: { detour: ... }`**：所有远程规则集（内置与用户自定义）一律使用 1.14/1.15+ 标准的 `http_client.detour`，绝不可被 `download_detour` 覆盖。<br>**2. 原生 `sing-tun` 协议栈**：在 TUN 入站配置中，当系统协议栈被选择时，省略废弃的 `stack: 'system'`，直接启用 Sing-box 1.15 原生高性能 `sing-tun` 栈。<br>**3. 现代化 `cache_file`**：将已废弃的 `store_rdrc: true` / `rdrc_timeout` 重构为规范的 `store_fakeip: true, store_dns: true`。<br>**4. 现代 DNS 路由结构**：DNS 规则采用 `evaluate` 关联 `main-dns`，对 `geoip-cn` 采用 `match_response: true` 结合 `action: 'route'`；彻底剔除 DNS 规则 action 中已弃用的 `strategy: prefer_ipv6`。<br>**5. 自定义规则集空键清理**：对 `rule_set_ip_cidr_accept_empty` 与 `rule_set_ip_cidr_match_source` 进行判空过滤（`? true : null`），防止在 JSON 中生成无效空字段。 |
| **`root/etc/init.d/homeproxy`** | **双重兼容防线** | 在 procd 实例启动参数中，注入 `ENABLE_DEPRECATED_LEGACY_RULE_SET_DOWNLOAD_DETOUR=true` 与 `QUIC_GO_DISABLE_GSO=true`，作为底层双重兜底保险。 |

---

## 4. 核心网络与分流排错黄金准则（诊断经验库）

在配置与使用 HomeProxy 时，必须牢记以下核心网络机制，避免“看似连接成功实则打不开网页”：

1. **国内 DNS 投毒防范**：
   - 阿里公共 DNS（`223.5.5.5`）或运营商 DNS **只能用于解析国内域名（`geosite-cn`）**。
   - 绝不可将非中国域名发送给 `223.5.5.5`！否则 GFW 会针对海外域名实施 DNS 投毒，返回被封禁的虚假 IP（例如将 Facebook/Twitter 解析为受污染的 `104.244.42.197`），导致浏览器报 TLS 握手超时错误。
2. **海外 DNS 节点必须走代理出站（Detour）**：
   - 外网 DNS 服务器（如 `8.8.8.8`、Cloudflare DNS、自定义海外服务器）的 **出站节点（Outbound）必须指定为代理节点（如 `vless_hk` 等）**，绝对不能设置为 `Direct`（直连），否则其 DNS 查询请求会直接撞墙并被丢弃。
3. **优先使用 IPv4 DNS 策略**：
   - 建议在 HomeProxy 全局设置中将 **默认 DNS 策略设为 `IPv4 only`**。
   - 避免因本地运营商 IPv6 路由劣化或代理节点不支持 IPv6 时，每次网络请求都白白消耗 2 秒等待双栈超时。
4. **严禁本地 PC 客户端与路由器 TUN 双重嵌套**：
   - 当路由器开启 HomeProxy TUN 模式时，局域网电脑上的代理软件（如 GUI.for.SingBox、Clash Verge、v2rayN）**必须退出或设为 Direct 直连**。
   - 若本地电脑代理也处于开启状态，它发往香港 VPS 节点的流量会被路由器 TUN 再次捕获并二次丢入路由器代理节点隧道，造成“两层嵌套代理”，产生高达 6~9 秒的握手延迟乃至无法通信。

---

## 5. 未来指挥 AI 进行智能合并的提示词模版（一键复制）

在未来需要同步 ImmortalWrt 官方仓库时，可直接向 AI 发送以下指令：

````text
请帮我智能合并 HomeProxy 官方上游仓库的代码：

【基本信息】
1. 本地工作区：D:\ProgramData\Antigravity\homeproxy
2. 当前工作分支：master
3. 我的专属仓库 origin：https://github.com/sjnhnp/homeproxy.git
4. 官方上游仓库 upstream：https://github.com/immortalwrt/homeproxy.git（分支 master）

【执行原则与保护规范】
请严格查阅并遵守根目录下的 `UPSTREAM_MERGE_GUIDE.md`：
1. 必须保留 `Makefile` 中 `LUCI_DEPENDS` 新增的 `+kmod-tun` 与 `+ip-full`。
2. 必须保留 `root/etc/homeproxy/scripts/generate_client.uc` 中的 Sing-box 1.15+ 纯净无警告现代化语法：
   - 远程规则集必须使用 `http_client: { detour: ... }`，绝对不可倒退回 `download_detour`；
   - TUN 入站采用省略 `stack: 'system'` 的原生 `sing-tun` 架构；
   - cache_file 必须使用 `store_fakeip: true, store_dns: true`；
   - DNS 规则采用 `evaluate` 与 `match_response` 现代规范，清理废弃的 `strategy`；
   - 自定义规则集对空键进行过滤。
3. 请在临时隔离分支（如 sync-upstream-xxxx）中拉取 upstream 并执行三方合并。
4. 如遇代码冲突，必须严格按照指南优先保留现代化语法结构。
5. 合并后必须执行 `git grep` 验证，确认无任何废弃语法倒退，最后合并回 master 并推送到 origin。
````

---

## 6. 智能合并标准执行流程（Step-by-Step）

### 第一步：检查远端并获取官方最新变动

```bash
# 确认 remotes
git remote -v
# origin   https://github.com/sjnhnp/homeproxy.git (fetch & push)
# upstream https://github.com/immortalwrt/homeproxy.git (fetch & push)

# 获取官方 master 最新变动
git fetch upstream master
```

### 第二步：创建隔离合并分支

```bash
git checkout master
git pull origin master

# 创建隔离验证分支
git checkout -b sync-upstream-$(date +%Y%m%d)
```

### 第三步：发起三方合并

```bash
git merge upstream/master
```

---

## 7. 常见代码冲突排查与官方解法

### 冲突 1：`Makefile`
* **冲突点**：官方可能更新了 `LUCI_DEPENDS`。
* **解决原则**：保留官方新依赖的同时，**末尾必须保留 `+kmod-tun +ip-full`**。
```makefile
LUCI_DEPENDS:=+sing-box +ucode +ucode-mod-fs +ucode-mod-uci +kmod-tun +ip-full
```

### 冲突 2：`generate_client.uc` 中的规则集生成（内置与自定义）
* **冲突点**：官方如果更新了规则集，往往会继续写旧的 `download_detour`。
* **必须保留的现代语法**：
```javascript
// 内置规则集（约 840~880 行）
push(config.route.rule_set, {
	type: 'remote',
	tag: 'geoip-cn',
	format: 'binary',
	url: 'https://fastly.jsdelivr.net/gh/1715173329/IPCIDR-CHINA@rule-set/cn.srs',
	http_client: {
		detour: 'main-out'
	}
});

// 自定义规则集（约 940~965 行）
let ruleset_item = {
	type: cfg.type,
	tag: 'cfg-' + cfg['.name'] + '-rule',
	format: cfg.format,
	path: cfg.path,
	url: cfg.url,
	update_interval: cfg.update_interval
};
if (!isEmpty(outbound_detour)) {
	ruleset_item.http_client = {
		detour: outbound_detour
	};
}
push(config.route.rule_set, ruleset_item);
```

### 冲突 3：`generate_client.uc` 中的 TUN 与 cache_file
* **必须保留**：
```javascript
// TUN 入站（省略废弃的 stack: 'system'）
inbound_item.stack = (cfg.stack === 'mixed') ? 'mixed' : (cfg.stack === 'gvisor') ? 'gvisor' : null;

// cache_file 现代规范
config.experimental.cache_file = {
	enabled: true,
	path: '/etc/homeproxy/cache.db',
	store_fakeip: true,
	store_dns: true
};
```

冲突解决完毕后提交：
```bash
git add .
git commit -m "chore(merge): sync upstream immortalwrt and preserve sing-box 1.15 pure specs"
```

---

## 8. 合并后必查清单（Checklist）

在提交并推送到 GitHub 前，运行以下命令验证纯净度：

```bash
# 1. 检查是否存在已废弃的 download_detour（输出必须为空）
git grep "download_detour" root/etc/homeproxy/scripts/

# 2. 检查 http_client 是否已完全生效
git grep "http_client" root/etc/homeproxy/scripts/

# 3. 检查 Makefile 中是否包含 TUN 依赖
git grep "kmod-tun" Makefile
git grep "ip-full" Makefile

# 4. 检查是否仍存在废弃的 store_rdrc（输出必须为空）
git grep "store_rdrc" root/etc/homeproxy/scripts/
```

全部检查通过后，合并回 `master` 主分支并推送到远程：
```bash
git checkout master
git merge --ff-only sync-upstream-$(date +%Y%m%d)
git push origin master

# 删除临时分支
git branch -d sync-upstream-$(date +%Y%m%d)
```

---

## 9. 热更新到真机路由器调试验证

当个人仓库 `sjnhnp/homeproxy` 更新后，无需重新编译整个固件，即可在路由器后台 **【系统】->【TTYD 终端】** 中一键热替换测试：

```bash
# 1. 下载个人仓库最新脚本
wget -qO /etc/homeproxy/scripts/generate_client.uc https://raw.githubusercontent.com/sjnhnp/homeproxy/master/root/etc/homeproxy/scripts/generate_client.uc
wget -qO /etc/init.d/homeproxy https://raw.githubusercontent.com/sjnhnp/homeproxy/master/root/etc/init.d/homeproxy

# 2. 补齐执行权限并重启服务
chmod +x /etc/init.d/homeproxy /etc/homeproxy/scripts/generate_client.uc
/etc/init.d/homeproxy restart

# 3. 验证进程与日志
ps | grep sing-box
logread | grep homeproxy
```
