# PacketForge · 浏览器抓包 / 拦截 / 重放
PacketForge 是橘先生开发的一款 Chrome 扩展（v1.1.0），基于 Chrome DevTools Protocol（CDP） 的浏览器抓包工作台。它不依赖 Charles/mitmproxy 这类外部代理，装好扩展就能用。
简单说：想在不装代理、不碰证书的前提下，对浏览器流量做抓包、改包、重放和导出，这是目前最贴近"原生流量"的一条路。适合接口调试、安全测试和逆向分析场景。

> 作者：**橘先生**
> 版本：v1.1.0

基于 Chrome DevTools Protocol（CDP）的浏览器抓包工作台。不依赖外部代理，装完即用。
<img width="1920" height="977" alt="image" src="https://github.com/user-attachments/assets/e0843f3c-3eef-4cc8-9a2d-463d1f85efda" />
<img width="1920" height="298" alt="image" src="https://github.com/user-attachments/assets/6802c852-747e-4a66-9884-8178e3c6d02e" />
<img width="1920" height="522" alt="image" src="https://github.com/user-attachments/assets/5d2243fd-9230-4ec6-a3e3-a49c82c460c8" />
<img width="1920" height="903" alt="image" src="https://github.com/user-attachments/assets/c5e83db8-3848-4533-a250-6ba3e6ac2e43" />
<img width="1920" height="701" alt="image" src="https://github.com/user-attachments/assets/17820915-fa9c-4ce4-871f-39939babe22f" />

## 功能

| 能力         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| **实时抓包** | 完整抓取所有请求/响应，含请求体、响应体、真实 Cookie 头、重定向链、发起者调用栈 |
| **请求拦截** | 按 URL（子串 / 正则）+ 方法命中即挂起，可改 URL / 方法 / 头 / 体后放行 |
| **伪造响应** | 直接返回自定义状态码和响应体，请求不发给服务器               |
| **请求阻断** | 直接掐断请求                                                 |
| **历史记录** | IndexedDB 持久化，每标签页保留最近 5000 条，支持关键字/类型/状态过滤 |
| **一键重放** | 三种通道：CDP 页面上下文（推荐）/ 页面注入 / 后台 fetch。可改任意字段后重发 |
| **导出**     | HAR 1.2（可导入 Burp / Charles）、JSON、cURL 脚本、Python requests 代码 |
| **详情分析** | 请求头/响应头、JSON 格式化、图片预览、二进制 Hex 视图、CDP 时序 |

## 安装

1. 打开 `chrome://extensions/`
2. 右上角打开「开发者模式」
3. 点「加载已解压的扩展程序」→ 选择本目录（`packet-forge`）
4. 点扩展图标 → 「打开工作台」

> 需要 Chrome 116+（`chrome.debugger` 在 MV3 下稳定工作）。

## 使用

### 抓包

1. 工作台顶部选目标标签页 → 点「附加」
2. 目标页面上正常操作，请求会实时出现在列表
3. 点任意行看详情

**关于顶部的「正在调试此浏览器」横幅**：这是 `chrome.debugger` API 的强制行为，无法隐藏。点横幅上的「取消」等于分离。为了减少干扰，可以只在你关心的那个标签页上附加。

### 拦截改包

1. 点「规则」→ 添加匹配条件，例如：
   - 匹配方式「包含」，URL 填 `/api/user`
   - 匹配方式「正则」，URL 填 `^https://target\.com/api/`
2. 打开「拦截」开关
3. 命中规则的请求会被挂起，出现在底部「拦截队列」
4. 点「处理」，可以：
   - **修改后放行**：改方法/URL/头/体，再发给服务器
   - **返回伪造响应**：不发给服务器，直接返回你写的内容
   - **阻断请求**：直接掐断
   - **原样放行**：不动

挂起超过规则里的「超时」会自动放行，防止页面卡死。

### 重放

详情面板最后一个 tab「重放」：

- **通道选择**

  | 通道           | 什么时候用                                                   |
  | -------------- | ------------------------------------------------------------ |
  | CDP 页面上下文 | 默认。自动带页面 Cookie / Origin / Referer，不受页面 CSP 限制，最接近真实请求 |
  | 页面注入 fetch | 同 CDP 的 Cookie 行为，但受页面 CSP 的 `connect-src` 限制    |
  | 后台 fetch     | 不受 CORS 限制，能从扩展直接发跨域请求；但 Origin 是扩展源，且 Cookie 依赖浏览器策略 |

