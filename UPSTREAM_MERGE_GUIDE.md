# HomeProxy 智能合并上游与保留 Sing-box 1.15+ 自定义规范指南

本文档专门用于指导如何安全、优雅地将 ImmortalWrt 官方上游（`immortalwrt/homeproxy:master`）的最新更新同步合并到本项目（`sjnhnp/homeproxy`），同时**完整保留我们针对 Sing-box 1.14 / 1.15+ 现代化规范的重构与兼容性补丁**。

---

## 目录
1. [核心架构与自定义资产清单](#1-核心架构与自定义资产清单)
2. [为什么不能直接盲目 pull 覆盖？](#2-为什么不能直接盲目-pull-覆盖)
3. [智能合并标准流程（Step-by-Step）](#3-智能合并标准流程step-by-step)
4. [核心文件冲突解法与标准比对](#4-核心文件冲突解法与标准比对)
5. [合并后一键自检命令](#5-合并后一键自检命令)
6. [热更新部署到路由器验证（免刷机）](#6-热更新部署到路由器验证免刷机)

---

## 1. 核心架构与自定义资产清单

在执行任何合并操作前，请先明确本项目中**必须永久保留的核心自研修改**：

| 文件路径 | 资产属性 | 核心作用与保留要求 |
| :--- | :--- | :--- |
| `root/etc/homeproxy/scripts/generate_client.uc` | **核心规则重构** | **替换废弃字段**：官方上游仍在使用已废弃的 `download_detour`，本项目已将其全面重构为 Sing-box 1.14/1.15+ 官方标准规范的 `http_client: { detour: '...' }`。**合并时绝对不能被上游的旧字段覆盖！** |
| `root/etc/init.d/homeproxy` | **守护进程双保险** | **注入兼容性环境变量**：在脚本顶部以及 `sing-box-c`（客户端）与 `sing-box-s`（服务端）服务实例中，注入了 `ENABLE_DEPRECATED_LEGACY_RULE_SET_DOWNLOAD_DETOUR=true`，确保任何历史遗留配置都不会触发 sing-box 内核 fatal 熔断。 |

---

## 2. 为什么不能直接盲目 pull 覆盖？

* **风险 1：Sing-box 1.15 闪退崩溃（Fatal Crash Loop）**  
  Sing-box 自 1.14.0 起废弃了 `download_detour`。如果盲目合并官方上游代码，会把旧的 `download_detour` 带回配置生成器，导致 sing-box 启动 0 秒直接报错退出：
  ```text
  FATAL[0000] to continuing using this feature, set environment variable ENABLE_DEPRECATED_LEGACY_RULE_SET_DOWNLOAD_DETOUR=true
  ```
* **风险 2：守护进程丢变量**  
  如果 `init.d/homeproxy` 被上游直接覆盖，缺少了 procd 环境变量声明，用户在添加第三方旧格式规则集时也会崩溃。

因此，必须遵循 **“创建隔离分支 -> 获取上游代码 -> 三方合并 -> 人工核对关键字段 -> 推送”** 的标准规范。

---

## 3. 智能合并标准流程（Step-by-Step）

### 第一步：检查并配置远程仓库

在 `D:\ProgramData\Antigravity\homeproxy` 目录下的终端中执行：

```bash
# 查看当前 remotes
git remote -v
```

确保包含以下两个远端（若无 `upstream` 则添加）：
```bash
# origin 指向你的个人 fork 仓库 (sjnhnp/homeproxy)
# upstream 指向 ImmortalWrt 官方仓库 (immortalwrt/homeproxy)
git remote add upstream https://github.com/immortalwrt/homeproxy.git 2>/dev/null || true

# 拉取上游最新变动
git fetch upstream master
```

### 第二步：创建升级隔离分支

切勿直接在 `master` 上盲目合并，建议创建带日期的隔离分支：

```bash
git checkout master
git pull origin master

# 创建隔离合并验证分支
git checkout -b sync-upstream-$(date +%Y%m%d)
```

### 第三步：执行合并

```bash
git merge upstream/master
```

此时会出现以下两种情况之一：
1. **自动合并成功（Auto-merging clean）**：上游改动与我们的关键修改不在同一行，Git 自动合并成功。
2. **发生代码冲突（CONFLICT）**：通常会标记在 `generate_client.uc` 或 `init.d/homeproxy`，此时转入 [第四章](#4-核心文件冲突解法与标准比对) 进行处理。

---

## 4. 核心文件冲突解法与标准比对

### 冲突点 1：`root/etc/homeproxy/scripts/generate_client.uc`

搜索代码中的冲突标记 `<<<<<<< HEAD`。

#### A. 默认规则集部分（约第 845~875 行）：
上游可能会增加新的规则，但**请务必确保每个 remote rule_set 使用的是 `http_client` 而非 `download_detour`**：

```javascript
// ✅ 正确规范（必须保留）：
	if (routing_mode === 'bypass_mainland_china') {
		push(config.route.rule_set, {
			type: 'remote',
			tag: 'geoip-cn',
			format: 'binary',
			url: 'https://fastly.jsdelivr.net/gh/1715173329/IPCIDR-CHINA@rule-set/cn.srs',
			http_client: {
				detour: 'main-out'
			}
		});
		push(config.route.rule_set, {
			type: 'remote',
			tag: 'geosite-cn',
			format: 'binary',
			url: 'https://fastly.jsdelivr.net/gh/1715173329/sing-geosite@rule-set-unstable/geosite-geolocation-cn.srs',
			http_client: {
				detour: 'main-out'
			}
		});
		push(config.route.rule_set, {
			type: 'remote',
			tag: 'geosite-noncn',
			format: 'binary',
			url: 'https://fastly.jsdelivr.net/gh/1715173329/sing-geosite@rule-set-unstable/geosite-geolocation-!cn.srs',
			http_client: {
				detour: 'main-out'
			}
		});
	}
```

#### B. 自定义规则集部分（约第 935~955 行）：
```javascript
// ✅ 正确规范（必须保留）：
	/* Rule set */
	uci.foreach(uciconfig, uciruleset, (cfg) => {
		if (cfg.enabled !== '1')
			return null;

		let outbound_detour = get_outbound(cfg.outbound);
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
	});
```

---

### 冲突点 2：`root/etc/init.d/homeproxy`

#### A. 顶部变量声明（约第 5~10 行）：
```bash
USE_PROCD=1
export ENABLE_DEPRECATED_LEGACY_RULE_SET_DOWNLOAD_DETOUR=true  # <- 确保这行存在
```

#### B. `sing-box-c` 与 `sing-box-s` 服务实例中（约第 155 行与 200 行）：
```bash
		# QUIC-GO GSO is broken on kernel 6.6 currently
		uname -r | grep -Eq "^6\.6" && procd_set_param env "QUIC_GO_DISABLE_GSO"="true"
		procd_set_param env "ENABLE_DEPRECATED_LEGACY_RULE_SET_DOWNLOAD_DETOUR"="true"  # <- 确保这行存在
```

---

## 5. 合并后一键自检命令

在提交合并前，在终端运行以下自检命令：

```bash
# 1. 检查是否还有任何遗留的旧字段（正常输出应为空）
git grep "download_detour" root/etc/homeproxy/scripts/

# 2. 检查现代化的 http_client 是否完整就绪（应输出对应代码行）
git grep "http_client" root/etc/homeproxy/scripts/

# 3. 检查守护进程环境变量是否就绪（应输出 3 处匹配）
git grep "ENABLE_DEPRECATED_LEGACY_RULE_SET_DOWNLOAD_DETOUR" root/etc/init.d/homeproxy
```

*若自检完全通过，即可完成合并并推送到 GitHub*：

```bash
# 提交合并改动
git add .
git commit -m "chore(merge): sync with upstream immortalwrt and preserve sing-box 1.15 specs"

# 合并回 master 并推送到你的仓库
git checkout master
git merge sync-upstream-$(date +%Y%m%d)
git push origin master

# 删除本地临时测试分支
git branch -d sync-upstream-$(date +%Y%m%d)
```

---

## 6. 热更新部署到路由器验证（免刷机）

一旦代码推送到 `sjnhnp/homeproxy`，无需重新编译或刷机，即可在路由器的 **【系统】 -> 【TTYD 终端】** 中一键热加载测试：

```bash
# 从你的个人分支直接拉取已合并的最新脚本
wget -qO /etc/homeproxy/scripts/generate_client.uc https://raw.githubusercontent.com/sjnhnp/homeproxy/master/root/etc/homeproxy/scripts/generate_client.uc
wget -qO /etc/init.d/homeproxy https://raw.githubusercontent.com/sjnhnp/homeproxy/master/root/etc/init.d/homeproxy

# 赋予权限并重启
chmod +x /etc/init.d/homeproxy /etc/homeproxy/scripts/generate_client.uc
/etc/init.d/homeproxy restart

# 检查进程状态（确保看到正常常驻）
ps | grep sing-box
```
