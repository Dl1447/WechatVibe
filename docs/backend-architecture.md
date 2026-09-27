# 后端分层与重构约束

本次拆分把原 `bridge/real_backend.py` 的数据读取、模型进程、存储和业务调度分开。
目标是减少跨职责修改；不改变接口格式、模型权重、推理算法、账号边界或持久化格式。

## 模块职责

| 模块 | 职责 | 不应承担的职责 |
| --- | --- | --- |
| `real_http.py` | HTTP 参数、响应和请求生命周期；组装应用服务 | SQL、模型进程管理 |
| `backend_service.py` | `Backend` 应用服务：账号作用域、任务调度、API 画像和缓存清理协调 | 直接查询/更新数据库、实现微信读取或 Node 通信 |
| `batch_engine.py` | 批处理调度、模型调用和扫描位置决策 | SQL、直接取得数据库连接 |
| `result_store.py` | `ResultStore`：SQLite schema 兼容升级、结果/进度/摘要/API 缓存、事务 | 调度线程、调用模型、读取微信原库 |
| `batch_state.py` | `BatchStateStore`：批次覆盖、断点、冻结的旧结果边界和原子提交 | HTTP、业务调度、模型调用 |
| `wechat_source.py` | `WeChatSource`：当前账号校验、只读会话/消息/媒体适配 | 分析结果存储、推理任务调度 |
| `node_analysis.py` | `NodeAnalysis`：Node 进程、请求协议、恢复及结果校验 | 账号结果数据库访问 |
| `backend_contracts.py` | 共享异常、版本键、常量及纯转换/校验函数 | 任何服务、适配器或存储层依赖 |
| `real_backend.py` | 旧导入路径的显式兼容导出 | 新业务实现 |

`backend_contracts.py` 只依赖标准库。存储层及适配器不得反向导入
`backend_service.py`、`real_http.py`、`batch_engine.py` 或兼容门面。
HTTP 入口直接导入应用服务及其适配器；已有调用仍可从 `real_backend` 导入类、异常和领域函数。
异常与类保持同一个对象，不使用复制类或门面子类。

## 服务与存储边界

- `Backend(source, analyzer=..., store_factory=...)` 保留构造注入，便于用合成来源、假模型及临时数据库测试。
- `project_result_store(account, workdir)` 在存储层生成原有账号哈希路径；快照工作目录不改变结果库位置。
- 服务调用 `ResultStore` 的 `save`、`progress`、`profile_state`、API 缓存等方法，不获取 SQLite 连接。
- 批处理通过 `first_skipped_position`、`legacy_known`、`move_cursor` 等仓储方法访问数据。
  服务负责判断“是否应该推进”，存储负责在原有账号/会话/版本/主体作用域内读写。
- `ResultStore.connect()` 保留给仓储内部及底层测试；不作为服务层接口使用。
- `store.path` 仍是现有任务去重键和同库批次仓储的身份标识，不是服务层读取数据库的入口。
- 原 SQL、表名、迁移、事务提交/异常回滚和旧结果冻结边界保持不变；本次不搬迁或清理已有用户数据。

## 兼容与打包

- 原 `real_backend` 的主要公开符号仍可导入。测试替换函数/常量时需 patch 其实际所属模块，
  例如 `backend_service.browse_history`、`wechat_source.contact_display`。
- `ModelSourceUnavailable` 和 `LOCAL_SOURCE_ID` 下沉到纯契约模块；`model_source` 继续导出同一个异常和常量。
- 新模块已加入 `scripts/stage-real-client.py` 的显式发布白名单。
  新增测试把白名单里的真实 bridge 文件复制到独立临时目录，以 Python 隔离模式导入桌面入口，
  防止源码目录掩盖漏打包依赖。
- 导入模块不得启动线程、Node 进程、数据库连接或网络连接。

## 验证

先按 README 安装 Node/Python 及锁定依赖。在项目根目录运行：

```powershell
.\.venv\Scripts\python.exe bridge/test_backend_layers.py -v
.\.venv\Scripts\python.exe -m unittest discover -s bridge -p "test_*.py"
```

新增测试覆盖兼容导出、单向模块依赖、服务无 SQL、导入无运行副作用、打包闭包、账号路径、
旧批次边界、事务回滚/持久化、游标移动、API 缓存隔离及仓储方法委派。
还应运行既有 Node、桌面/更新、Python 构建/启动测试。
统一 `npm test` 入口属于单独的测试流程改进，不是本次拆分的必要代码依赖。

## 后续范围

`Backend` 中的 API 画像与任务编排仍是较大的服务实现；本次先建立稳定的职责边界，
不同时重写并发模型或拆散账号锁生命周期。前端模块化、跨微信版本真实数据库 fixtures
和完整数据生命周期文档是另外的工作，不能以此次重构视为已完成。
