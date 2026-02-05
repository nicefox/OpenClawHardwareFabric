# 教程：设计您的第一个自定义芯片

在此教程中，您将引导一个代理使用 CIM (存算一体) 技术设计一个**稀疏注意力加速器**，从高级意图到 GDSII 布局。

**预计时长**: 45 分钟  
**前置条件**: 已完成[快速开始](../getting-started/quickstart.md)  
**最终成果**: 生成的 Verilog 代码和可用于流片的物理布局

## 概述

我们将设计一个针对**4位量化的大语言模型推理**进行优化的芯片，目标：

- 28nm 工艺 (首次流片的经济选择)
- M.2 封装 (2280)
- <10W 功耗
- 对于 7B 参数模型，50+ tokens/秒

## 步骤 1：定义工作负载特征

创建 `workload_analysis.py`:

```python
from openclaw import WorkloadProfiler
import numpy as np

# 模拟代理的典型推理模式
profiler = WorkloadProfiler()

# 分析注意力机制 (瓶颈)
attention_pattern = {
 "sequence_length": 4096,
 "head_dim": 128,
 "num_heads": 32,
 "sparsity_ratio": 0.85, # 85% 稀疏 (典型的长序列)
 "bitwidth": 4 # INT4 量化
}

# 分析计算与内存强度
roofline = profiler.analyze(attention_pattern)

print(f"算术强度: {roofline.ops_per_byte:.2f} OPs/byte")
print(f"计算密集型: {roofline.is_compute_bound}")
print(f"最优精度: {roofline.suggested_bitwidth}")

"""
预期输出:
Arithmetic Intensity: 12.5 OPs/byte
Compute Bound: False (Memory bound!)
Optimal Precision: INT4
Recommendation: CIM architecture recommended
"""

!!! info "为什么选择 CIM?"
由于工作负载是内存密集型的（算术强度低），CIM 通过在内存阵列中直接计算消除了数据移动，提供 10-100 倍的能效提升。
```

## 步骤 2：生成硬件规格
根据分析，生成机器可读的规格：

```python
from openclaw import HardwareSpec, CIMConfig

spec = HardwareSpec(
 process_node="28nm",
 target_tflops=50,
 power_budget_w=10,
 area_budget_mm2=50,
 form_factor="m.2_2280",
 memory_architecture="cim_sram", # 关键决策
 quantization_scheme="int4"
)

# 配置 CIM 宏详细信息
cim_config = CIMConfig(
 array_rows=512,
 array_cols=512,
 adc_resolution=8,
 digital_accumulation=True,
 sparse_access_optimization=True # 注意力的关键
)

spec.add_accelerator("sparse_attention_unit", cim_config)
```

## 步骤 3：自动化 RTL 生成
提交给 AI 原生设计流程：

```python
from openclaw import SiliconCompiler

compiler = SiliconCompiler(network="local")

# 这会触发"启蒙"AI 设计系统
job = compiler.compile(
 spec=spec,
 optimization_target="energy_efficiency",
 constraint_set="mobile_aggressive"
)

print(f"设计任务 ID: {job.id}")

# 监控 AI 生成的设计决策
async for decision in job.stream_decisions():
 print(f"[{decision.stage}] {decision.action}")
 print(f" 推理: {decision.ai_reasoning}")

"""
样本输出:
[Architecture] Selected hierarchical mesh NoC
 Reasoning: Reduce congestion for sparse traffic patterns
[Microarchitecture] Generated 4x CIM banks with shared ADCs
 Reasoning: Area efficiency while maintaining throughput
[RTL] Generated Chisel code: 15,000 lines
 Components: Sparse decoder, CIM arrays, Accumulators
"""
```

## 步骤 4：物理设计审查
RTL 完成后，系统自动运行物理设计：

```python
# 访问生成的布局查看器
layout = job.get_layout_view()

# 关键指标
print(f"芯片尺寸: {layout.die_size_mm2:.2f} mm²")
print(f"功耗: {layout.estimated_power_mw:.0f} mW")
print(f"最大频率: {layout.max_freq_mhz:.0f} MHz")
print(f"利用率: {layout.utilization_percent:.1f}%")

# 检查关键路径
critical_paths = layout.get_timing_report(n_worst=5)
for path in critical_paths:
 print(f"路径 {path.name}: {path.slack_ns:.3f}ns 余量")

"""
预期结果:
Die Size: 42.3 mm² (在 50 预算内)
Power: 8.5W (在 10W 预算内)
Max Frequency: 850 MHz
Utilization: 72.4%
"""
```

## 步骤 5：验证与签核
运行自动化验证：

```python
# 形式等价性检查
formal_check = job.run_formal_verification(
 reference_model="behavioral_python",
 timeout_hours=2
)

assert formal_check.equivalent, "设计与规格不等价!"

# 功耗分析
power_analysis = job.run_power_analysis(
 workload_traces="llm_inference_realistic.vcd"
)

print(f"动态功耗: {power_analysis.dynamic_mw} mW")
print(f"漏电功耗: {power_analysis.leakage_mw} mW")
```

## 步骤 6：导出用于流片
生成制造交付件：

```python
# GDSII + LEF + LIB
deliverables = job.export(
 format="gdsii",
 pdk="tsmc28hpc", # TSMC 28nm HPC+
 encryption_key=None # 设置安全 IP 交付
)

print(f"GDSII 文件: {deliverables.gds_cid}") # IPFS 哈希
print(f"LEF 文件: {deliverables.lef_cid}")
print(f"时序 Liberty: {deliverables.lib_cid}")

# 保存清单
deliverables.save_manifest("chip_v1_manifest.json")
```

## 完整脚本
组合所有步骤的完整脚本：

```python
#!/usr/bin/env python3
"""
完整的 CIM 芯片设计教程
"""
import asyncio
from openclaw import *

async def design_custom_chip():
 # 1. 分析工作负载
 profiler = WorkloadProfiler()
 analysis = profiler.analyze({
 "sparsity": 0.85,
 "bitwidth": 4,
 "pattern": "attention"
 })

 # 2. 创建规格
 spec = HardwareSpec(
 process_node="28nm",
 target_tflops=50,
 power_budget_w=10,
 area_budget_mm2=50,
 form_factor="m.2_2280"
 )

 # 3. 编译
 compiler = SiliconCompiler()
 job = await compiler.compile_async(spec)

 # 4. 等待完成
 result = await job.wait()

 if result.success:
 print(f"✅ 设计完成!")
 print(f" 性能: {result.tflops} TFLOPS")
 print(f" 功耗: {result.power_w}W")
 print(f" 效率: {result.tflops_per_watt} TFLOPS/W")

 # 导出
 result.export("./my_first_chip/")
 else:
 print(f"❌ 失败: {result.error_message}")

if __name__ == "__main__":
 asyncio.run(design_custom_chip())
```

## 您学到的内容

- 如何分析代理工作负载以进行硬件优化
- 自动化 CIM 架构生成
- 物理设计约束和权衡
- AI 生成硬件的验证方法

## 下一步

- [实时迁移](./live-migration.md): 了解如何将您的代理迁移到这个新芯片