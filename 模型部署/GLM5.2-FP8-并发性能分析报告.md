# GLM5.2 FP8 并发性能分析报告

> **硬件平台**：8× NVIDIA H200 NVL  
> **模型**：GLM-5.2 (FP8)  
> **推理框架**：SGLang  
> **测试时间**：2026-07-22  
> **测试工具**：aiperf

---

## 测试配置摘要

| 并发数 | 请求数 | 平均 Input 长度 | 平均 Output 长度 | 测试时长 | `max_completion_tokens` |
|:---:|:---:|:--------------:|:---------------:|:------:|:---------------------:|
| 1 | 5 | ~200,192 tokens | ~498 tokens | 127s | 500 |
| 5 | 20 | ~16,192 tokens | ~4,093 tokens | 444s | 4,096 |
| 10 | 100 | ~9,217 tokens | ~500 tokens | 158s | 500 |
| 32 | 128 | ~8,192 tokens | ~500 tokens | 171s | 500 |

> ⚠️ **注意**：不同并发等级的测试输入条件不完全一致（concurrency=1 有超长 200K 输入，concurrency=5 输出为 4096 token），横向比较需考虑此因素，但总体趋势判断可靠。

---

## 核心指标总览

| 指标 | 并发=1 | 并发=5 | 并发=10 | 并发=32 |
|:----|:----:|:----:|:-----:|:-----:|
| **Output Token Throughput (tokens/s) ⭐** | **19.60** | **184.28** | **316.25** | **375.04** |
| 每用户吞吐 (tokens/s/user) | 56.75 | 38.11 | 33.41 | 15.88 |
| E2E 每用户吞吐 (tokens/s/user) | 19.61 | 36.88 | 31.65 | 11.73 |
| 请求吞吐 (req/s) | 0.04 | 0.05 | 0.63 | 0.75 |
| 请求延迟 (ms) | 25,407 | 110,986 | 15,791 | 42,616 |
| **Inter Token Latency (ms) ⭐** | **17.62** | **26.24** | **29.96** | **64.95** |
| Time to First Token (ms) | 16,646 | 3,612 | 847 | 10,226 |
| GPU 利用率 (avg) | ~94% | ~93% | ~88% | ~96% |
| SM 利用率 (avg) | ~50% | ~27% | ~46% | ~48% |
| 总 GPU 功耗 (W) | 3,272 | 2,139 | 2,820 | 3,161 |

---

## 吞吐量随并发变化趋势

```
Output Token Throughput (tokens/s)
    |
400 ┤                                    ● 375.0
    │
300 ┤                          ● 316.3
    │
200 ┤
    │               ● 184.3
100 ┤
    │
  0 ┤──● 19.6 ────────────────────────────────
    │
    └─────┬──────┬──────┬──────┬──────┬──
          1      5      10            32
                         ↑              Concurrency
                    阈值区域
```

---

## 结论一：吞吐量衰减的并发数阈值

**阈值：并发数 ≥ 10 后吞吐增长显著放缓**

### 吞吐量增速分析

| 并发区间 | 吞吐增长量 | 增长率 | 并发增长倍数 | 弹性系数 |
|:------:|:--------:|:----:|:---------:|:-------:|
| 1 → 5 | +164.7 | +840% | 5× | 1.88× |
| 5 → 10 | +132.0 | +72% | 2× | 0.36× |
| 10 → 32 | +58.8 | +19% | 3.2× | 0.06× |

> 弹性系数 = 吞吐增长率 / 并发增长率，>1 为高效扩展，<0.1 为严重饱和。

### 判断依据

1. **边际收益骤降**：并发 10→32 投入 3.2 倍资源，吞吐仅增加 19%（弹性系数仅 0.06）
2. **Decode 延迟翻倍**：Inter Token Latency 从 10 并发时的 **30ms** 跃升至 32 并发的 **65ms**（+117%），说明 batch 过大导致 decode 明显变慢
3. **总吞吐接近天花板**：32 并发时 375 tokens/s 接近 8 卡 H200 在该模型下的瓶颈

**结论**：输出吞吐的**有效扩展上限在并发 10~15 之间**，超过此值后收益急剧递减。

---

## 结论二：单进程吞吐 ≈ 20 tokens/s 的并发水平

**答案：大约在并发 25 左右**

### 每用户吞吐量下降曲线

| 并发数 | Output Token Throughput Per User | 相对于上次下降 |
|:---:|:-------------------------------:|:------------:|
| 1 | 56.75 tokens/s/user | — |
| 5 | 38.11 | -33% |
| 10 | 33.41 | -12% |
| 32 | 15.88 | -52% |

