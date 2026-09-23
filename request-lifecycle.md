# 一次请求的完整链路

把项目从「打开页面」到「看到数据」的整条路径串起来。

好处是：很多零散的知识点（跨域、代理、缓存、渲染、中间件）都能挂到这条链路上，不用单独去背。

---

## 一、浏览器发出请求

输入地址到发出 HTTP 请求，中间大概经历：

```
DNS 解析（域名 → IP）
  → TCP 三次握手（建立连接）
  → TLS 握手（如果是 HTTPS）
  → 发送 HTTP 请求（方法、路径、请求头、请求体）
```

请求头和请求体是两回事：头里是元信息（`Content-Type`、`Cookie` 等），体里才是实际数据（比如下单时提交的 JSON）。

---

## 二、请求到达服务器：谁在转发？

同一个 `/api/products`，在开发环境和生产环境走的路径不同：

```
开发环境：
浏览器 → Vite Dev Server（5173）→ 匹配到 /api → 转发到 localhost:3000 → Express

生产环境：
浏览器 → Nginx（80 / 443）→ 匹配到 /api/ → 转发到 127.0.0.1:3000 → Express
```

区别只是「谁来转发」：开发时是 Vite 的 proxy，部署后是 Nginx。所以这两件事本质上是同一个动作的两个环境版本。

Nginx 里那段配置：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;
}
```

意思是：**请求的 URL 路径**以 `/api/` 开头的，转发给本机的 3000 端口。

注意 `/api/` 匹配的是「请求路径」，不是服务器上的某个文件夹。

### 如果没有 proxy 会怎样

（这两个情况我实际推演过）

- 前端代码写 `/api/products`：请求会发到 5173（Vite 本身是个 HTTP 服务器），但它没有这个路由 → **返回 404**
- 前端代码写完整的 `http://localhost:3000/api/products`：请求能发出去、后端也确实处理了，但 **5173 到 3000 属于跨域**，浏览器不允许 JS 拿到响应 → 控制台报 CORS 错误

所以开发环境用 proxy，本质是让浏览器「以为」请求发给了同一个源，从而绕开同源策略。

---

## 三、Express 处理请求

请求进入 Express 后，走的是「中间件链」：

```
express.json()（解析请求体）
  → 路由匹配（/api/products）
  → 鉴权中间件（验证 JWT：通过就 next()，不通过就 401）
  → 路由处理函数
```

### `req.body` 是从哪来的

原生 Node 的 `req` 上**没有 `body`**。请求体在原生 Node 里是一个 **stream（流）**，需要自己监听 `data` / `end` 事件把数据拼起来，再解析。

`express.json()` 这个中间件做的就是这件事：读流 → 解析 JSON → 挂到 `req.body`。

所以在 Express 里能直接写 `req.body.xxx`，是因为中间件提前替你做完了。

### 中间件的三种走向

- **放行**：调用 `next()`，交给下一个
- **拦截**：直接响应（比如 `res.status(401).json(...)`），后面的都不再执行
- **抛错**：`next(err)`，交给错误处理中间件

---

## 四、访问数据库

处理函数里通过连接池执行 SQL：

```js
pool.query('SELECT ... FROM products WHERE ...', [参数])
```

项目里用到的几个关键点：

- **参数化查询**：SQL 里用 `?` 占位，参数单独传 → 防止 SQL 注入
- **连接池**：复用连接，避免每次请求都新建
- **条件更新防超卖**：`UPDATE ... WHERE id = ? AND stock >= ?` —— 把「检查」和「修改」放进同一条语句，由行级锁保证不会有两个请求同时扣成功；`affectedRows = 0` 说明条件没满足（库存不足）
- **事务**：`BEGIN → 多条 SQL → COMMIT / ROLLBACK`，保证一组操作要么全成功、要么全回滚
- **事务 ≠ 锁**：事务解决的是「这一组操作怎么原子提交」，锁解决的是「并发时谁能改这一行」

---

## 五、返回响应

```js
res.json({ products })
```

这一句做的事：

1. 把 JS 对象序列化成 JSON
2. 设置响应头 `Content-Type: application/json`
3. 通过 `res.send()` → 最终调用 `res.end()`，由 Node 把数据写回 socket

所以：`res.json()` 是 Express 提供的便捷方法，**真正把数据发出去的是 Node 的 http 模块**。

---

## 六、浏览器接收响应：缓存

响应里可能带缓存相关的头：

- `Cache-Control: max-age=3600` → **强缓存**：一小时内浏览器直接用本地缓存，**完全不发请求**
- `ETag` / `Last-Modified` → **协商缓存**：浏览器发请求时带上 `If-None-Match`，服务器比较后如果没变就返回 **304 Not Modified**（不带响应体），浏览器继续用本地缓存

所以两者的差别是：强缓存是 0 次请求，协商缓存是 1 次请求但不下载资源。

这些头由服务器设置（后端代码或 Nginx 都可以）。也就是说，缓存不是前端框架偷偷做的，而是 **HTTP 协议层面的机制**。

---

## 七、浏览器渲染

拿到 HTML 和 CSS 之后：

```
HTML → DOM 树
CSS  → CSSOM 树
DOM + CSSOM → Render Tree（display: none 的元素在这一步被排除）
  → Layout（计算每个元素的位置和尺寸）
  → Paint（绘制背景、文字、边框）
  → Composite（把图层合成最终画面）
```

### 三个和性能有关的点

1. **重排 vs 重绘**：改 `width` / `height` / 位置会触发 Layout（重排，代价大）；只改颜色、背景一般只触发 Paint（重绘，代价小）
2. **为什么动画推荐用 `transform` / `opacity`**：这两个属性通常只需要 Composite，不触发 Layout 和 Paint，所以更流畅
3. **强制同步布局（Forced Synchronous Layout）**：如果改完样式立刻读 `offsetWidth` / `getBoundingClientRect()`，浏览器会被迫马上把布局算出来——本该稍后统一做的事被提前做了。所以要避免「写 → 读 → 写 → 读」交替（也叫 layout thrashing）

### JS 执行和渲染的关系

一次事件循环里大致是：宏任务 → 微任务 → 浏览器有机会渲染 → 下一帧。

`requestAnimationFrame` 的回调是在「下一次渲染之前」执行，它和浏览器的帧调度关联，不等于 `setTimeout(fn, 16)`。

如果一段 JS 长时间占着主线程，浏览器就没机会渲染和处理用户输入，页面就会卡。

---

## 一句话总结

```
浏览器
  → HTTP
  → Nginx / Vite proxy
  → Express（中间件 → 路由 → 鉴权）
  → MySQL（查询 / 事务 / 条件更新）
  → HTTP 响应（缓存头）
  → 浏览器渲染（DOM → Render Tree → Layout → Paint → Composite）
```
