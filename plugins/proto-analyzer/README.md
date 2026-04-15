# proto-analyzer

一个 Claude Code 技能：将原型链接自动转化为完整的后端开发文档。

## 功能

提供一个 Figma、Axure 或任意原型链接，自动生成三份文档：

| 文档 | 内容 |
|------|------|
| **PRD.md** | 业务需求、字段规则、用户流程 |
| **SYSTEM-DESIGN.md** | 架构设计、数据模型、项目结构 |
| **API-SPEC.md** | RESTful 接口定义、请求/响应契约 |

## 工作流程

1. **原型探索** — 自动浏览原型页面，识别页面类型（表单/列表/详情/面板），提取字段并过滤模拟数据
2. **业务上下文** — 询问业务背景、目标用户和预期目标
3. **流程确认** — 展示完整业务流程，等待你确认后继续
4. **技术栈选择** — 选择后端框架（Spring Boot / FastAPI / Gin / NestJS 等）
5. **规范发现** — 自动查询所选框架的最佳实践和编码规范
6. **文档生成** — 先写 PRD，再并行生成 System Design + API Spec
7. **三轮验证** — 原型回溯检查 → 覆盖率检查 → 文档边界合规检查

## 安装

本项目已支持通过 Claude Code 插件市场一键安装。请在 Claude Code 命令行中依次执行以下命令：

**1. 添加插件市场**
```bash
/plugin marketplace add Fectivnfy112357/proto-analyzer
```

**2. 安装插件**
```bash
/plugin install proto-analyzer@proto-analyzer-market
```

**3. 重载配置**
```bash
/reload-plugins
```

## 使用

安装后，在对话中提供原型链接即可自动触发：

```
分析这个原型：https://www.figma.com/proto/xxx
```

## 输出目录

所有文档生成在项目根目录的 `docs/` 下：

```
docs/
├── PRD.md
├── SYSTEM-DESIGN.md
├── API-SPEC.md
└── VERIFICATION.md
```

## 支持的原型平台

- Figma
- Axure
- CoDesign / 蓝湖 / 墨刀
- 已部署的 HTML 演示页面
- 任何可访问的网页 URL
