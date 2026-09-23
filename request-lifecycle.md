# 一次请求的完整链路

把项目从「打开页面」到「看到数据」的整条路径串起来。

之前很多东西我是「会用但没完全理顺」：知道要配 proxy、知道有缓存、知道动画要用 transform，但说不清每一步到底是谁在负责。这次把散落的概念挂到同一条链路上，理解才顺过来。

---

## 一、浏览器发出请求

从输入地址到发出 HTTP 请求，中间经过：

```
DNS 解析（域名 → IP）
  → TCP 三次握手（建立连接）
  → TLS 握手（如果是 HTTPS）
  → 发送 HTTP 请求
```

HTTP 请求分成两部分：**请求头**（`Content-Type`、`Cookie` 等元信息）和**请求体**（实际携带的数据，比如下单时提交的 JSON）。

---

## 二、请求到达服务器：谁在转发

同一个 `/api/products`，开发环境和生产环境走的路径不一样：

```
开发环境：
浏览器 → Vite Dev Server（5173）→ 匹配到 /api → 转发到 localhost:3000 → Express

生产环境：
浏览器 → Nginx（80 / 443）→ 匹配到 /api/ → 转发到 127.0.0.1:3000 → Express
```

**Vite 本身就是开发阶段的一个 HTTP Server**，它不只是打包工具。所以请求 `/api/products` 时，请求是真的发到了 5173，只是 Vite 没有这个路由，通常返回 404 —— 不是「请求发不出去」。

如果前端代码直接写完整的 `http://localhost:3000/api/products`，请求能真正到达后端，但 5173 和 3000 是**不同的 origin**，会触发浏览器的同源策略，需要后端允许 CORS 才能拿到响应。

所以开发环境配 proxy，本质是让浏览器「以为」请求发给了同一个源。

Nginx 里那段配置：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;
}
```

这里的 `/api/` 匹配的是**请求 URL 的路径**，不是服务器上的文件夹。

另外 Nginx 是**双向代理**：

```
浏览器 → Nginx → Node
浏览器 ← Nginx ← Node
```

Node 处理完返回的 HTTP Response 也要先回到 Nginx，再由 Nginx 返回给浏览器。

---

## 三、Express 处理请求：`req.body` 是怎么来的

请求进入 Express 后走「中间件链」：

```
express.json()
  → 路由匹配
  → 鉴权中间件（通过就 next()，不通过就 401）
  → 路由处理函数
```

这里我之前一直没搞清楚 `req.body` 的来源。现在理解是三个不同的概念：

```
body     = HTTP 请求携带的数据
stream   = Node 读取这些数据的方式
req.body = Express 解析请求体后，提供给业务代码使用的 JS 数据
```

链路是这样：

```
HTTP Request Body
    ↓
Node 的 req（IncomingMessage，本质是一个 Readable Stream）
    ↓
读取请求体数据
    ↓
express.json()
    ↓
解析 JSON
    ↓
req.body
```

所以：**原生 Node 的 `req` 上没有 Express 意义上的 `req.body`**。HTTP 请求当然是有 body 的，只是 Node 不会直接给你一个已经解析好的对象——如果自己用原生 Node，需要监听 `data` / `end` 事件把数据读出来，再自己解析。`express.json()` 就是把这件事封装好的中间件。

中间件有三种走向：

- **放行**：调用 `next()`，交给下一个
- **拦截**：直接响应（比如 `res.status(401).json(...)`），后面的不再执行
- **抛错**：`next(err)`，交给错误处理中间件

---

## 四、返回响应：`res` 的几个方法

Express 的 `res` 是在 Node 原生 `ServerResponse` 之上封装的，提供了更方便的 API：

```
Express 的 res
    ↓
提供 json() / send() / status() 等便利 API
    ↓
底层 Node HTTP ServerResponse
    ↓
HTTP Response
    ↓
浏览器
```

`res.json(data)` 可以理解成：

```
JS 对象
  → 序列化成 JSON
  → 设置合适的 Content-Type
  → 写入 HTTP Response
  → 结束响应
```

而 Node 原生可以直接用 `res.end(...)` 写响应。**真正把响应通过网络发送出去的底层机制是 Node 的 HTTP 模块**，Express 只是在这之上提供了更好用的方法。

---

## 五、浏览器接收响应：缓存是谁在管

这块我之前一直没吃透，现在理解是分工问题：

```
服务器：通过 HTTP Response Header 告诉浏览器缓存规则
浏览器：按照这些规则决定是否保存、使用、去验证缓存
```

这些响应头可以由 Node / Express 设置，也可以由 Nginx 设置——**不是「缓存由 Nginx 做」或者「缓存由前端做」**。

### 强缓存

服务器返回：

```http
Cache-Control: max-age=3600
```

第一次请求会正常拿到数据；在 3600 秒内再次访问时：

```
浏览器检查本地缓存
    ↓
缓存没过期
    ↓