- 改完点「发送」，结果会显示在下方，同时作为一条 `Replay` 记录写进历史。

**重放指针**：如果服务端校验 `Origin` / `Referer` / 签名，用「CDP 页面上下文」通道——它跑在页面里，这些头由浏览器自动生成，跟你手点一样。

### 导出

右上「导出 ▾」：

- **HAR 1.2**：直接拖进 Burp / Charles / Chrome DevTools 都能读
- **JSON**：原始记录，含全部字段
- **cURL / Python**：生成可直接跑的脚本

单条记录也可以点详情右上角的 `cURL` / `Python` 复制。

## 架构

```
manifest.json          MV3 配置
bg/
  service-worker.js    CDP 会话、抓包事件、拦截队列、消息路由
  db.js                IndexedDB 封装（records / bodies 分离存储）
  replay.js            三通道重放实现
  har.js               HAR / cURL / Python 导出
panel/
  panel.html           工作台骨架
  panel.css            样式
  panel.js             虚拟列表、详情渲染、重放、拦截处理
popup/
  popup.html/js        快捷控制
```

### 为什么用 `chrome.debugger` 而不是 `webRequest`

MV3 的 `webRequest` 只能**读**请求头，拿不到请求体/响应体，也改不了任何东西。
`declarativeNetRequest` 只能改响应头，不能改 body。
真正能"看完整包 + 改包"的只有 CDP：

- `Network.*` 域 → 抓完整请求/响应（`getResponseBody` 拿 body）
- `Fetch.*` 域 → `requestPaused` 真正挂起请求，`continueRequest` / `fulfillRequest` / `failRequest` 决定它怎么走

## 反检测：网站能不能发现你在抓包

"被发现"要分三层看，三层的结论完全不同。

| 层                        | 谁能看见     | 本扩展的状态                                                 |
| ------------------------- | ------------ | ------------------------------------------------------------ |
| **网站 JS（前端反调试）** | 页面脚本     | ✅ **看不到**。CDP 不进页面、不 hook 任何函数、不改窗口尺寸   |
| **Chrome 浏览器本体**     | 本机用户肉眼 | ⚠️ **会看到**顶部「正在调试此浏览器」横幅，API 强制行为       |
| **服务端 / 网络中间设备** | 远程         | ✅ **看不到**。请求仍由浏览器自己发出，TLS/HTTP2 指纹与不抓包时完全一致 |

### 为什么前端检测抓不到

常见的反调试套路和它们的失效原因：

| 检测手段                           | 原理                                          | 为什么失效                                  |
| ---------------------------------- | --------------------------------------------- | ------------------------------------------- |
| `outerWidth - innerWidth > 160`    | 开 DevTools 面板会挤占窗口                    | 不开面板，窗口尺寸不变                      |
| `debugger;` 计时                   | DevTools 打开时命中断点，时间差暴露           | 本扩展**不启用 Debugger 域**，语句不暂停    |
| `console.log` 对象 getter          | 打开 Console 会让 CDP 序列化对象、触发 getter | 本扩展**不启用 Runtime 域**，不上报 console |
| `Function.prototype.toString` 检测 | 检查函数是否被代理包裹                        | 本扩展不修改任何页面函数                    |
| 检测 `window.devtools` 等特征      | 依赖 DevTools 注入的全局对象                  | CDP 不注入任何全局对象                      |

当前实现**只启用 `Network` 一个域**，这是纯被动监听，页面侧没有任何可观测变化。

顺带一提：这比开 DevTools 抓包更隐蔽。DevTools 会触发上面几乎全部检测手段，而 CDP 一个都不触发。

### CDP 协议层检测（反爬系统的头号武器）

搜索反爬资料后发现的、专门针对 **CDP 抓包本身**的检测，比上面的 DevTools 检测更难对付：

| 检测手段                        | 原理                                                         | 本扩展                                        |
| ------------------------------- | ------------------------------------------------------------ | --------------------------------------------- |
| `Runtime.enable` 上下文泄漏     | Cloudflare / DataDome 等检测 `executionContextCreated` 事件——正常浏览永不触发，只有 CDP 客户端 `Runtime.enable` 才产生 | ✅ 免疫，从不调用 `Runtime.enable`             |
| `sourceURL` 注入                | Puppeteer/Playwright 注入的脚本带 `//# sourceURL=pptr://...` 注释可被溯源 | ✅ 免疫，重放用内联表达式，无 sourceURL        |
| 扩展 WAR 指纹                   | 网页 `fetch('chrome-extension://<id>/<资源>')` 探测扩展是否安装 | ✅ 免疫，未声明任何 `web_accessible_resources` |
| isolated world / content script | 检测注入的 content script 或孤立世界、`MutationObserver` 监视 DOM 篡改 | ✅ 免疫，无 content script，不碰 DOM           |

