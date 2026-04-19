# 飞书知识库资产目录

## 背景

目标：编写脚本递归下载飞书所有知识库文档（包含子节点内容）

## 问题分析：API 用错了

之前的代码用 `GET /wiki/v2/spaces/get_node` 并传入 `parent_node_token`，期望返回子节点列表，这完全不符合 API 设计意图。

| 接口 | 功能 | 关键参数 | 之前是否正确使用 |
|------|------|---------|----------------|
| `GET /wiki/v2/spaces/get_node` | **获取单个节点信息** | `token`（节点 token） | ❌ 错用，传了 `parent_node_token` 试图获取列表 |
| `GET /wiki/v2/spaces/:space_id/nodes` | **分页获取子节点列表** | `parent_node_token` | ❌ 从未调用过 |

获取子节点列表应使用 `/wiki/v2/spaces/:space_id/nodes` 接口，通过 `parent_node_token` 查询参数指定父节点。而 `get_node` 接口的查询参数是 `token` 而非 `parent_node_token`，用于获取单个节点的详细信息。

**递归失败的原因：根本没用对 API，而不是什么 Bug 或权限问题。**

### 执行过程

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

## 正确的递归下载方案

### 2.1 第一步：获取知识库根节点

先调用获取知识空间列表接口 `/wiki/v2/spaces`，拿到 `space_id` 和根节点 `node_token`。

### 2.2 第二步：递归获取子节点列表

对于每个节点（从根节点开始），调用 `GET /open-apis/wiki/v2/spaces/:space_id/nodes`，传入 `parent_node_token` 参数：

- `:space_id`：知识库的空间 ID（路径参数）
- `parent_node_token`：父节点的 token（查询参数）
- `page_size`：每页大小，最大 50
- `page_token`：分页标记，用于获取下一页

响应中的 `has_more` 字段指示是否还有更多数据，若有则用返回的 `page_token` 继续请求。

### 2.3 第三步：获取节点内容

对于叶子节点，调用 `GET /wiki/v2/spaces/get_node` 接口，传入节点的 `node_token`，获取节点详细信息，包括 `obj_type` 和 `obj_token`，然后用 `obj_token` 去获取实际文档内容。

### 2.4 第四步：处理快捷方式节点

响应中若 `node_type` 为 `shortcut`，则 `origin_node_token` 指向实际文档所在的节点，需要追踪到实际节点再获取内容。

## 待确认问题

文档虽然指明了正确的 API，但以下关键信息在文档中**未找到或需进一步验证**：

### 3.1 API 路径差异

官方文档中 `list` 接口的路径是 `/wiki/v2/spaces/:space_id/nodes`。但 `lark-cli` 是否支持该路径、支持的参数映射关系如何，文档中没有说明。

### 3.2 权限要求

`list` 接口需要"父节点阅读权限"，`get_node` 接口需要"节点阅读权限"。当前 `lark-cli` 使用的凭证是否具备这些权限，**文档未提供验证方法**。

### 3.3 `lark-cli` 的具体用法

`lark-cli` 官方介绍仅提及支持"知识库 查询空���、管理节点和文档层级"，但未给出具体 API 调用示例，需要自行摸索。

### 3.4 飞书 Wiki 与普通文档的内容获取差异

从已有经验看，Wiki 需要先通过 `get_node` 获取节点信息，再用返回的 `obj_token` 去获取实际文档内容，不能直接用 `node_token` 请求文档 API。

## 下一步计划

### 4.1 优先级最高：验证 API 路径（今日完成）

用 curl 直接调用正确的 `list` API，排除 `lark-cli` 封装干扰：

```bash
curl --location --request GET 'https://open.feishu.cn/open-apis/wiki/v2/spaces/{space_id}/nodes?parent_node_token={parent_token}&page_size=50' \
--header 'Authorization: Bearer {access_token}'
```

对比 `lark-cli` 能否发出相同请求，判断 CLI 是否支持该接口。

### 4.2 确认权限范围（今日完成）

检查当前 `lark-cli` 使用的 access_token 实际包含的权限 scope，确认是否包含：

- 查看知识空间节点列表
- 查看、编辑和管理知识库
- 查看知识库

### 4.3 实现递归下载脚本（本周完成）

无论 `lark-cli` 是否支持，用 Python 脚本实现：

1. 获取 tenant_access_token
2. 获取所有知识库的 space_id 和根节点
3. 递归调用 `list` 接口获取子节点
4. 调用 `get_node` 获取节点详情
5. 下载文档内容并转换为 Markdown

### 4.4 处理快捷方式节点（后续）

对 `node_type: shortcut` 的节点，需要追踪到 `origin_node_token` 指向的实际节点再获取内容。

### 4.5 如果 lark-cli 确实不支持

放弃依赖 CLI，直接基于飞书 Python SDK 或原生 REST API 实现完整脚本。

## 总结

1. **没有先看 API 文档就直接动手写代码**，导致用错了接口。
2. **前期排查方向错误**，猜测 Bug 和权限问题，但根本原因是 API 调用错误。
3. **没有先做对照验证**，如果用 curl 调用正确的 list API，问题早就定位了。

下一步按计划执行，预期本周内完成可用的递归下载脚本。