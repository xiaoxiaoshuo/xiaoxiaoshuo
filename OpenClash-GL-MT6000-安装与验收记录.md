# GL-MT6000 安装 OpenClash 与 DSH 访问排障记录

> 初次记录：2026-10-18  
> 更新记录：2026-10-24（导入多份配置、切换 clasv2、修复 DSH web_fetch 的 Fake-IP 问题）  
> 设备：GL.iNet GL-MT6000（Flint 2）  
> 路由器固件：GL.iNet 4.9.1 / OpenWrt 21.02-SNAPSHOT / kernel 5.4.238 / aarch64_cortex-a53  
> 操作方式：已登录的 Chrome（CDP 远程调试）+ 路由器 SSH

---

## 一、目标

1. 在 GL-MT6000 上安装 OpenClash，补齐依赖，导入订阅并启用透明代理。
2. 排查 Mac 浏览器无法访问 Google 的问题。
3. 导入 Clash Verge 中的其他配置，并支持切换。
4. 修复 DSH `web_fetch` 因 Fake-IP 导致 SSRF 校验失败的问题。

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
9. 通过 OpenClash 的订阅机制导入「一分机场」订阅（UA 使用 `clash.meta`）。
10. 设置配置文件路径并启用 OpenClash，启动服务。
11. 验证代理端口、出口 IP、DNS 和常见网站访问。
12. 排查 Mac 浏览器：定位到 `/etc/hosts` 把 `www.google.com` 指向 `127.0.0.1`，删除对应条目并刷新 DNS。
13. 在 Chrome 中完成 Google 首页、Google 搜索、Gmail、YouTube、GitHub 的访问验收。
14. 从 Clash Verge 直接复制另外三份配置到路由器，导入 OpenClash。
15. 切换到 `clasv2.yaml`，验证代理。
16. 将 OpenClash 从 Fake-IP 切换为 Redir-Host，修复 DSH `web_fetch` 的 SSRF 校验问题。

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

- `config_path=/etc/openclash/config/clasv2.yaml`（当前启用）
- `enable=1`
- `core_type=Meta`
- `core_version=linux-arm64`
- `en_mode=redir-host`
- `operation_mode=redir-host`
- `proxy_mode=rule`（规则模式，不是全局模式）
- DNS 端口：`7874`
- 控制面板端口：`9090`
- 「一分机场」订阅段：
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

### 3.4 配置与生成文件

配置目录：`/etc/openclash/config/`

| 文件 | 大小 | 来源/说明 |
|---|---|---|
| `clasv2.yaml` | 约 1.4 KB | Clash Verge 本地配置，当前启用 |
| `yfjc.yaml` | 约 46.3 KB | 一分机场订阅 |
| `edgetunnel.yaml` | 约 699.9 KB | edgetunnel 远程订阅 |
| `tested_within.yaml` | 约 716.2 KB | tested_within 远程订阅，已修复 1 个缺 `cipher` 的节点 |

其他文件：

- `/etc/openclash/clasv2.yaml`：当前运行配置（OpenClash 启动时生成）
- `/etc/openclash/yfjc.yaml`：之前运行配置
- `/etc/openclash/core/clash_meta`：mihomo 内核（约 12.8 MB）
- `/tmp/openclash.log`：运行日志（临时）
- `/root/tested_within.yaml.orig`：`tested_within.yaml` 原始文件备份
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

### 3.7 DSH `web_fetch` / Fake-IP 问题修复

**问题现象**

- OpenClash 原为 `fake-ip` 模式，DNS 返回 `198.18.x.x`
- 浏览器走代理，代理内部知道真实 IP，所以浏览器能正常访问网站
- DSH 的 `web_fetch` 会做真实 DNS 解析 + SSRF 校验，看到 `198.18.x.x` 后判定为内网/保留地址，导致所有站点访问失败

**修复方式**

1. 将 OpenClash 全局模式改为 `redir-host`：
   ```bash
   uci set openclash.config.en_mode='redir-host'
   uci set openclash.config.operation_mode='redir-host'
   uci commit openclash
   /etc/init.d/openclash restart
   ```
2. 修改当前 `clasv2.yaml` 的 DNS 配置：
   - `enhanced-mode: redir-host`
   - 国内域名使用 `doh.pub` / `dns.alidns.com`
   - 国外域名通过代理组 `#Proxy` 查询 `1.1.1.1` / `8.8.8.8` 的 DoH，避免国内 DNS 污染
   - 增加 `fallback-filter`、`proxy-server-nameserver`，解决用代理查询 DNS 的依赖问题
3. 刷新 Mac DNS 缓存：
   ```bash
   dscacheutil -flushcache
   killall -HUP mDNSResponder
   ```

**验证结果**

DNS 已返回真实公网 IP，不再出现 `198.18.x.x`：

| 域名 | 返回示例 |
|---|---|
| `www.google.com` | `142.251.156.119` 等 |
| `www.youtube.com` | `142.251.150.4` 等 |
| `chatgpt.com` | `172.64.155.209` / `104.18.32.47` |
| `www.baidu.com` | `39.156.70.239` 等 |
| `example.com` | `172.66.147.243` 等 |

代理仍正常：

- 出口 IP：`101.32.98.238`
- 位置：日本东京 / 腾讯云（AS132203）
- Google / YouTube / GitHub / 百度均访问正常

> 说明：规则模式/全局模式不是本次问题的关键；关键在 DNS 的 `enhanced-mode`。当前仍为规则模式：国内直连，国外走 `Proxy`。

