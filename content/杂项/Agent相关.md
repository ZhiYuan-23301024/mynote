---
title: Agent相关
---
如何开发agent
- 训练模型
- 构建Harness：编写代码，为模型提供可操作的环境

```
Harness = Tools + Knowledge + Observation + Action Interfaces + Permissions

    Tools:          文件读写、Shell、网络、数据库、浏览器
    Knowledge:      产品文档、领域资料、API 规范、风格指南
    Observation:    git diff、错误日志、浏览器状态、传感器数据
    Action:         CLI 命令、API 调用、UI 交互
    Permissions:    沙箱隔离、审批流程、信任边界
```

Harness 工程师在做什么？
- 实现工具：读文件？调API？浏览器控制？->原子化、可组合
- 策划知识：产品文档？风格指南？
- 管理上下文：上下文压缩？子agent隔离？任务系统？
- 控制权限：沙箱化文件访问？
- 搜集任务过程数据：为agent进化赋能