# GL-MT6000 安装 OpenClash 与 Google 访问排障记录

> 记录时间：2026-10-18  
> 设备：GL.iNet GL-MT6000（Flint 2）  
> 路由器固件：GL.iNet 4.9.1 / OpenWrt 21.02-SNAPSHOT / kernel 5.4.238 / aarch64_cortex-a53  
> 操作方式：通过已登录的 Chrome（CDP 远程调试）+ 路由器 SSH

---

## 一、目标

在 GL-MT6000 上安装 OpenClash，补齐依赖，导入订阅并启用透明代理；随后排查 Mac 浏览器无法访问 Google 的问题。

---

## 二、操作流程

1. 以默认个人资料启动 Chrome，开启 `9222` 远程调试端口，通过 CDP 接管浏览器。
2. 确认路由器型号、固件、OpenWrt 版本、内核、CPU 架构和存储空间。
3. 在 GL 后台「系统 → 高级设置」安装 LuCI。
4. 确认可用的 LuCI 在 **8080** 端口；80 端口的 LuCI 因 `/ubus` 被 GL 后台 SPA 接管而无法正常使用。
5. 通过 LuCI/ubus 将本机 SSH 公钥写入 `/etc/dropbear/authorized_keys`，后续使用 SSH 完成安装。
6. 下载 `luci-app-openclash_0.48.002_all.ipk`，上传到路由器并安装。
7. 安装缺失依赖。
8. 下载 mihomo `linux-arm64` 内核，安装到 OpenClash 核心目录。
9. 通过 OpenClash 的订阅机制导入订阅配置（UA 使用 `clash.meta`）。
10. 设置配置文件路径并启用 OpenClash，启动服务。
11. 验证代理端口、出口 IP、DNS 和常见网站访问。
12. 排查 Mac 浏览器：定位到 `/etc/hosts` 把 `www.google.com` 指向 `127.0.0.1`，删除对应条目并刷新 DNS。
13. 在 Chrome 中完成 Google 首页、Google 搜索、Gmail、YouTube、GitHub 的访问验收。

---

## 三、路由器上的改动

### 3.1 安装的软件

- `luci`（通过 GL 后台「高级设置」安装）
- `luci-app-openclash 0.48.002`
- 依赖包：
  - `bash`
  - `ca-bundle`
  - `ca-certificates`
  - `ip-full`
  - `ruby`
  - `ruby-yaml`
  - `coreutils-nohup`
  - `kmod-inet-diag`
  - `libcap-bin`
- 复用系统已有依赖：
  - `dnsmasq-full 2.92`
  - `ipset`
  - `iptables-mod-tproxy`
  - `kmod-tun`
  - `kmod-ipt-ipset`
  - 等

### 3.2 OpenClash 内核

- 类型：mihomo（Clash Meta）
- 版本：`alpha-gbc76c13 linux arm64`
- 路径：`/etc/openclash/core/clash_meta`
- 已通过 `clash_meta -t` 配置校验

### 3.3 关键配置

文件：`/etc/config/openclash`

- `config_path=/etc/openclash/config/yfjc.yaml`
- `enable=1`
- `core_type=Meta`
- `core_version=linux-arm64`
- 代理模式：`fake-ip`
- DNS 端口：`7874`
- 控制面板端口：`9090`
- 新增订阅段：
  - 名称：`yfjc`
  - User-Agent：`clash.meta`
  - `sub_convert=0`
  - `enabled=1`
  - 订阅 URL 的 token 已写入该 UCI 配置，本文档不展开

文件：`/etc/config/dhcp`

- OpenClash 添加了 `list server '127.0.0.1#7874'`，把 dnsmasq 的上游 DNS 指向 Clash

SSH：

- 在 `/etc/dropbear/authorized_keys` 中添加了本机 ed25519 公钥
- 按用户要求保留，未删除

### 3.4 生成的文件

