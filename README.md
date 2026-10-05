# peidian-agent —— 园区配电运维智能体

> 接手文档 ｜ 任务卡见 [TASK.md](TASK.md)（**owner 已裁定目标转向，以 TASK.md 为准**）｜ 项目原始自述见 [README-upstream.md](README-upstream.md)

## 项目是什么

面向 10kV/0.4kV 园区配电（含储能/光伏/充电桩）的运维 agent 系统。工程主线：「受控行动网关（Policy 三值 + HITL 审批）+ 两票制红线 + 仿真驱动 EVAL + 黄金集飞轮」。已完成 M0–M9 模块族、ParkDSL v1.1 园区规则语言（含练习场三节）、15 类故障注入引擎、论文故障库（11 条目）、arena 模拟练习场（编排层+收敛判定）、人类体验模式（CLI 交互 + 全新 ui/ 前端 + serve API）与部署包（deploy/）。

**owner 2026-10-01 裁定转向**（TASK.md v1.1）：先建「用户侧园区电力经营管理」理论（docs/theory/ 四域卷，62 条参考文献）→ 冻结 agent 行为契约（docs/contracts/）→ 模块图 → **模拟练习场**：外部系统全部替身仿真，故障注入+时间加速，数万次模拟收敛最佳行为逻辑；人类可手动注入故障、关闭 agent 自行诊断（ui/）。

**安全设计核心**：危险动作（遥控/操作票）必须过受控网关；创建操作票 ≠ 执行遥控；两票制签发权永远在人（HITL 不可委派）；最终验收用与开发隔离的 holdout 独立实例。

## 架构一句话

ParkDSL 描述园区拓扑与规程 → 仿真环境（伪遥测/故障注入）→ agent 走「confirm/diagnose/judge/plan/isolate/restore/summary/verify」八相反应流，每个动作过 Policy 三值网关（allow/deny/需人工），全链留事件流证据。

## 构建与运行

- 环境：**Python 3.12 必须**（3.11 编译 `context_builder.py` 报语法错；pyproject 里 `>=3.11` 的声明待修正为 `>=3.12`，见已知问题）。
- 依赖安装：以 `pyproject.toml` 为准（`pip install -e .` 或按 README-upstream）。
- 评测门禁：`python run_evals.py --module all`
- 隔离扫描：`python scripts/ci_isolation.py`
- ParkDSL 套件：`python dsl/tests/run_tests.py`
- 故障注入套件：`fault/run_tests.py`（或 `python -m ...`，见 fault/README）
- 端到端演示：`python demo/run_demo.py --flow all`
- 练习场（阶段 d）：`python arena/tests/run_tests.py`（24 用例）；
  单场景 `python arena/run_scenario.py --config <yaml>`；
  人类体验模式 `python arena/serve.py --port 8790`（全新前端 ui/）；
  收敛 `python arena/converge.py calibrate|judge`

## 验收基线（2026-10-01 迁移后实测）

| 门 | 命令 | 基线 |
|---|---|---|
| G0-1 评测门禁 | `python run_evals.py --module all` | **233/233 PASS, exit 0（约 1 分钟）** |
| G0-2 隔离扫描 | `python scripts/ci_isolation.py` | **ISOLATION OK: zero hits**（仓内 grep holdout 关键词零命中） |
| G0-3 ParkDSL | `python dsl/tests/run_tests.py` | **25/25 PASS**（含 v1.1 练习场三节 10 个新用例） |
| G0-4 故障注入 | fault 套件 | **15/15 PASS**（含四类故障端到端） |
| G0-5 演示流 | `python demo/run_demo.py --flow all` | 全判据通过（pass_rate=1.0） |
| G0-6 练习场 | `python arena/tests/run_tests.py` | **24/24 PASS**（D-1 可跑/D-2 四步留痕/D-3 时间推移/确定性/人机对比/故障库装配/零信任/eval 口径） |

## 已知问题

1. **目标转向（2026-10-01 owner 裁定）**：在接入真实系统之前，先建「用户侧园区电力经营管理」理论认识 → 冻结 agent 行为契约 → 划分模块 → 构建模拟练习场（数万次模拟收敛最佳行为逻辑）。既有 M0–M9/ParkDSL/门禁体系作为工程底盘保留，复用/改造/废弃三态判定见 TASK.md。
2. **pyproject 声明与实际不符**：`>=3.11` 应为 `>=3.12`（3.11 下 6 个用例 SyntaxError），待修。
3. **holdout 纪律**：holdout 独立验收实例不在本仓库内（物理隔离）；`scripts/ci_isolation.py` 是防泄漏门，任何改动后必须保持 zero hits。
4. **LLM 注入桥**：`fault/llm_bridge.py` 支持 Higress 式网关注入（key 走环境变量，fail-closed 离线兜底）；无凭据时全链离线可跑（FakeLLM 脚本化验证），真实模型调用为 0 是当前实测态。

---

# 附：原独立仓 README（2026-09-30 冻结口径，两侧成果合并保留）


面向园区配电运维（10kV/0.4kV 配电房、储能/光伏/充电桩）的智能体系统：受控行动网关（Policy 三值 + HITL 审批）、两票制安全红线、证据化完成验证、仿真驱动的 EVAL 与黄金集飞轮。

## 仓库结构

```text
specs/        开发包规格（README 索引、00 本体、01 冻结契约、M1–M7 模块 spec、ADDENDUM 补遗）
ontology/     本体 YAML（对象/关系/动作/规则/枚举/种子实例，运行期只读）
regulations/  虚拟规程（REG-SAFE / REG-COMM / REG-TECH / REG-OP）
src/          m1_core / m2_information / m3_action / m4_semantic /
              m5_simulation / m6_flywheel / m7_registry + contracts
scenarios/    开发用仿真场景（dev-*.yaml）
golden/dev/   开发黄金集（cases/*.yaml）
tests/        EVAL 用例（test_m*.yaml，数据驱动）+ CHANGELOG.md
releases/     ReleaseBundle（不可变发布）
```

## 开发顺序

M4 → M5 → M2 → M3 → M1 → M6 → M7（依赖驱动，详见 `specs/README.md` §2）。

## 验收（holdout）说明

最终验收使用独立的 HOLDOUT 测试集（15 场景 + PARK-002 独立实例），**不在本仓库内**，由验收人持有。开发期间禁止以任何形式使用 holdout 数据做调优、修 bug、扩充黄金集；CI 断言仓库内 grep `PARK-002|TARIFF-2026B|OP-1x` 零命中。

## 工程约定

- Python 3.11+；SQLite 单文件起步；YAML 数据驱动（tests/ontology/golden/scenarios 全部文件化）。
- LLM 调用统一走 `src/m1_core/model_client`（provider 适配，mock 模式全离线跑 EVAL）。
- 时间纪律：电价判定只允许 M5 BUSINESS 时钟；CI 扫描业务代码无 wall-clock 判价。

## EVAL 运行方式

```bash
python run_evals.py --module m4     # 在仓库根（本目录）运行；--selftest / --module all 同
```

runner 以脚本自身位置锚定仓库根（pathlib，不假设其他 cwd）；摘要行输出 stdout（ASCII），
明细写 `runtime/eval_results.json`（仓库内）。若在仓库的上级目录执行同名命令，
由该目录下的转发启动器委托本仓库 runner（单一事实源在本仓库，上级目录仅转发）。
