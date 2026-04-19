# 飞书知识库资产目录

## 背景

目标：编写脚本递归下载飞书所有知识库文档（包含子节点内容）

## 存储结构

```
feishu_wiki/
├── metadata.json          # 整体元数据（知识库列表、统计）
└── spaces/
    ├── space_{space_id}/
    │   ├── metadata.json  # 知识库信息（name, space_id, description）
    │   └── nodes/
    │       ├── node_{node_token}.json   # 节点详情（node + detail）
    │       └── ...
    └── ...
```

## 环境配置

复制 `app/.env.example` 为 `app/.env` 并配置：

| 变量 | 说明 | 默认值 |
|------|------|--------|
| BASE_DIR | 数据输出目录 | ./data/feishu_wiki |
| PAGE_SIZE | API 分页大小 | 50 |
| REQUEST_DELAY | 请求间隔（秒） | 0.2 |

## 目录结构

```
examples/feishu-wiki/
├── app/
│   ├── config.py     # 配置和路径函数
│   └── catalog.py   # 主程序
├── data/            # 下载的数据
├── .env             # 环境变量（需复制）
├── .env.example     # 配置模板
└── .gitignore
```

## 配置说明

### config.py

```python
# 配置变量
BASE_DIR      # 数据输出目录
SPACES_DIR    # 知识库目录
PAGE_SIZE     # 分页大小
REQUEST_DELAY # 请求间隔

# 路径函数
space_dir(space_id)        # 知识库目录
node_file(space_id, node_token)  # 节点文件路径
metadata_file(space_id)    # 知识库元数据路径
ensure_space_dirs(space_id) # 创建目录
```

## 运行

```bash
cd examples/feishu-wiki/app
pip install python-dotenv
cp .env.example .env
python catalog.py
```

## 关键API

| 接口 | 功能 | 方式 |
|------|------|------|
| `/open-apis/wiki/v2/spaces` | 获取知识库列表 | `lark-cli api GET` |
| `/open-apis/wiki/v2/spaces/{id}/nodes` | 获取节点列表 | `lark-cli api GET` + 递归 |
| `wiki spaces get_node` | 获取节点详情 | `lark-cli wiki spaces get_node` |