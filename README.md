# 无限画布 · Muse2API · WorkBuddy 文档

本地 AI 影像工作台的部署与接入文档。三个部分：Muse2API 提供生图/生视频能力，无限画布做编排，WorkBuddy 通过 MCP 直接操作画布。

## 文档

| 文件 | 内容 | 适合谁 |
|---|---|---|
| [部署清单-云服务器版.md](部署清单-云服务器版.md) | 三个项目在云服务器的安装、nginx 反代、WebDAV 存储、MCP 配置、排错表 | 要上线部署的人 |
| [无限画布+WorkBuddy接入教程.md](无限画布+WorkBuddy接入教程.md) | 完整教程：环境准备、逐项目部署、画布配置、MCP 原理、常见问题 | 想搞懂原理的人 |
| [GPT_PROMPT_SPEC.md](GPT_PROMPT_SPEC.md) | 给 GPT 的提示词规范：Muse 的硬限制、写法模板、禁用词替换 | 写分镜/提示词的人 |

## 架构

```
浏览器（无限画布）
  ├─ 生成请求 ──▶ Muse2API :18610 ──▶ muse.ai ──▶ data/media/
  └─ Agent SSE ◀── Canvas Agent :17371 ◀── MCP ── WorkBuddy
```

## 关键版本

- 无限画布：basketikun/infinite-canvas v0.20.0
- canvas-agent：0.6.0（捆绑 Codex 0.146.0）
- Muse2API：czg86389-hub/muse2api
- 实测环境：Windows 11 + Node 22.22.2 + Python 3.12

## 三条铁律

1. **视频只能 16:9**，时长按 10 秒提交
2. **人物一致性只能靠参考图**：先出定妆图，后续每镜引用
3. **一次只提交一条生成**，画布轮询上限 300 秒

---

文档里的 key、路径均为占位符，使用时替换成自己的。
