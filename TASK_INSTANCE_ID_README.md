# Task Instance ID 功能使用说明

## 功能概述

本功能允许在 Addax 任务执行完成后，将 `task_instance_id` 添加到结果上报报告中。该 ID 从作业 JSON 配置的 `job.setting.taskInstanceId` 字段中读取。

## 配置方式

### 在作业 JSON 配置中指定

在您的作业 JSON 配置文件的 `job.setting` 节点下添加 `taskInstanceId` 字段：

```json
{
  "job": {
    "setting": {
      "speed": {
        "byte": -1,
        "channel": 1
      },
      "errorLimit": {
        "record": 0,
        "percentage": 0.02
      },
      "taskInstanceId": "TASK_20260428_001"
    },
    "content": {
      // ... 其他作业配置
    }
  }
}
```

**完整示例：**

```json
{
  "job": {
    "setting": {
      "speed": {
        "byte": 1048576,
        "channel": 3
      },
      "errorLimit": {
        "record": 0,
        "percentage": 0.02
      },
      "taskInstanceId": "TASK_20260428_MYSQL_TO_HIVE_001"
    },
    "content": [
      {
        "reader": {
          "name": "mysqlreader",
          "parameter": {
            "username": "root",
            "password": "password",
            "column": ["id", "name", "create_time"],
            "connection": [
              {
                "jdbcUrl": ["jdbc:mysql://localhost:3306/test"],
                "table": ["users"]
              }
            ]
          }
        },
        "writer": {
          "name": "hdfswriter",
          "parameter": {
            "defaultFS": "hdfs://namenode:8020",
            "fileType": "text",
            "path": "/data/users",
            "fileName": "users",
            "column": [
              {"name": "id", "type": "long"},
              {"name": "name", "type": "string"},
              {"name": "create_time", "type": "date"}
            ]
          }
        }
      }
    ]
  }
}
```

## 上报结果示例

当配置了 `core.server.address` 后，Addax 会将包含 `task_instance_id` 的结果报告发送到指定服务器。上报的 JSON 格式如下：

```json
{
  "startTimeStamp": 1234567890,
  "endTimeStamp": 1234567900,
  "totalCosts": 10,
  "totalBytes": 1048576,
  "byteSpeedPerSecond": 104857,
  "recordSpeedPerSecond": 1000,
  "totalReadRecords": 10000,
  "totalErrorRecords": 0,
  "jobName": "mysql.to.hive",
  "jobContent": "...",
  "task_instance_id": "TASK_20260428_MYSQL_TO_HIVE_001"
}
```

## 日志输出

成功获取到 `task_instance_id` 时，会在日志中看到：

```
INFO  - Added task_instance_id: TASK_20260428_MYSQL_TO_HIVE_001 to result report
```

如果未在配置中找到，会看到（无日志输出或 DEBUG 级别）：

```
DEBUG - task_instance_id not found in job settings
```

## 注意事项

1. **配置位置**：`taskInstanceId` 必须位于 `job.setting` 节点下，与 `speed`、`errorLimit` 同级
2. **上报前提**：必须在 `conf/core.json` 或环境变量中配置 `core.server.address` 才能触发结果上报
3. **唯一性建议**：建议使用具有业务意义的唯一标识符，例如：`{日期}_{源系统}_{目标系统}_{序号}`
4. **字段名称**：配置时使用驼峰命名 `taskInstanceId`，上报时会转换为下划线命名 `task_instance_id`

## 技术实现

- **常量定义**: `CoreConstant.JOB_SETTING_TASK_INSTANCE_ID = "job.setting.taskInstanceId"`
- **核心逻辑**: `JobContainer.logStatistics()` - 从作业配置中读取并注入到报告
- **注入位置**: 在构建 `resultLog` Map 时添加

## 故障排查

### 问题：task_instance_id 未出现在上报结果中

**检查步骤**：

1. ✅ 确认作业 JSON 中是否在 `job.setting` 下正确配置了 `taskInstanceId`
2. ✅ 确认是否配置了 `core.server.address`
3. ✅ 检查 JSON 格式是否正确（注意层级关系和逗号）
4. ✅ 查看日志中是否有 "Added task_instance_id" 的消息
5. ✅ 验证配置路径是否正确：必须是 `job.setting.taskInstanceId`

### 常见错误示例

❌ **错误的位置**：
```json
{
  "job": {
    "taskInstanceId": "xxx"  // 错误：应该在 setting 下
  }
}
```

❌ **错误的层级**：
```json
{
  "core": {
    "taskInstanceId": "xxx"  // 错误：不应该在 core 下
  }
}
```

✅ **正确的位置**：
```json
{
  "job": {
    "setting": {
      "taskInstanceId": "xxx",  // 正确：与 speed、errorLimit 同级
      "speed": {...},
      "errorLimit": {...}
    }
  }
}
```

### JSON 语法检查

使用在线 JSON 验证工具或命令行工具检查配置文件的语法：

```bash
# 使用 python 验证 JSON 格式
python -m json.tool your_job.json > /dev/null
```

## 最佳实践

1. **命名规范**：使用有意义的命名，便于追踪和统计
   - 推荐：`TASK_20260428_ORDER_SYNC_001`
   - 不推荐：`task1`, `test`, `abc`

2. **统一管理**：如果使用调度系统（如 Airflow、DolphinScheduler），可以在生成作业 JSON 时动态注入 `taskInstanceId`

3. **版本控制**：将作业 JSON 配置文件纳入版本控制系统，便于追溯变更

4. **环境区分**：不同环境使用不同的 ID 前缀
   - 开发环境：`DEV_TASK_xxx`
   - 测试环境：`TEST_TASK_xxx`
   - 生产环境：`PROD_TASK_xxx`

5. **与现有配置保持一致**：`taskInstanceId` 应该与 `speed`、`errorLimit` 等配置项放在同一层级，符合 Addax 的配置规范

## 配置结构说明

根据 [Addax 官方文档](https://wgzhao.github.io/Addax/5.1.2/setupJob/#settings)，`job.setting` 用于定义任务的控制参数，包括：

- `speed`: 流控配置（字节速度、记录速度、通道数）
- `errorLimit`: 错误限制（错误记录数、错误率）
- `taskInstanceId`: 任务实例 ID（新增，用于结果上报追踪）

这种配置方式符合 Addax 的设计规范，使得配置结构更加清晰和统一。
