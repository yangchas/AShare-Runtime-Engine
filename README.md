# AShare Runtime Engine

A股竞价分析、盘中题材轮动、市场状态跟踪与策略决策辅助的运行时引擎。

本仓库由原项目中的 `engine_next` 模块独立拆出，保留包名 `engine_next`，便于原有导入路径和测试路径继续工作。

完整模块说明见 [`engine_next/README.md`](engine_next/README.md)，里面包含数据源、缓存键、运行时模型、阶段时间线和代码阅读顺序。

## 核心能力

- 竞价阶段分析、开盘验证和竞价锚点处理
- 盘中上下文构建、题材强弱判断和主线迁移跟踪
- 接力环境、梯队结构、前排/龙头状态的辅助判断
- 离线数据同步、缓存视图构建和运行时数据接入
- 策略信号审计、风险提示和终端输出渲染
- Python 运行时与 Rust 核心模块的渐进式集成

## 目录结构

```text
engine_next/
  adapters/              # 运行时适配层
  audit/                 # 复盘和审计流程
  connectors/            # 外部数据源连接器
  contracts/             # 数据契约和源语义
  domain/                # 领域模型
  offline/               # 离线同步、持久化和缓存构建
  resources/             # 单例资源和技能资源
  runtime/               # 盘中运行时、控制器、渲染器
  rust_core/             # Rust 核心与 Python 适配
  source_policies/       # 数据源和网络访问策略
  strategy_skill_layer/  # 策略技能、题材和梯队分析
  tests/                 # 项目测试
```

## 本地检查

可先做轻量语法检查：

```powershell
python -m py_compile engine_next/runtime/controllers/auction_runtime_controller.py
```

如本地依赖完整，可运行测试：

```powershell
python -m unittest discover engine_next/tests
```

注意：正式数据抓取、Redis、TDengine、盘中运行和远程服务相关流程依赖服务器环境；本地更适合做代码维护、静态检查、轻量导入和测试。

## 编码安全

项目包含大量中文注释、终端输出和策略标签。修改代码时请使用 UTF-8，并在提交前检查是否出现乱码、替换字符或 Windows 默认编码导致的中文损坏。

## 许可

本项目采用 PolyForm Noncommercial License 1.0.0。

仅允许非商业使用。任何商业使用、商业集成、商业部署或以营利为目的的使用，均需获得作者的明确书面授权。
