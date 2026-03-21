---
name: commit-assistant
description: 当用户要求提交commit、生成commit消息、编写提交信息时使用此skill。帮助生成规范的Git commit消息，包含类型、作用域、简短描述和详细更改列表。
---

# Commit 提交助手

## 使用场景

当用户要求以下操作时，必须使用此skill：
- 提交 commit
- 生成 commit 消息
- 编写提交信息
- git commit

## Commit 消息格式规范

### 格式结构

```
<type>(<scope>): <简短描述>

[文件路径1]
- 具体更改点1
- 具体更改点2

[文件路径2]
- 具体更改点3

影响范围:
- 受影响的模块/功能1
- 受影响的模块/功能2

注意事项:（可选）
- 需要注意的点1
- 需要注意的点2
```

### Type 类型说明

| 类型 | 说明 |
|------|------|
| feat | 新功能 |
| fix | 修复bug |
| docs | 文档更新 |
| style | 代码格式（不影响代码运行的变动） |
| refactor | 重构（既不是新增功能，也不是修复bug） |
| perf | 性能优化 |
| test | 测试相关 |
| chore | 构建过程或辅助工具的变动 |
| ci | CI配置相关 |
| build | 构建系统或外部依赖变更 |

### Scope 作用域

- 表示影响的模块/组件名称
- 使用小写字母
- 简短明确

## 编写规则

1. **首行**：`type(scope): 简短描述`，不超过50个字符
2. **文件分组**：用 `[文件路径]` 标注，每个文件下列出该文件的具体改动
3. **详细列表**：每个更改点用 `-` 开头，清晰描述具体改动
4. **影响范围**：必填，说明此次改动影响哪些模块/功能
5. **注意事项**：可选，提醒其他开发者需要注意的点
6. **语言**：使用中文描述

## 示例

### 示例1：功能新增

```
feat(kaca): YOLO 推理后台线程化，优化 UI 响应性能

[workers.py]
- 新增 YoloWorker 类，将 YOLO 推理和后处理移至后台线程
- 新增 AprilTagWorker 类，将 AprilTag 检测移至后台线程

[main_window.py]
- 移除 REFRESH_MS 限制，使用默认刷新间隔
- 重构 _apply_yolo_overlay() 使用非阻塞模式
- 新增 _draw_yolo_detections() 方法仅负责绘制
- 新增 _process_apriltag_async() 非阻塞检测方法
- 在 closeEvent() 中正确停止所有后台线程

影响范围:
- 视频预览模块
- YOLO 检测结果渲染
- AprilTag 检测流程

注意事项:
- model.predict() (100-500ms) 已移至后台，UI线程仅负责轻量绘制
- 关闭窗口时需等待后台线程正确停止
```

### 示例2：Bug修复

```
fix(auth): 修复用户登录后token过期问题

[token_manager.py]
- 修正 token 刷新逻辑中的时间计算错误
- 添加 token 过期前5分钟自动刷新机制

[api_client.py]
- 修复并发请求时的 token 竞争条件
- 添加请求重试机制

影响范围:
- 用户登录状态维护
- 所有需要认证的 API 请求

注意事项:
- 旧版本用户可能需要重新登录一次
```

### 示例3：重构

```
refactor(database): 重构数据库连接池管理

[config/db_config.yaml]
- 新增连接池配置项

[database/connection_manager.py]
- 新增 ConnectionManager 类统一管理连接
- 将连接池配置从硬编码改为配置文件读取

[database/legacy.py]
- 移除废弃的 legacy_connect() 方法

影响范围:
- 所有数据库操作模块
- 应用启动流程

注意事项:
- 部署时需要添加 db_config.yaml 配置文件
- 连接泄漏问题已修复，无需手动释放连接
```

## 执行步骤

1. 分析用户的代码更改（通过 git diff 或用户描述）
2. 确定合适的 type 和 scope
3. 提炼简短描述（核心改动）
4. 列出所有具体更改点
5. 如有必要，添加分类说明（性能改进、破坏性变更等）
6. 按格式生成完整的 commit 消息
7. 使用 temp.txt 文件方式提交（防止中文乱码）
8. 提交后检查提交内容是否存在乱码，如有问题则回退并重新提交

## 提交方式（防乱码）

由于终端直接传递中文commit消息容易产生编码问题导致乱码，**必须使用临时文件方式提交**：

### 标准提交流程

```bash
# 1. 将commit消息写入临时文件（UTF-8编码）
# 2. 使用 -F 参数从文件读取commit消息
#    ⚠️ 注意：必须在 git add 之后、git commit 之前创建 temp.txt
#    这样 temp.txt 不会被 git add 纳入暂存区
git commit -F temp.txt
# 3. 提交完成后删除临时文件
rm temp.txt
```

### ⚠️ 严禁：temp.txt 误提交问题

**已知错误模式**：先创建 temp.txt → 再执行 git add . → 导致 temp.txt 被纳入暂存区并随代码一起提交。

**正确操作顺序（必须严格遵守）**：
```bash
# 1. 先暂存代码变更
git add .
# 2. 暂存完成后，再创建 temp.txt（此时 temp.txt 不在暂存区中）
# 3. 执行提交
git commit -F temp.txt
# 4. 删除临时文件
rm temp.txt
```

**如果 temp.txt 已被误提交，严禁用"删除文件+再提交一次"的方式修复。正确做法是撤销错误提交并重新提交**：
```bash
# 1. 回退错误的提交（保留代码更改）
git reset --soft HEAD~1
# 2. 将 temp.txt 移出暂存区
git reset HEAD temp.txt
# 3. 重新提交（此时暂存区中不含 temp.txt）
git commit -F temp.txt
# 4. 删除临时文件
rm temp.txt
```

### 提交后乱码检查

提交完成后，**必须执行以下检查**：

```bash
# 查看最近一次提交的消息
git log -1 --pretty=format:"%s%n%n%b"
```

**检查标准**：
- 中文字符正常显示，无 `??`、`\xxx`、`锟斤拷` 等乱码
- 标点符号正确显示
- 文件路径中的中文正常显示

**如果发现乱码**：
```bash
# 1. 回退提交（保留代码更改）
git reset --soft HEAD~1
# 2. 确认临时文件编码为UTF-8后重新提交
git commit -F temp.txt
# 3. 再次检查
git log -1 --pretty=format:"%s%n%n%b"
```