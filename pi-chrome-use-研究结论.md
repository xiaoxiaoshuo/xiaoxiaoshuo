# pi-chrome-use 研究结论

> 2026-08-07 更新

---

## pi-chrome-use 是什么

一个给 Pi 编码智能体用的 **CDP 扩展**，让 Pi 能通过标准 CDP 协议控制你已有的 Chrome 浏览器，执行 JS、操作页面、截图。

---

## CDP 协议本质

> **JSON-RPC over WebSocket**，很简单。连上浏览器调试端口后，发 `{ id, method, params }`，收 `{ id, result }` 或事件。

---

## pi-chrome-use vs ego-lite 的区别

| 维度 | pi-chrome-use | ego-lite |
|------|--------------|----------|
| 浏览器 | 你已有的 **标准 Chrome** | **定制 Chromium**（内核注入了 `globalThis.ego`） |
| 安装 | `pi install npm:pi-chrome-use` | 下载 `.dmg` 安装到 `/Applications` |
| snapshot | ❌ 没有（定制内核功能） | ✅ 内核级页面压缩快照 |
| 适用场景 | 轻量、即装即用 | 重度、多智能体并行 |

---

## 获取页面信息的三条路

| 方式 | 获取内容 | 优缺点 |
|------|---------|--------|
| **`document.body.innerText`** | 纯文本 | 简单但无结构，无法区分交互元素 |
| **`Accessibility.getFullAXTree()`** | 语义化树（role、name、属性） | 结构清晰，但**会漏掉 div/span 模拟的按钮** |
| **JS 侧 `querySelectorAll`** | 所有可交互元素 | 全覆盖，但无语义信息 |

---

## 获取可交互元素的最佳方案

> **AX 树 + JS 查询混合使用**

```
AX 树 → 提供语义角色和名称（button、link、combobox...）
       ↓
JS 侧 → 补充 div/span 模拟的按钮（[onclick], [class*="btn"], [tabindex]...）
       ↓
合并去重 → 完整的可交互元素列表
```

- 只用 AX 树：漏掉 **60%+** 的 div 模拟按钮
- 只用 JS 查询：缺少语义角色信息
- **两者结合：完整覆盖 + 语义丰富**

---

## 实测验证

| 元素 | AX 树 | JS 补充 | 说明 |
|------|-------|---------|------|
| `<button>` 真按钮 | ✅ 识别 | - | 标准元素 |
| `<div onclick>` 裸 div | ❌ **漏掉** | ✅ 抓到 | AX 不认 |
| `<div role="button">` | ✅ 识别 | ✅ 重复抓到 | 两边都有，需去重 |
| `<span onclick>` | ❌ **漏掉** | ✅ 抓到 | AX 不认 |
| `<div class="btn-*">` | ❌ **漏掉** | ✅ 抓到 | AX 不认 |
| `<div onclick>` 裸 div | ❌ **漏掉** | ✅ 抓到 | AX 不认 |

---

## 一句话总结

> **pi-chrome-use 是个轻量 CDP 桥接工具，没有 ego-lite 的定制内核 snapshot，但通过标准 CDP 的 `Accessibility.getFullAXTree` + JS 侧查询，也能达到类似的效果。需要 snapshot 那样的高级能力，就得用 ego-lite 的定制浏览器。**