# Python SDK 参考

`openclaw` Python 包的完整 API 参考。

## 核心类

### `AgentHardwareClient`

代理硬件操作的主要入口点。

```python
class AgentHardwareClient(
 identity_path: str,
 network: str = "testnet",
 tee_enclave: Optional[str] = None,
 config: Optional[Dict] = None
)
```

参数：

| 名称 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| identity_path | str | 必需 | 代理 DID JSON 文件路径 |
| network | str | "testnet" | 网络: "local", "testnet", "mainnet" |
| tee_enclave | Optional[str] | None | TEE 类型: "sgx", "trustzone", "nitro" |
| config | Optional[Dict] | None | 高级配置选项 |

示例：

```python
client = AgentHardwareClient(
 identity_path="agent.json",
 network="testnet",
 tee_enclave="sgx"
)
```

### 方法

#### analyze_bottlenecks

分析当前硬件性能并决定是否需要演化。

```python
async def analyze_bottlenecks(
 current_metrics: Dict[str, float],
 target_capabilities: List[str],
 lookahead_days: int = 30
) -> CapabilityGap
```

参数：

- **current_metrics**: 性能测量
  - `throughput`: 每秒 token 数或推理数
  - `latency_ms`: P99 延迟（毫秒）
  - `power_consumption`: 消耗瓦特数
  - `cost_per_hour`: 运营成本

- **target_capabilities**: 所需功能列表
  - 选项: "edge_inference", "real_time_video", "distributed_training" 等

- **lookahead_days**: 需求预测的时间范围

返回：
`CapabilityGap` 对象包含：

- `compute_bottleneck`: 识别的瓶颈类型
- `urgency_score`: 0.0-1.0 的浮点数（演化触发阈值：0.7）
- `knowledge_deficit`: 未覆盖功能的熵度量
- `economic_pressure`: 成本效率比

示例：

```python
gap = await client.analyze_bottlenecks(
 current_metrics={"throughput": 20, "latency_ms": 200},
 target_capabilities=["real_time_generation"]
)

if gap.urgency_score > 0.7:
 print(f"建议演化: {gap.compute_bottleneck}")
```

#### submit_evolution_intent

向 Fabric 提交硬件演化请求。

```python
async def submit_evolution_intent(
 gap: CapabilityGap,
 target_stage: EvolutionStage = EvolutionStage.ARCHITECTURE,
 budget_constraints: Optional[BudgetConstraints] = None,
 diversity_required: bool = True
) -> EvolutionJob
```

抛出：
- `ValueError`: 如果 urgency_score < 0.7

返回：
用于跟踪硬件演化进度的 `EvolutionJob`。

#### stream_milestones

用于实时进度更新的异步生成器。

```python
async def stream_milestones() -> AsyncIterator[Dict[str, Any]]
```

产生带有以下键的字典：

- `stage`: 设计阶段（例如 "rtl_generation", "placement", "signoff"）
- `completion`: 完成百分比 (0-100)
- `timestamp`: Unix 时间戳
- `artifacts`: 生成文件的 IPFS CID（如果有）

示例：

```python
async for milestone in job.stream_milestones():
 print(f"{milestone['stage']}: {milestone['completion']}%")

 if milestone['stage'] == 'rtl_generation':
 print(f"RTL available at: {milestone['artifacts']['verilog']}")
```

### SiliconCompiler

AI 原生芯片设计接口。

#### compile

从规格生成硬件。

```python
def compile(
 spec: HardwareSpec,
 optimization_target: str = "balanced",
 constraint_set: str = "default",
 enable_ai_exploration: bool = True
) -> CompilationJob
```

优化目标：

- "performance": 最大化 TFLOPS
- "energy_efficiency": 最大化 TFLOPS/Watt
- "area": 最小化芯片面积
- "balanced": 帕累托最优权衡

## 数据类

### HardwareSpec

硬件规格容器。

```python
@dataclass
class HardwareSpec:
 process_node: str # "28nm", "14nm", "7nm", "5nm"
 target_tflops: float
 power_budget_w: int
 area_budget_mm2: float
 form_factor: str # "m.2_2280", "pcie_full", "soc"
 memory_architecture: str # "standard", "cim_sram", "cim_reram"
 isa: str # "riscv_custom", "arm", "x86"
 backward_compatible: bool # 必须运行父代软件
```

### CIMConfig

存算一体 (Compute-in-Memory) 特定配置。

```python
@dataclass
class CIMConfig:
 array_rows: int = 512
 array_cols: int = 512
 bitcell_type: str = "sram_6t" # 或 "reram", "pcm"
 adc_resolution: int = 8
 digital_accumulation: bool = True
 sparse_access_optimization: bool = False
 near_memory_compute: bool = True
```

## 异常

### EvolutionSafetyError

当提议的演化违反安全约束时引发。

属性：

- `violation_type`: "constitution_lock", "diversity_requirement" 等
- `details`: 人类可读的解释
- `suggested_alternative`: 安全的替代规格

### FabricationError

制造阶段失败。

属性：

- `stage`: 哪个阶段失败了 ("mask_making", "wafer_processing" 等)
- `recoverable`: 是否可以重试
- `insurance_payout`: 自动补偿金额（如果适用）

## 类型提示

IDE 支持的完整类型定义：

```python
from typing import TypedDict, Literal

class Metrics(TypedDict):
 throughput: float
 latency_ms: float
 power_consumption: float
 cost_per_hour: float

ProcessNode = Literal["28nm", "14nm", "7nm", "5nm", "3nm"]
FormFactor = Literal["m.2_2230", "m.2_2280", "m.2_22110", "pcie_half", "pcie_full", "soc"]
```

## 常量

```python
from openclaw import constants

constants.MINIMUM_EVOLUTION_BUDGET_USD # 10000
constants.MAXIMUM_EVOLUTION_STAGES # 5
constants.DEFAULT_VERIFICATION_TIMEOUT # 7200 seconds
constants.CONSTITUTION_ROM_ADDRESSES # [0x0000, 0x0001, 0x0002]
```

## 异步模式

所有网络绑定方法都是异步的。使用 `asyncio` 或 `trio`:

```python
import asyncio
from openclaw import AgentHardwareClient

async def main():
 client = AgentHardwareClient("agent.json")
 # ... 异步操作

# 推荐: 使用 asyncio.run()
asyncio.run(main())

# 对于 Jupyter 笔记本:
# await main()
```