- `/etc/openclash/config/yfjc.yaml`：订阅原始配置（约 46KB）
- `/etc/openclash/yfjc.yaml`：OpenClash 启动时生成的运行配置（约 49KB）
- `/etc/openclash/core/clash_meta`：mihomo 内核（约 12.8MB）
- `/tmp/openclash.log`：运行日志（临时）
- OpenClash 自动生成的代理认证与面板密码，存放于 `/etc/config/openclash`

### 3.5 运行时网络改动

- iptables 新增链：
  - `openclash`
  - `openclash_output`
  - `openclash_upnp`
- TCP 透明代理重定向到 `7892`
- UDP TPROXY 到 `7895`
- DNS 劫持/转发到 `7874`
- 监听端口：
  - `7890` HTTP
  - `7891` SOCKS
  - `7892` Redirect
  - `7893` Mixed
  - `7895` TProxy
  - `7874` DNS
  - `9090` 控制面板

### 3.6 LuCI 菜单修复

安装 OpenClash 后，LuCI 左侧菜单一度不显示「服务 → OpenClash」，处理方式：

- 清理路由器上的 `/tmp/luci-indexcache.*`
- 给当前 LuCI 会话补上 `luci-app-openclash` 的 ACL
- 清除 Chrome 中 3 个 LuCI 标签页的 `sessionStorage` 菜单缓存

处理后菜单正常显示。

---

## 四、Mac 上的改动

### 4.1 问题定位

Mac 的 `/etc/hosts` 中存在：

```
127.0.0.1    www.google.com
127.0.0.1    www.google.com
```

Chrome 因此把 `www.google.com` 解析到本机，访问时出现：

```
ERR_CONNECTION_REFUSED
```

这不是路由器代理的问题。使用 `curl --resolve` 绕过 hosts 指向路由器 fake-IP 时，Google 可以正常返回。

### 4.2 修复

- 删除 `/etc/hosts` 中两条 `127.0.0.1 www.google.com`
- 原文件备份：`/etc/hosts.bak.20261018-220805`
- 刷新 macOS DNS 缓存：
  - `dscacheutil -flushcache`
  - `killall -HUP mDNSResponder`

### 4.3 未改动的部分

- 未修改系统代理（检查时系统代理已关闭）
- 未修改 Clash Verge 配置（其仍在 `127.0.0.1:7897` 监听，但未作为系统代理）
- 保留了 `/etc/hosts` 中的 `github.com` 固定 IP 和 `localhost` 映射

---

## 五、验收结果

### 5.1 路由器侧

- OpenClash：运行中
- 开机自启：已启用
- 出口 IP：`203.10.98.186`（日本节点）
- 透明代理：TCP/UDP 均已接管
- DNS：fake-ip 模式正常工作

### 5.2 浏览器侧（Chrome）

- Google 首页：正常打开（跳转到 `google.com.hk`）
- Google 搜索：正常返回结果
- Gmail：正常跳转登录页
- YouTube：正常打开
- GitHub：正常访问
- 百度等国内站点：正常

---

## 六、访问入口

- LuCI：`http://192.168.8.1:8080/cgi-bin/luci/`
- OpenClash：LuCI → 服务 → OpenClash
- 说明：不要使用 80 端口的 LuCI 进行 OpenClash 配置，80 端口的 `/ubus` 被 GL 后台接管

---

## 七、未执行/注意事项

- 未升级路由器固件
- 未修改路由器管理员密码
- 未开启订阅自动更新（可在 OpenClash「配置订阅」中手动更新或开启自动更新）
- 未删除 SSH 公钥
- 未修改 Clash Verge 配置
- 未清理 `/etc/hosts` 中的 `github.com` 固定 IP 条目

---

## 八、回滚方式

### 8.1 恢复 Mac hosts

```bash
sudo cp /etc/hosts.bak.20261018-220805 /etc/hosts
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

### 8.2 停用 OpenClash

```bash
uci set openclash.config.enable='0'
uci commit openclash
/etc/init.d/openclash stop
```

### 8.3 卸载 OpenClash

```bash
opkg remove luci-app-openclash
```

> 卸载前建议先在 OpenClash 中停止服务，并确认 DNS/防火墙已恢复。
