# 开发笔记

记录我在项目开发中遇到的问题，以及排查、定位和解决的过程。重点放在**问题的原理**和**为什么这么解决**，而不是操作步骤。

大部分问题来自两条途径：写完功能后的自查，以及借助 AI 工具做代码审查。我负责把问题理解清楚、修复，并总结成可以复用的经验。

## 问题排查（troubleshooting）

| 文档 | 主题 |
|---|---|
| [Express 路由参数顺序](troubleshooting/express-route-params.md) | `res` 被当成了 `req`：接口 500 但前端没有提示 |
| [滚动加载的请求竞态](troubleshooting/request-race-condition.md) | 「加载更多」与「切换筛选」同时发生时的数据错乱 |
| [数据库行锁](troubleshooting/mysql-row-lock.md) | 条件更新防超卖、固定加锁顺序防死锁 |
| [无限滚动里的视口与 DOM](troubleshooting/infinite-scroll-dom.md) | 哨兵位置、IntersectionObserver 触发时机、v-for 重建 |

## 相关项目

- [MiniMall 前端](https://github.com/hgao2930-cloud/mini-mall-frontend)
- [MiniMall 后端](https://github.com/hgao2930-cloud/mini-mall-server)
