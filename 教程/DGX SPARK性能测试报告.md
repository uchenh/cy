# DGX SPARK性能测试报告

DGX SPARK性能测试报告


| 组成 | 配置 |
|---|---|
| 显卡 | 英伟达Blackwell GB10 |
| 处理器 | 20核Grace处理器（10个Cortex-X925性能核+10个Cortex-A725能效核） |
| 内存 （CPU+GPU共用） | 128G 低功耗版 DDR5统一内存 \| 4266 MHz \| 16通道 |
| 存储 | 1 TB 或 4 TB NVMe M.2存储 |
| 网络 | 1x10 GbE电口 \| 1xConnectX-7网卡 双端口200G \| Wi-Fi 7 \| 蓝牙 5.4 |
| 接口 | 4xUSB Type-C \| 1xHDMI |
| 视频处理引擎 | 1x视频编码引擎,1x视频解码引擎 |
| 尺寸 | 长150mm 宽150mm 高50.5mm |
| 电源 | 240W |


| 模块 | 详细信息 |
|---|---|
| 操作系统 | NVIDIA DGX OS 7.2.3（定制化Ubuntu 24.04） |
| 驱动版本 | 580.95.05 |
| CUDA版本 | 13.0.2 |
| 预装软件 | NVIDIA DGX OS、CUDA,cuDNN、Docker、 NVIDIA Container Runtime、NGC |

内存带宽测试

工具:STREAM

实测内存总带宽约为：120GB/s

测试结果参考下图

处理器性能测试

工具：SYSBENCH

单核Events per second分数：5615.58

多核Events per second分数：88603.31

测试结果参考下图

硬盘带宽性能测试

顺序读总带宽：8635MB/s

使用命令：fio --name=seqread --filename=/testfile --size=100G --direct=1     --bs=4k --ioengine=libaio --iodepth=64 --numjobs=4 --rw=read

顺序写总带宽：8743MB/s

使用命令：fio --name=seqwrite --filename=/testfile --size=100G --direct=1     --bs=1M --ioengine=libaio --rw=write

测试结果参考下图

双机带宽性能测试

针对大模型领域，我们进行了多种场景的双机带宽性能测试，结果如下：

1、输出收集、特征聚合场景：

平均总线带宽：15.1365GB/s

使用命令：mpirun -np 2 -H node1:1,node2:1   --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"   -x LD_LIBRARY_PATH=$LD_LIBRARY_PATH   $HOME/nccl-tests/build/all_gather_perf -b 1M -e 2G -f 2 -g 1

2、梯度同步场景：

平均总线带宽：14.4222GB/s

使用命令：mpirun -np 2 -H node1:1,node2:1   --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"   -x LD_LIBRARY_PATH=$LD_LIBRARY_PATH   $HOME/nccl-tests/build/all_reduce_perf -b 1M -e 2G -f 2 -g 1

3、参数初始化场景：

平均总线带宽：17.4517GB/s

使用命令：mpirun -np 2 -H node1:1,node2:1   --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"   -x LD_LIBRARY_PATH=$LD_LIBRARY_PATH   $HOME/nccl-tests/build/broadcast_perf -b 1M -e 2G -f 2 -g 1

4、数据归约场景：

平均总线带宽：16.2659GB/s

使用命令：mpirun -np 2 -H node1:1,node2:1   --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"   -x LD_LIBRARY_PATH=$LD_LIBRARY_PATH   $HOME/nccl-tests/build/reduce_perf -b 1M -e 2G -f 2 -g 1

5、模型并行场景：

平均总线带宽：17.2241GB/s

使用命令：mpirun -np 2 -H node1:1,node2:1   --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"   -x LD_LIBRARY_PATH=$LD_LIBRARY_PATH   $HOME/nccl-tests/build/alltoall_perf -b 1M -e 2G -f 2 -g 1

测试结果参考下图

模型推理性能实测

测试场景一：

推理引擎：VLLM

模型：Llama-3.1-8B-Instruct-FP4

单并发吞吐量：32tokens/s

10并发吞吐量：263tokens/s

20并发吞吐量：373tokens/s

50并发吞吐量：575tokens/s

100并发吞吐量：582tokens/s

测试结果参考下图

测试场景二：

推理引擎：VLLM

模型：Qwen3-8B-FP4

单并发吞吐量：31tokens/s

10并发吞吐量：248tokens/s

20并发吞吐量：377tokens/s

50并发吞吐量：524tokens/s

100并发吞吐量：541tokens/s

测试结果参考下图

测试场景三：

推理引擎：VLLM

模型：Qwen3-14B-FP4

单并发吞吐量：19tokens/s

10并发吞吐量：163tokens/s

50并发吞吐量：386tokens/s

100并发吞吐量：396tokens/s

测试结果参考下图

模型训练性能实测

在这里分别使用了单机和双机DGX SPARK进行RESNET18和RESNET50的训练测试，并记录使用同样1281280张图片的数据集进行1轮Epoch训练所花费的时间。

测试一

模型：RESNET18

Epoch：1

数据集规模：1281280张图片

单机总耗时：2717.06秒

双机总耗时：1203.16秒

测试结果参考下图

测试二

模型：RESNET50

Epoch：1

数据集规模：1281280张图片

单机总耗时：7298.37秒

双机总耗时：3872.33秒

测试结果参考下图

测评总结：从上面的结果可以看出，DGX SPARK整体有的不错的性能表现，对于开发者而言，它为用户随时随地运行大模型相关任务提供了可能。并且DGX SPARK内置的NVIDIA定制系统，提供了模型相关任务所需的工具集，能够让用户实现开箱即用。DGX SPARK具有较大的显存，并且在低精度场景有着不错的性能表现，很适合处理低精度模型的微调、推理任务。


> 本文档含 31 张插图，见 `assets/DGX SPARK性能测试报告/`。

![图 1](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img1.png)

![图 2](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img2.png)

![图 3](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img3.png)

![图 4](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img4.png)

![图 5](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img5.png)

![图 6](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img6.png)

![图 7](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img7.png)

![图 8](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img8.png)

![图 9](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img9.png)

![图 10](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img10.png)

![图 11](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img11.png)

![图 12](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img12.png)

![图 13](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img13.png)

![图 14](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img14.png)

![图 15](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img15.png)

![图 16](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img16.png)

![图 17](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img17.png)

![图 18](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img18.png)

![图 19](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img19.png)

![图 20](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img20.png)

![图 21](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img21.png)

![图 22](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img22.png)

![图 23](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img23.png)

![图 24](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img24.png)

![图 25](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img25.png)

![图 26](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img26.png)

![图 27](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img27.png)

![图 28](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img28.png)

![图 29](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img29.png)

![图 30](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img30.png)

![图 31](assets/DGX%20SPARK%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A/img31.png)
