# Muse 无限画布

浏览器里的 AI 影像工作台：无限画布负责编排，Muse2API 提供生图/生视频能力，WorkBuddy 通过 MCP 直接操作画布。

## 架构

```
浏览器（无限画布 :3000）
  ├─ 生成请求 ──▶ Muse2API :18610 ──▶ muse.ai ──▶ data/media/
  └─ Agent SSE ◀── Canvas Agent :17371 ◀── MCP ── WorkBuddy
```

三个服务独立，缺一个有功能不可用：

| 服务 | 端口 | 缺了会怎样 |
|---|---|---|
| 无限画布前端 | 3000 | 打不开页面 |
| Muse2API 网关 | 18610 | 能开页面，但生成全失败 |
| Canvas Agent | 17371 | 能开页面，但 Agent 面板和 MCP 不可用 |

## 本主题文档

| 阶段 | 文档 | 内容 |
|---|---|---|
| 部署 | [deploy/云服务器部署清单.md](deploy/云服务器部署清单.md) | 三个项目上云：Docker、nginx 反代、WebDAV、自检、排错表 |
| 接入 | [integration/WorkBuddy接入教程.md](integration/WorkBuddy接入教程.md) | 从零到能对话操作画布，含环境准备与原理 |
| 规范 | [prompts/提示词规范.md](prompts/提示词规范.md) | 给 GPT 用的提示词规则：硬限制、模板、禁用词 |

## 建议阅读顺序

1. 先看 `deploy/` 把服务跑起来
2. 再看 `integration/` 接进 WorkBuddy
3. 写分镜时对照 `prompts/`

## 关键版本

| 组件 | 版本 |
|---|---|
| 无限画布 | basketikun/infinite-canvas v0.20.0 |
| canvas-agent | 0.6.0（捆绑 Codex 0.146.0） |
| Muse2API | czg86389-hub/muse2api |
| 实测环境 | Windows 11 + Node 22.22.2 + Python 3.12 |

## 三条铁律

1. **视频只能 16:9**，时长按 10 秒提交
2. **人物一致性只能靠参考图**：先出定妆图，后续每镜引用
3. **一次只提交一条生成**，画布轮询上限 300 秒

## 计划增加

- `troubleshooting/` 排错手册（把踩过的坑按现象归档）
- `workflow/` 实战流程（分镜 → 关键帧 → 视频 → 剪辑）