---

## 四、Mac 上的改动

### 4.1 Google 访问问题定位

Mac 的 `/etc/hosts` 中存在：

```
127.0.0.1    www.google.com
127.0.0.1    www.google.com
```

Chrome 因此把 `www.google.com` 解析到本机，访问时出现：

```
ERR_CONNECTION_REFUSED
```

这不是路由器代理的问题。使用 `curl --resolve` 绕过 hosts 指向路由器 DNS 时，Google 可以正常返回。

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
- 当前配置：`clasv2.yaml`
- 节点：`vless-31170`
- 出口 IP：`101.32.98.238`（日本东京 / 腾讯云）
- DNS 模式：`redir-host`
- 透明代理：TCP/UDP 均已接管
- DNS：返回真实公网 IP，无 `198.18.x.x`

### 5.2 浏览器侧（Chrome）

- Google 首页：正常打开（跳转到 `google.com.hk`）
- Google 搜索：正常返回结果
- Gmail：正常跳转登录页
- YouTube：正常打开
- GitHub：正常访问
- 百度等国内站点：正常

### 5.3 DSH

- DNS 层根因已消除：DSH 再做真实 DNS 解析时会拿到真实公网 IP，不再被 SSRF 校验拦截
- 需要**重启 DSH**（或其容器/服务）清理旧 DNS 缓存
- 若仍失败，请在 DSH 环境执行 `dig www.google.com`：
  - 返回 `198.18.x.x`：说明 DSH 没走路由器 DNS，或自身有独立 DNS 缓存
  - 返回 `142.251.x.x` 等真实 IP：SSRF 校验应可通过

---

## 六、访问入口与密码说明

### 6.1 访问入口

- GL 后台：`http://192.168.8.1`
- LuCI：`http://192.168.8.1:8080/cgi-bin/luci/`
- OpenClash：LuCI → 服务 → OpenClash
- 说明：不要使用 80 端口的 LuCI 进行 OpenClash 配置，80 端口的 `/ubus` 被 GL 后台接管

### 6.2 密码说明（重点）

**GL 后台管理员密码 = LuCI 的 root 密码 = SSH 的 root 密码，是同一个密码。**

- GL 后台登录：使用管理员密码
- LuCI 登录：用户名 `root`，密码使用同一个管理员密码
- SSH 登录：用户 `root`，密码同样使用这个管理员密码
- 本次操作**没有修改**管理员密码
- LuCI 出现「需要授权」只是因为登录会话过期，重新用 `root` + 管理员密码登录即可

**以下密码不是管理员密码：**

- OpenClash 控制面板密码：由 OpenClash 自动生成，独立于管理员密码
- OpenClash 代理端口认证（HTTP/SOCKS）：由 OpenClash 自动生成，独立于管理员密码
- 上述密码存放在 `/etc/config/openclash`，可在 LuCI/OpenClash 设置中查看或修改

---

## 七、未执行/注意事项

- 未升级路由器固件
- 未修改路由器管理员密码
- 未开启订阅自动更新（可在 OpenClash「配置订阅」中手动更新或开启自动更新）
- 未删除 SSH 公钥
- 未修改 Clash Verge 配置
- 未清理 `/etc/hosts` 中的 `github.com` 固定 IP 条目
- `.pi/` 与 `node_modules/` 未提交到 Git
- 当前 `clasv2.yaml` 已是 Redir-Host DNS；如果切换到其他配置，建议确认其 DNS/fallback 是否也适合 Redir-Host

---

## 八、产物与回滚

### 8.1 备份文件

路由器：

- `/root/openclash.uci.20261024-181341.bak`：切换到 `clasv2` 前的 UCI 备份
- `/root/clasv2.runtime.20261024-181341.bak`：切换前运行配置备份
- `/root/clasv2.source.20261024-181605.bak`：Redir-Host DNS 修改前的源配置备份
- `/root/tested_within.yaml.orig`：`tested_within` 原始文件备份

Mac：

- `/etc/hosts.bak.20261018-220805`

### 8.2 恢复 Mac hosts

```bash
sudo cp /etc/hosts.bak.20261018-220805 /etc/hosts
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

### 8.3 切换配置

```bash
# 切换到 edgetunnel
uci set openclash.config.config_path='/etc/openclash/config/edgetunnel.yaml'
uci commit openclash
/etc/init.d/openclash restart

# 切换回一分机场
uci set openclash.config.config_path='/etc/openclash/config/yfjc.yaml'
uci commit openclash
/etc/init.d/openclash restart
```

### 8.4 回滚 Fake-IP

```bash
uci set openclash.config.en_mode='fake-ip'
uci set openclash.config.operation_mode='fake-ip'
uci commit openclash
/etc/init.d/openclash restart
```

### 8.5 停用/卸载 OpenClash

```bash
# 停用
uci set openclash.config.enable='0'
uci commit openclash
/etc/init.d/openclash stop

# 卸载
opkg remove luci-app-openclash
```

> 卸载前建议先在 OpenClash 中停止服务，并确认 DNS/防火墙已恢复。

---

## 九、Git 记录

- 主分支 `main`：`3e6834a docs: 记录 GL-MT6000 OpenClash 安装与验收`
- 本地改动备份分支 `backup/local-changes`：`5d4b096 docs: 保存本地文档与项目配置改动`
- `.gitignore` 已排除 `.pi/` 与 `node_modules/`