### 调试端口探测（外部 CDP 模式独有的风险）

扩展模式（`chrome.debugger`）**不开放任何本地端口**，网站无从探测。

但若改用外部 CDP（`--remote-debugging-port=9222`）启动浏览器，网页可以用 `fetch`（no-cors 模式）、WebSocket、或计时探测 `localhost:9222` 是否开放：

- **绝不加 `--remote-allow-origins`** —— 否则 `/json` 端点直接把 tab 列表暴露给网页
- Chrome 自带的 Host 校验能挡住 DNS rebinding 读取内容，但**挡不住「端口是否开放」这个信号**
- 结论：**要隐蔽就用扩展模式；外部 CDP 模式天然多一个端口信号**

### 后端能检测到什么

服务端和中间设备能观测的只有传输层与行为层。CDP 模式下抓包**完全透明**：

| 检测手段            | 原理                                       | 本扩展                             |
| ------------------- | ------------------------------------------ | ---------------------------------- |
| TLS 指纹（JA3/JA4） | ClientHello 的扩展顺序、加密套件、椭圆曲线 | ✅ 完全原生，请求由 Chrome 自己发出 |
| HTTP/2 指纹         | SETTINGS 帧、初始窗口大小、优先级树        | ✅ 完全原生，未做任何改写           |
| 请求头顺序          | 不同 HTTP 库有固定排序特征                 | ✅ 原生顺序，抓包不干预             |
| `sec-fetch-*` 系列  | 浏览器自动生成，手工伪造必然露馅           | ✅ 浏览器自动生成，值正确           |
| 连接复用 / 出口 IP  | 挂代理会导致 IP 跳变、连接池重建           | ✅ 走本机浏览器网络栈，IP 不变      |
| 证书 pinning        | 校验服务端证书是否被中间人替换             | ✅ **不适用**，见下                 |

### 证书检测：为什么不是问题

这是最容易误解的一点。**CDP 抓包根本不存在证书环节**：

- **代理方案**（Charles / mitmproxy / Fiddler）：必须给客户端装根证书、由代理重新签发目标站证书 → 目标一旦做 pinning 或校验证书链，立刻暴露
- **CDP 方案**：抓到的数据是浏览器**在正常完成 TLS 握手之后**从内存里读出来的。握手、证书链、OCSP、CT 日志一个都没被碰过

所以「绕过证书检测」对这套方案是个伪命题——**没有要绕的东西**。

真正会遇到证书问题的是**非浏览器目标**（App、IoT、桌面客户端）。那些做了 pinning 的流量，浏览器扩展从原理上就抓不到，需要 Frida / objection 这类运行时 hook 工具。这不在本扩展的职责范围内，也不是加个选项能解决的。

### 隐蔽性自检面板

工具栏的「隐蔽性」按钮会实时拉取当前会话的评估，逐条列出每个检测向量的状态：

- 🟢 **不可检测** — 页面和服务器都拿不到信号
- 🔵 **仅本机可见** — 比如调试横幅，只有你自己能看到
- 🟡 **远程可测** — 比如开启拦截后的延迟，页面能用计时器测出来

附加前、附加后、开关拦截时，评级会实时变化。动手前先看一眼，比事后猜强。

### 实测数据

用 Chrome（headless=new）跑了三组对照，同一个检测页面，只改 CDP 附加方式：

```
检测项                   A 不附加    B 仅Network   C 加Runtime
--------------------------------------------------------------
debugger 暂停耗时         0 ms        0 ms          0 ms
console getter 被触发    否          否            否
窗口尺寸差               18 px       18 px         18 px
toString 保持原生        是          是            是
定时器抖动               12 ms       12 ms         12 ms
```

A 与 B 完全一致 —— **只开 Network 域时，前端反调试拿不到任何差异信号**。

C 组（额外开 Runtime）在本次测试中**没有**复现出稳定差异（第一次测到 33ms 抖动，复测回落到 12ms，判定为噪声）。所以准确说法是：`Runtime.enable` 是可避免的攻击面，移除它没有功能损失，但"移除后能防住某个具体检测"这件事本机没有测出来。不过度声称。

