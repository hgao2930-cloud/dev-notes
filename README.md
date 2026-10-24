# 开发笔记

记录项目中遇到的问题，以及理解这些问题需要的机制。围绕具体场景解释数据怎么流动、代码在哪里处理、方案有什么边界，方便后续排错和复盘。

## 文章目录

| 文章 | 主要内容 | 对应项目 |
| --- | --- | --- |
| [Flowboard 中的 WebSocket：从建立连接到同步白板](flowboard-websocket.md) | HTTP 升级、Yjs 同步、Awareness、远端拖动、持久化与权限边界 | 实时协作白板 |
| [一次请求的完整链路](request-lifecycle.md) | 浏览器、代理、Express、缓存、渲染与数据库 | MiniMall |
| [MiniMall 开发中的排错记录](minimall-debugging.md) | 路由参数、请求竞态、库存并发、无限滚动 | MiniMall |

## 阅读方式

想了解整体机制，可以先看请求链路与白板协作链路；遇到具体故障，可以从排错记录中寻找相似场景。文章中的实现取舍只对应当时的项目条件，不代表所有项目都应照搬。

笔记结合开发过程、代码审查与实际验证整理，使用 AI 辅助查阅和组织内容。已验证结果、解释性示例和未完成的验证尽量分别说明；后续理解发生变化时继续修订。

## 项目

- [Flowboard 实时协作白板](https://github.com/hgao2930-cloud/flowboard)
- [MiniMall 迷你商城](https://github.com/hgao2930-cloud/mini-mall-frontend)