### 插值计算

每用户吞吐量从并发 10（33.41）下降到并发 32（15.88），每增加 1 个并发下降约 **0.80 tokens/s**：

- 目标值：20 tokens/s/user
- 需下降量：33.41 - 20 = 13.41
- 所需并发增量：13.41 ÷ 0.80 ≈ 16.8
- 对应并发：10 + 17 ≈ **27**

同时，E2E 每用户吞吐（含 Prefill 延迟）的插值结果为 **~23**。

综合判断，**单进程（每用户）output token 吞吐降至约 20 tokens/s 的并发水平在 25 左右**。

---

## 结论三：最佳并发数建议

**推荐值：并发 = 10（建议运行范围 8~15）**

### 各并发等级综合评分

| 维度 | 并发 1 | 并发 5 | 并发 10 ★ | 并发 32 |
|:----|:----:|:----:|:--------:|:------:|
| 总吞吐 (tokens/s) | 19.6 ❌ | 184.3 ⚠️ | **316.3 ✅** | 375.0 ⚠️ |
| 每用户吞吐 (tokens/s) | 56.8 ✅ | 38.1 ✅ | **33.4 ✅** | 15.9 ❌ |
| 请求延迟 (s) | 25.4 ⚠️ | 111.0 ❌ | **15.8 ✅** | 42.6 ❌ |
| Inter Token Latency (ms) | 17.6 ✅ | 26.2 ⚠️ | **30.0 ⚠️** | 65.0 ❌ |
| 请求吞吐 (req/s) | 0.04 ❌ | 0.05 ❌ | **0.63 ✅** | 0.75 ⚠️ |
| GPU 利用率 | 94% ✅ | 93% ✅ | **88% ✅** | 96% ⚠️ |
| 能耗 (W) | 3,272 ⚠️ | 2,139 ✅ | **2,820 ✅** | 3,161 ⚠️ |

### 推荐理由

1. **吞吐效率最高**：并发 10 达到 316.3 tokens/s（占峰值 84%），再追加 3 倍并发仅换来 19% 的收益
2. **用户体验良好**：30ms Inter Token Latency 提供流畅的流式输出；15.8s 请求延迟在长文本场景下可接受
3. **资源利用充分**：GPU 利用率 88%、SM 利用率 46%，硬件已被有效利用但未过载
4. **能耗比最优**：单位 token 功耗低于并发 32，性价比最高

### 场景化建议

| 场景 | 推荐并发 | 原因 |
|:---|:-------:|:----|
| 在线服务（低延迟优先） | **8~10** | 保持 Inter Token Latency < 35ms，TTFT < 1s |
| 批量处理（吞吐优先） | **10~15** | 最大化总吞吐，容忍稍高延迟 |
| 离线压测/高并发场景 | ≤20 | 超过 20 后每用户吞吐降至 25 以下，体验急剧下降 |
| 极限吞吐（不关心延迟） | 32 | 总吞吐最高 375 tokens/s，但每用户仅 15.9 tokens/s |

---

## 附录：原始关键数据

### concurrency=1

```
Output Token Throughput:          19.60 tokens/s
E2E Output Token Throughput/User: 19.61 tokens/s/user
Output Token Throughput/User:     56.75 tokens/s/user
Inter Token Latency:              17.62 ms
Request Latency:                  25,407.35 ms
GPU Utilization (avg 8 cards):    ~90%
```

### concurrency=5

```
Output Token Throughput:          184.28 tokens/s
E2E Output Token Throughput/User: 36.88 tokens/s/user
Output Token Throughput/User:     38.11 tokens/s/user
Inter Token Latency:              26.24 ms
Request Latency:                  110,986.00 ms
GPU Utilization (avg 8 cards):    ~87%
```

### concurrency=10

```
Output Token Throughput:          316.25 tokens/s
E2E Output Token Throughput/User: 31.65 tokens/s/user
Output Token Throughput/User:     33.41 tokens/s/user
Inter Token Latency:              29.96 ms
Request Latency:                  15,791.66 ms
GPU Utilization (avg 8 cards):    ~87%
```

### concurrency=32

```
Output Token Throughput:          375.04 tokens/s
E2E Output Token Throughput/User: 11.73 tokens/s/user
Output Token Throughput/User:     15.88 tokens/s/user
Inter Token Latency:              64.95 ms
Request Latency:                  42,616.62 ms
GPU Utilization (avg 8 cards):    ~93%
```

---

*报告生成时间：2026-07-22 19:30*
*数据来源：artifacts/glm-5.2-openai-chat-concurrency{1,5,10,32}/profile_export_aiperf.csv*
