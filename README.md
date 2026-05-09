# My Skills - Claude Code 技能库

本仓库用于管理和同步 Claude Code 自定义技能。

## 技能列表

### psp-dev

**Product Sales Planning (PSP) 全栈开发助手**

基于 Frappe + Vue 3 的商品销售规划系统开发技能，适用于：
- 开发、修改、调试 product_sales_planning 项目的任何代码
- 创建或修改 Frappe DocType、编写 API 接口、配置权限
- 添加 Vue 页面/组件/composable、配置路由
- 审批流程、飞书集成、ClickHouse 查询等高级功能

**使用方法：**
```bash
# 复制到 Claude Code 技能目录
cp -r psp-dev ~/.claude/skills/
```

---

## 技能开发指南

Claude Code 技能通过 `/skill-name` 命令触发，每个技能包含：
- `SKILL.md` - 技能定义和主要指令
- `references/` - 详细参考文档（可选）

详细开发文档：https://docs.anthropic.com/claude-code/skills

## 版本历史

| 日期 | 技能 | 版本 | 说明 |
|------|------|------|------|
| 2025-05-09 | psp-dev | v1.0 | 初始版本 |
