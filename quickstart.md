# 快速开始

本指南将引导您完成 OpenClaw 硬件演化系统的安装和首次硬件演化。

## 系统要求

- Python 3.10+
- Docker 和 Docker Compose
- 8GB 可用内存
- 10GB 可用磁盘空间

## 安装

### 1. 验证安装
```bash
openclaw --version
# 应输出: openclaw-sdk 0.1.0-alpha
```

### 2. 启动本地开发环境
我们提供了一套 Docker Compose 设置，包含所有依赖项：

```bash
# 克隆仓库
git clone https://github.com/openclaw-foundation/hardware-fabric.git
cd hardware-fabric

# 启动本地区块链、IPFS 和模拟服务
docker-compose up -d

# 检查状态
docker-compose ps
```

这将启动：

- 本地以太坊节点 (Anvil) 在端口 8545
- 用于存储设计文件的 IPFS 节点
- 模拟云端芯片设计的模拟 EDA 服务
- 在端口 3000 的代理监控仪表板

### 3. 初始化您的代理
创建新的代理身份 (DID):

```python
# init_agent.py
from openclaw import AgentIdentity

# 创建具有 TEE 支持的身份 (本地模式下模拟)
identity = AgentIdentity.create(
 name="my-first-agent",
 network="local",
 enable_tee=False # 生产环境中设置为 True (SGX/SEV)
)

identity.save("my_agent.json")
print(f"Agent DID: {identity.did}")
```

运行它：
```bash
python init_agent.py
# 输出: Agent DID: did:oc:0x742d35Cc6634C0532925a3b844Bc9e7595f...
```

### 4. 执行第一次演化
创建 `first_evolution.py`:

```python
import asyncio
from openclaw import AgentHardwareClient

async def main():
 # 初始化客户端
 client = AgentHardwareClient(
 identity_path="my_agent.json",
 network="local"
 )

 # 模拟当前性能不佳
 metrics = {
 "throughput": 10, # 非常慢
 "latency_ms": 500,
 "power_consumption": 300
 }

 print("🔍 分析瓶颈...")
 gap = await client.analyze_bottlenecks(
 current_metrics=metrics,
 target_capabilities=["edge_inference"]
 )

 print(f"紧急程度: {gap.urgency_score:.2f}")

 if gap.urgency_score > 0.7:
 print("🧬 开始硬件演化...")
 job = await client.submit_evolution_intent(gap)

 # 监控进度
 async for milestone in job.stream_milestones():
 print(f" {milestone['stage']}: {milestone['completion']}%")

 print("✅ 演化完成！查看仪表板 http://localhost:3000")
 else:
 print("✅ 软件优化足够")

if __name__ == "__main__":
 asyncio.run(main())
```

运行：
```bash
python first_evolution.py
```

### 5. 在仪表板上验证

- 实时设计进度
- 资源利用率
- 智能合约状态

## 下一步

- [代理初始化教程](../tutorials/agent-initialization.md)
- [瓶颈分析教程](../tutorials/bottleneck-analysis.md)
- [自定义芯片设计教程](../tutorials/custom-chip-design.md)

## 故障排除

### 端口冲突
如果端口 8545 被占用：
```bash
export LOCALCHAIN_PORT=8546
docker-compose up -d
```

### Python 版本问题
我们要求 Python 3.10+。使用以下命令检查：
```bash
python --version
```

如果需要，使用 pyenv：
```bash
pyenv install 3.11.0
pyenv local 3.11.0
```

### Docker 权限被拒绝
将您的用户添加到 docker 组：
```bash
sudo usermod -aG docker $USER
# 注销并重新登录
```