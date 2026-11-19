# ai- 文档库

个人 AI 实践文档集合。每个主题一个顶层目录，目录内按阶段分层。

## 目录结构

```
ai-/
├─ README.md                     本文件：全库索引 + 命名约定
└─ muse-canvas/                  「Muse 无限画布」主题
   ├─ README.md                  主题索引
   ├─ deploy/                    部署
   │  └─ 云服务器部署清单.md
   ├─ integration/               接入
   │  └─ WorkBuddy接入教程.md
   └─ prompts/                   规范
      └─ 提示词规范.md
```

## 已有主题

| 主题 | 说明 | 入口 |
|---|---|---|
| **muse-canvas** | 无限画布 + Muse2API 本地影像工作台，接 WorkBuddy | [muse-canvas/README.md](muse-canvas/README.md) |

## 新增内容时的约定

**1. 一层一个 README**
每个主题目录必须有 `README.md`，写清这个主题是什么、下面每一层放什么。从根 README 点进来，不用猜。

**2. 目录名用英文，文件名用中文**
目录走 ASCII（URL 友好、命令不用转义），文件名用中文（GitHub 上直接可读）。

**3. 按「阶段」分层，不按「时间」**

```
主题/
├─ deploy/          怎么装
├─ integration/     怎么接
├─ prompts/         怎么写
├─ troubleshooting/ 怎么排错
└─ changelog/       怎么变的
```

新增文档先想清楚属于哪个阶段，再决定放哪层。最多三层，再深说明该拆成新主题了。

**4. 主题独立**
每个主题自带完整链路，不跨主题引用。要复用就复制过去，别搞软链接。

**5. 不写密钥**
文档里一律用占位符（`m2a_xxx`、`你的域名`）。真实 key 放本地 `.env`，不进仓库。

## 提交规范

```
docs(主题): 说明   → 新增或修改文档
chore: 说明        → 结构、配置调整
```

换行由 `.gitattributes` 统一：文本 LF，`.bat` / `.cmd` 保持 CRLF。
