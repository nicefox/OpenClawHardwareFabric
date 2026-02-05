# 故障排除常见问题

## 安装问题

### "ModuleNotFoundError: No module named 'openclaw'"

**原因**: SDK 未安装或 Python 路径问题。

**解决方案**:
```bash
# 验证安装
pip list | grep openclaw

# 如果缺失
pip install --upgrade openclaw-sdk

# 如果仍然失败，检查 Python 版本（需要 3.10+）
python --version

# 使用特定 Python 的 pip
python3.11 -m pip install openclaw-sdk
```

### Docker Compose 启动失败
错误：`ports are already allocated`

解决方案：
```bash
# 检查什么在使用端口 8545
lsof -i :8545

# 停止服务或使用不同端口
export LOCALCHAIN_PORT=8546
export IPFS_PORT=5002
docker-compose up -d
```

## 运行时错误

### "InsufficientFundsError: Evolution budget too low"
原因：代理钱包中没有足够的 USDC 进行存款。

解决方案：
```python
# 检查余额
balance = await client.wallet.get_balance()
print(f"Balance: {balance} USDC")

# 对于测试网，使用水龙头
curl -X POST https://faucet.openclaw.foundation \
 -d '{"address": "0x...", "amount": 10000}'
```

### "EvolutionSafetyError: Constitution Lock violated"
原因：提议的硬件设计试图：
- 修改自己的奖励函数
- 绕过透明度日志
- 超过最大演化速率

解决方案：
在异常中查看 `safety_report`：
```python
try:
 job = await client.submit_evolution_intent(gap)
except EvolutionSafetyError as e:
 print(f"Violation: {e.violation_type}")
 print(f"Details: {e.details}")
 print(f"Suggested fix: {e.suggested_alternative}")

 # 根据建议修改意图
 safe_intent = e.suggested_alternative
 job = await client.submit_evolution_intent(safe_intent)
```

### 迁移失败并出现 "Equivalence check failed"
原因：新硬件行为与旧硬件不同（回归）。

立即操作：系统自动回滚到旧硬件。

诊断：
```python
# 获取详细的失败报告
report = await client.migration.get_failure_report()
print(report.divergence_point) # 哪个指令/操作不同
print(report.expected_output) # 正确输出
print(report.actual_output) # 错误输出
```

常见修复：

- 精度不匹配：检查量化方案是否匹配
- 定时问题：新硬件太快/太慢，调整约束
- 内存排序：不同的一致性模型，添加围栏

## 性能问题

### EDA 编译缓慢（花费数小时）
原因：默认云资源不足以应对设计复杂性。

解决方案：
```python
# 请求更多计算资源
compiler = SiliconCompiler(
 cloud_config={
 "min_cores": 1000, # 从默认 100 扩展
 "priority": "high", # 预占式与专用
 "region": "us-west" # 靠近数据源
 }
)
```

### 瓶颈分析延迟高
原因：分析太多指标或历史记录过长。

优化：
```python
# 使用采样代替全追踪
gap = await client.analyze_bottlenecks(
 current_metrics=metrics,
 sampling_rate=0.1, # 分析 10% 的流量
 history_window="7d" # 而不是默认的 30d
)
```

## 网络与区块链

### "Nonce too low" 错误
原因：交易 nonce 与链不同步。

解决方案：
```python
# 重置 nonce 跟踪
await client.wallet.sync_nonce()

# 或手动设置
await client.wallet.set_nonce(await client.web3.eth.get_transaction_count(
 client.wallet.address,
 'pending'
))
```

### 获取设计文件时 IPFS 超时
原因：IPFS 网络拥塞或内容未固定。

解决方案：
```bash
# 检查 CID 是否可访问
ipfs dag stat QmAbC...

# 如果找不到，检查 Fabric 是否固定
openclaw fabric pin-check QmAbC...

# 如需要手动固定
ipfs pin add QmAbC...
```

## 调试工具

### 启用详细日志
```python
import logging
logging.basicConfig(level=logging.DEBUG)
logging.getLogger("openclaw").setLevel(logging.DEBUG)
```

### 检查服务健康状况
```python
# 完整系统诊断
health = await client.system.health_check()
print(health.blockchain_sync) # 链同步状态
print(health.ipfs_peers) # IPFS 连接性
print(health.eda_queue) # 云 EDA 作业队列深度
```

### 重置本地状态（危险！）
```bash
# 清除所有本地数据，重新开始
rm -rf ~/.openclaw/data/
docker-compose down -v
docker-compose up -d
```

## 获取帮助

如果问题持续存在：

- 检查日志：`docker-compose logs agent | grep ERROR`
- 运行诊断：`openclaw doctor`
- 提交错误：在 GitHub 问题中包含 `openclaw system report` 输出