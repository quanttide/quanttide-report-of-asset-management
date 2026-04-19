# 飞书知识库资产目录

## 背景

目标：编写脚本递归下载飞书所有知识库文档（包含子节点内容）

## 执行过程

### 1. 初始版本

- 使用 `lark-cli docs +fetch` 下载文档内容
- 输出为 JSON 格式

### 2. 第一次改进

- 使用 `lark-cli api GET /wiki/v2/spaces` 获取知识库列表
- 支持多个知识库同时下载
- 输出到 data 目录

### 3. 增加递归逻辑

- 飞书 Wiki API 支持 `parent_node_token` 参数查询子节点
- 实现递归函数逐层获取子节点

### 4. 问题排查

尝试了以下方式：

| 方式 | 结果 |
|------|------|
| 直接 GET 带参数 | 返回根节点 |
| --params 参数 | 返回根节点 |
| --as bot 身份 | 返回根节点 |
| 空 parent_node_token | 返回根节点 |

### 5. 最终方案

- 放弃递归获取子节点
- 只保存第一层节点
- 通过 `has_child=true` 标记哪些节点存在子节点

## 问题分析

| 问题 | 可能原因 |
|------|---------|
| parent_node_token 参数被忽略 | API bug 或权限限制 |
| 权限问题 | 需 wiki:node:retrieve 范围 |
| lark-cli 封装 | CLI 工具限制 |

飞书 Wiki API 文档要求权限：
- `wiki:node:retrieve` - 查看知识库节点列表
- `wiki:wiki` - 查看、编辑和管理知识库

lark-cli 的权限范围可能不包含节点列表查询。


## 后续建议

1. 确认 lark-cli 权限范围
2. 考虑为应用添加更多 Wiki 权限
3. 逐个调用 `get_node` API 获取子节点（需多次调用）
4. 或使用原生飞书应用获取完整权限
