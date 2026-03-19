# AI 爆品预测工具

> AI 驱动的电商数据分析与爆品预测平台

---

## 项目简介

本项目旨在通过 AI 大模型 + 海量数据分析，帮助电商商家发现运营规律、预测爆款商品、优化直播策略。

**核心思想：** 想是想不出来的，只能调研分析外界需要什么。

---

## 文档目录

### 一、产品需求设计

| 文档 | 说明 |
|------|------|
| [产品需求文档 (PRD)](./docs/product/prd-ai-predict-tool.md) | 产品定位、功能模块、MVP 规划 |
| [项目调研文档](./docs/product/research-ai-predict-tool.md) | 市场调研、竞品分析、可行性评估 |

### 二、技术方案实现调研

| 文档 | 说明 |
|------|------|
| [数据采集方案](./docs/technical/data-collection-design.md) | 抖音/淘宝/1688 爬虫、反爬对抗、代理池 |
| [爆品预测算法设计](./docs/technical/prediction-algorithm-design.md) | 特征工程、模型训练、预测服务 |
| [爆品预测算法调研](./docs/technical/hot-product-prediction-research.md) | 时序预测、机器学习、深度学习技术调研 |
| [技术架构设计](./docs/technical/technical-architecture-design.md) | 微服务架构、数据库设计、AI 能力层 |

---

## 核心功能

- **数据采集** - 多平台数据抓取（抖音、淘宝、1688）
- **爆品预测** - AI 预测未来爆品
- **运营分析** - 直播策略、话术建议
- **知识图谱** - 商品关联、竞品分析

---

## 技术栈

| 层级 | 技术 |
|------|------|
| 后端 | Java 17 + Spring Boot 3 |
| 前端 | Vue 3 + TypeScript |
| 数据库 | MySQL + Redis + ES + ClickHouse |
| AI | LangChain4j / Spring AI |
| 部署 | Docker + Kubernetes |

---

## 项目地址

- GitHub: https://github.com/qq296629801/ai-code-guide

---

## License

MIT