直接使用
```

**不会再发送 HTTP 请求。**

### 协商缓存

服务器返回：

```http
ETag: "abc123"
```

缓存需要验证时，浏览器会在请求里带上：

```http
If-None-Match: "abc123"
```

服务器判断资源有没有变化：

```
没变化 → 304 Not Modified → 浏览器继续用本地缓存
变化   → 200 + 新资源
```

区别记住这一句就够：

```
强缓存：不请求服务器
协商缓存：请求服务器做验证，但没变化时不重新下载资源
```

---

## 六、浏览器渲染

拿到 HTML 和 CSS 之后：

```
HTML → DOM
CSS  → CSSOM

DOM + CSSOM
    ↓
Render Tree（display: none 的元素在这一步被排除）
    ↓
Layout
    ↓
Paint
    ↓
Composite
```

### DOM 不等于屏幕上的像素

这是我这次自己想明白的一个点：**DOM 是文档结构，屏幕上看到的画面不是 DOM 本身**，而是 DOM + CSS 经过 Layout、Paint、Composite 之后画出来的结果。

所以一个元素被另一个元素盖住时，DOM 并没有消失，只是最终绘制出来的像素被覆盖了。动画也不是不断创建和删除 DOM——DOM 还是那个 DOM，变的是它最终怎么被画出来。

### 重排和重绘

不要理解成「改某个属性就一定重排」这种绝对规则，更准确的是：

```
修改 width / height
  → 可能影响元素尺寸和其他元素的位置
  → 需要重新 Layout
  → 后续可能需要 Paint / Composite

修改 background-color
  → 通常不影响布局
  → 通常不需要重新 Layout
  → 需要重新 Paint
```

### 为什么动画推荐用 transform / opacity

之前我的理解是「transform 只重新渲染动作的部分」，不太准确。更准确的说法是：

> `transform` / `opacity` 在很多情况下可以避免重新 Layout，也可以避免重新 Paint，主要在 **Composite 阶段**处理。

比如一个 div 用 `transform: translateX(...)` 移动时，DOM 仍然是同一个 DOM；浏览器可以复用已经 Paint 好的内容，只在 Composite 时改变它最终显示的位置。

所以核心原因是：**尽量避开昂贵的 Layout / Paint，让动画停留在相对便宜的 Composite 阶段。**

### 强制同步布局（Forced Synchronous Layout）

```
修改样式
    ↓
浏览器本来可以稍后统一做 Layout
    ↓
立刻读取 offsetWidth / offsetHeight / getBoundingClientRect()
    ↓
浏览器为了给出准确结果，可能被迫马上计算 Layout
```

所以「写 → 读 → 写 → 读」这种交替写法可能造成 layout thrashing，性能会变差。

---

## 七、JS 执行和渲染的关系

一次事件循环大致是：

```
宏任务
  ↓
微任务
  ↓
浏览器有机会渲染
  ↓
下一帧
```

`requestAnimationFrame` 的回调和浏览器下一次绘制的调度相关，不等于简单的 `setTimeout(fn, 16)`。

如果一段 JS 长时间占着主线程，浏览器就没机会及时渲染和响应用户输入。

---

## 八、MySQL

处理函数里通过连接池执行 SQL：

```js
pool.query('SELECT ... FROM products WHERE ...', [参数])
```

- **参数化查询**：SQL 用 `?` 占位、参数单独传 → 防止 SQL 注入
- **连接池**：复用连接，避免每次请求都新建

### 条件更新为什么能防超卖

```sql
UPDATE products
SET stock = stock - ?
WHERE id = ? AND stock >= ?
```

关键不在于「因为是行级锁所以不会超卖」，而是：

> **「检查库存」和「扣减库存」被放进同一条 UPDATE 语句里**，而不是先 SELECT 检查、再 UPDATE 扣减。

如果拆成两步，就会出现这个竞态窗口：

```
请求 A：SELECT 检查 → 库存够
请求 B：SELECT 检查 → 库存也够
A 扣一次
B 也扣一次   ← 超卖
```

写成一条语句后，并发的修改由数据库协调相关行，就不会出现这种「两个请求都以为库存够」的情况。

`affectedRows = 0` 表示这条 UPDATE 没有修改到任何行，通常说明库存条件没满足（也要考虑商品不存在等情况）。

### 事务和锁

```
事务 ≠ 锁

事务：保证一组数据库操作作为一个整体提交或回滚
锁：解决多个事务并发访问 / 修改相同数据时的协调问题
```

---

## 最后

```
浏览器加载前端资源
    ↓
Vue / JS 开始运行
    ↓
Axios / fetch 发 HTTP 请求
    ↓
开发环境：Vite proxy
生产环境：Nginx
    ↓
Express
    ↓
中间件
    ↓
路由处理
    ↓
MySQL
    ↓
HTTP Response
    ↓
Nginx / Vite 返回给浏览器
    ↓
浏览器处理响应
    ↓
Vue 更新响应式状态
    ↓
Vue 更新 DOM
    ↓
浏览器重新计算并绘制
    ↓
最终看到页面数据
```

这次主要是把以前项目里「会用但没完全理顺」的东西串起来。以后遇到 Vite、Axios、Express、Nginx、浏览器渲染这些概念，可以先放回这条链路里理解，而不是单独背 API。
