# AI 编程工具详解

本文档详细介绍主流 AI 编程工具的特点、适用场景和上手指南。

## IDE 类

### Cursor

**特点：**
- 基于 VS Code，AI 原生设计
- 支持多模型切换（Claude、GPT-4 等）
- 强大的代码补全和重构能力
- Composer 模式支持多文件编辑

**适用场景：**
- 日常前端/后端开发
- 代码重构和优化
- 快速原型开发

**上手：** https://cursor.sh

### Trae

**特点：**
- 字节跳动出品
- 支持中文友好
- 集成豆包模型

**适用场景：**
- 中文开发者
- 国内网络环境

---

## CLI 类

### Claude Code

**特点：**
- Anthropic 官方 CLI 工具
- 强大的代码理解和生成能力
- 支持项目级上下文

**安装：**
```bash
npm install -g @anthropic-ai/claude-code
```

### OpenClaw

**特点：**
- 开源 AI 助手框架
- 多平台支持（QQ、Telegram、Discord 等）
- 插件化架构，可扩展
- Agent Skills 技能系统

**适用场景：**
- 自动化任务
- 个人助理
- 多平台机器人

**安装：** https://github.com/openclaw/openclaw

---

## 插件类

### GitHub Copilot

**特点：**
- GitHub 官方出品
- 支持 VS Code、JetBrains 全家桶
- 企业级方案成熟

**适用场景：**
- 企业开发团队
- 现有 IDE 用户增强

---

## 选择建议

| 场景 | 推荐工具 |
|------|----------|
| 个人开发者，追求效率 | Cursor |
| 团队协作，企业合规 | GitHub Copilot |
| 命令行爱好者 | Claude Code / OpenClaw |
| 中文用户，国内网络 | Trae |
| 想要开源可控 | OpenClaw |