如果想复现，`stealth-test.mjs` 的逻辑可以直接改造复用。

### 只有一个破绽：顶部横幅

`chrome.debugger.attach()` 必然让 Chrome 显示「XX 正在调试此浏览器」。这是浏览器给用户的知情权提示，**没有任何 manifest 配置、权限声明或 API 能关掉它**。

如果要连这行横幅都消掉，只有一条路——**不走扩展的 debugger API，改用外部 CDP**：

```powershell
# 用调试端口启动 Chrome（会使用独立的用户目录，不影响日常配置）
"C:\Program Files\Google\Chrome\Application\chrome.exe" `
  --remote-debugging-port=9222 `
  --user-data-dir="C:\chrome-debug-profile"
```

这样启动的 Chrome **没有任何横幅**，外部工具（自写脚本 / Puppeteer / 本扩展的"外部 CDP 模式"）连 `ws://127.0.0.1:9222` 就能拿到同样的能力。代价是要单独启动一个浏览器实例。

### 唯一会露馅的操作：开启拦截

抓包是纯被动的，但**开启拦截**会让每个请求多一次往返延迟，页面能测出来（比如用一个 `performance.now()` 计时 fetch）。所以：

- 只抓包不改包 → 全程无感
- 需要改包 → 建议对人少的目标或自己的测试账号操作，抓完就关掉拦截

## 抓不到包的常见原因

| 现象                          | 原因                                                     | 处理                                              |
| ----------------------------- | -------------------------------------------------------- | ------------------------------------------------- |
| 整个标签页没记录              | 没点「附加」，或已被 DevTools 占用                       | 关掉 DevTools 重新附加                            |
| 只有页面主请求，接口全缺      | 接口在 Service Worker / Web Worker 里发                  | 已通过 `Target.setAutoAttach` 支持，v1.0.1 起生效 |
| 某个接口缺 `chrome://` 内部页 | 浏览器内部页不支持 attach                                | 无解                                              |
| 有记录但响应体为空            | 请求来自缓存、或 SW 直接返回                             | 看 `resBodyError` 字段                            |
| 完全连不上                    | 目标用了非 HTTP 协议（如 QUIC 以外的私有协议、App 内置） | 不在浏览器抓包范围                                |

## 已知限制

1. **必须点「附加」**：CDP 需要显式 attach，无法像代理一样全局透明抓包。
2. **顶部调试横幅**：Chrome 强制显示，无法去除。
3. **`chrome://` 页面抓不了**：浏览器内部页不支持 debugger attach。
4. **响应体上限**：单条 body 最多存 2 MB，超出截断（`resBodyTruncated` 标记）。
5. **`Fetch.enable` 有性能开销**：开启拦截时每个请求都会走一次 SW 往返，页面会略慢。不需要改包时把拦截关掉。
6. **预检请求**：`OPTIONS` 如果被规则命中也会挂起，挂太久会导致 CORS 预检失败。规则写精确一点。
7. **Service Worker 休眠**：附加期间有保活心跳，但浏览器长时间空闲仍可能回收；此时重新点「附加」即可。
8. **重放不带 TLS 指纹伪装**：走的是浏览器自己的网络栈，指纹就是 Chrome 的（这反而是好事——与真实用户完全一致）。
9. **重放的通道差异**（唯一可能露馅的地方）：
   - `CDP 页面上下文`：`Origin` / `Referer` / `Cookie` / `sec-fetch-*` 全由浏览器按页面身份生成，与真实请求一致
   - `后台 fetch`：`Origin` 变成 `chrome-extension://...`，`sec-fetch-*` 也不同。服务端若校验这些头，会直接拒绝或返回风控页
   - 所以**重放优先用 CDP 通道**，除非你确定目标不校验 Origin

## 扩展方向

- WebSocket 帧抓取（`Network.webSocketFrameSent/Received`）
- 响应阶段拦截与改包（`Fetch` 的 `requestStage: "Response"`）
- 请求断点条件（按响应状态、耗时、大小）
- 录制/回放完整用户会话
- 与 Burp 双向同步
  
## 创作不易 感谢支持
<img width="300" height="300" alt="87b07bb227a3f3981109124e0ecdb0fc" src="https://github.com/user-attachments/assets/d36886ac-d8d0-492a-b461-5614250ef67d" />
<img width="300" height="300" alt="7030e4f4e62192f41a5f522b70b8596a" src="https://github.com/user-attachments/assets/a86d539a-67e1-4252-bc1a-28d24128f55c" />
