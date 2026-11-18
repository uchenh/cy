# 4 卡 RTX PRO 5000 NCCL 带宽约 20 GB/s 说明

## 结论

4 张 RTX PRO 5000 一起跑 NCCL，平均 bus 带宽是 **20.5 GB/s**。原因是主板第 4 个显卡槽实际走的是 **PCIe 4.0 x16**。四卡通信看最慢的那条链路，整组就被这个槽拖在 20 GB/s 这一档。

## 这台机器

- 主板：技嘉 TRX50 AI TOP
- CPU：AMD Ryzen Threadripper 9960X（非 PRO）
- 显卡：NVIDIA RTX PRO 5000 48GB Blackwell × 4
- 每张显卡自己的接口：PCIe 5.0 x16

## 测试结果

命令：

```bash
./build/all_reduce_perf -b 8M -e 1G -f 2 -g 4
```

低配（本机，4 卡 PRO 5000）：平均 bus 带宽 **20.46 GB/s**，数据校验通过。

同一种测法，中配（4 卡 PRO 6000D，8 个槽都是 PCIe 5.0 x16）：平均 bus 带宽 **34.49 GB/s**。

## 主板插槽

TRX50 AI TOP 的四个槽并不一样：

| 插槽 | 速率 |
| --- | --- |
| PCIEX16_1、PCIEX16_2、PCIEX16_3 | PCIe 5.0 x16 |
| PCIEX16_4 | 配 Threadripper PRO 7000 时为 PCIe 5.0 x16；配非 PRO 时为 PCIe 4.0 x16 |

9960X 不是 PRO。第 4 张卡插在 PCIEX16_4，链路是 PCIe 4.0 x16。前三张卡虽然是 PCIe 5.0 x16，也带不动这一条慢链路。

## 为什么正好是 20

PCIe 4.0 x16 单向理论大约 32 GB/s。四张卡做 allreduce 时要同时收发，实测 bus 带宽常见就是 **20 GB/s 左右**。这次的 20.5 GB/s 落在这一档。

PCIe 5.0 x16 的四卡 allreduce 常见是 **34 GB/s 以上**。中配那台就是这一档。

## 显卡本身

RTX PRO 5000 的对外接口是 PCIe 5.0 x16，和 PRO 6000 相同。显存带宽是 1344 GB/s，远高于这次测到的 20 GB/s。20 和 34 的差别来自主板第 4 槽，不来自显卡型号。

## 补充

技嘉后续规格写明：BIOS 升到 F10 或更新版本后，9960X 这种 9000 系列可以把 PCIEX16_4 训练成 PCIe 5.0 x16。本次测到 20.5 GB/s，说明测试时第 4 条链路没有跑在 PCIe 5.0。测试之后机器无法再登录，没有用 `nvidia-smi` 复核协商速率。

日志里还有一行 GDRCopy 不可用。那是一条可选传输路径没加载，改变不了「一条 PCIe 4.0 拖住四卡」这个结果。

## 机器恢复后可以这样复核

```bash
nvidia-smi --query-gpu=index,pcie.link.gen.current,pcie.link.width.current --format=csv
```

第 4 张卡如果是 `gen.current = 4`、`width.current = 16`，就和本说明一致。四张卡都是 `5` 和 `16` 时，同样的测试应接近或高于 34 GB/s。
