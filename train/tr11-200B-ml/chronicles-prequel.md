# Prequel

Trials and Tribulations prior to the start of training.

For the trials and tribulation during the training see: [chronicles](chronicles.md).

# A100 experiments

200B

torch.optim.Adam:

16 nodes:
- 1st node:  61GB
- all nodes: 47GB
- performance: XXX

apex.optimizers.FusedAdam

16 nodes:
- 1st node:  51GB
- all nodes: 44GB
- performance: XXX



## Best TFLOPs

To measure best TFLOPs possible use a single, so that it uses all the intra-node connections (NVLink) and doesn't touch the network:

### fp16

- 1 node, 1 replica

20B model: TP=8, PP=1, NLAYERS=8, NHIDDEN=14400, NHEADS=32, SEQ_LEN=2048, VOCAB_LENGTH=250k, GBS=2048

```
 iteration        2/   95367 | consumed samples:         4096 | consumed tokens:      8388608 | elapsed time per iteration (s): 769.99 | learning rate: 3.787E-06 | global batch size:  2048 | lm loss: 6.384045E+01 | loss scale: 4096.0 | grad norm: 15906.210 | num zeros: 0.0 | number of skipped iterations:   0 | number of nan iterations:   0 | samples per second: 2.660 | TFLOPs: 108.47 |
```

- 10 nodes, 1 replica

200B model: TP=8, PP=10, NLAYERS=80, NHIDDEN=14400, NHEADS=96, SEQ_LEN=2048, VOCAB_LENGTH=250k, GBS=2048

```
 iteration        2/   95367 | consumed samples:         4096 | consumed tokens:      8388608 | elapsed time per iteration (s): 844.81 | learning rate: 3.787E-06 | global batch size:  2048 | lm loss: 6.373861E+01 | loss scale: 4096.0 | grad norm: 34132.119 | num zeros: 0.0 | number of skipped iterations:   0 | number of nan iterations:   0 | samples per second: 2.424 | TFLOPs: 98.87 |
```

- 20 nodes, 2 replicas

```
 iteration        2/   95367 | consumed samples:         4096 | consumed tokens:      8388608 | elapsed time per iteration (s): 430.21 | learning rate: 3.787E-06 | global batch size:  2048 | lm loss: 6.373876E+01 | loss scale: 4096.0 | grad norm: 34027.311 | num zeros: 0.0 | number of skipped iterations:   0 | number of nan iterations:   0 | samples per second: 4.761 | TFLOPs: 97.07 |
```

It was puzzling why much less memory was used for identical set up with DP=2 over DP=1 - but it's because of ZeRO-1 that saves a lot of memory across all GPUs!


| GPUs | Size | DP | TP | PP | MBS | Mem  | TFLOPs | Notes |
| ---: | ---: | -: | -: | -: | --: | ---: | -----: | ----: |
|    8 | 20B  |  1 |  8 |  1 |   1 | 67GB | 108.47 | 02-17 |
|   80 | 200B |  1 |  8 | 10 |   1 | 73GB |  98.87 | 02-17 |
|  160 | 200B |  2 |  8 | 10 |   1 | 51GB |  97.07 | 02-17 |
|      |      |    |    |    |     |      |        |       |

*Mem = max memory used by the first (last) nodes with the word embedding matrix - max is 77GB


### bf16

- 1 node, 1 replica

20B model: TP=8, PP=1, NLAYERS=8, NHIDDEN=14400, NHEADS=32, SEQ_LEN=2048, VOCAB_LENGTH=250k, GBS=2048

```
 iteration        2/   95367 | consumed samples:         4096 | consumed tokens:      8388608 | elapsed time per iteration (s): 777.09 | learning rate: 3.787E-06 | global batch size:  2048 | lm loss: 6.381926E+01 | loss scale: 1.0 | grad norm: 2.763 | num zeros: 0.0 | number of skipped iterations:   0 | number of nan iterations:   0 | samples per second: 2.635 | TFLOPs: 107.48 |
```


- 10 nodes, 1 replica

200B model: TP=8, PP=10, NLAYERS=80, NHIDDEN=14400, NHEADS=96, SEQ_LEN=2048, VOCAB_LENGTH=250k, GBS=2048

```
 iteration        2/   95367 | consumed samples:         4096 | consumed tokens:      8388608 | elapsed time per iteration (s): 853.81 | learning rate: 3.787E-06 | global batch size:  2048 | lm loss: 6.369443E+01 | loss scale: 1.0 | grad norm: 4.461 | num zeros: 0.0 | number of skipped iterations:   0 | number of nan iterations:   0 | samples per second: 2.399 | TFLOPs: 97.82 |
```


- 20 nodes, 2 replicas


```
 iteration        2/   95367 | consumed samples:         4096 | consumed tokens:      8388608 | elapsed time per iteration (s): 434.14 | learning rate: 3.787E-06 | global batch size:  2048 | lm loss: 6.369444E+01 | loss scale: 1.0 | grad norm: 6.314 | num zeros: 0.0 | number of skipped iterations:   0 | number of nan iterations:   0 | samples per second: 4.717 | TFLOPs: 96.19 |
```


| GPUs | Size | DP | TP | PP | MBS | Mem  | TFLOPs | Notes |
| ---: | ---: | -: | -: | -: | --: | ---: | -----: | ----: |
|    8 | 20B  |  1 |  8 |  1 |   1 | 68GB | 107.48 | 02-17 |
|   80 | 200B |  1 |  8 | 10 |   1 | 75GB |  97.82 | 02-17 |
|  160 | 200B |  2 |  8 | 10 |   1 | 53GB |  96.19 | 02-17 |
|      |      |    |    |    |     |      |        |       |

*Mem = max memory used by the first (last) nodes with the word embedding matrix - max is 77GB

So we can load more stages as we get higher DP as ZeRO spreads out over more gpus - smaller shards.



## dealing with JZ hanging on the large model

This overcomes the hanging which in general should lead to a slower throughput since all CUDA operations become synchronous and would block until they are done.

```
export CUDA_LAUNCH_BLOCKING=1
```

200B, measuring 2nd iter:

| GPUs | async |  GBS | TFLOPs | Notes        |
| ---: | ----: | ---: | -----: | -----------: |
|   80 | no    |  512 |  91.04 |              |
|   80 | yes   |  512 |  97.20 |              |
|  160 | no    |  512 |  84.59 |              |
|  160 | yes   |  512 |  84.44 |              |
|  160 | no    | 2048 |  90.29 |              |
|  160 | yes   | 2048 |  90.25 | may hang     |
|  320 | no    | 2048 |  87.78 |              |
|  320 | yes   | 2048 |   xxxx | always hangs |
|      |       |      |        |              |

async/yes == `CUDA_LAUNCH_BLOCKING=0`

Interesting. Sometimes `CUDA_LAUNCH_BLOCKING=1` impacts the speed, at other times it doesn't. Perhaps with larger set ups it's barely impacting since there is a lot more comms than the small setup.


## Choosing the fastest 3D Topology

Benchmarking the fastest 3D topology. Constraint: can use at most 48 nodes of 8 gpu a100 80gb nodes.

Note that we want not the highest TFLOPs but the highest speed per iteration, since one can get high TFLOPs on less GPUs and overall slower speed, since we only care about how fast we can finish the training.

Also note that the model size isn't always the same as the number of layers had to be tweaked to fit PP and NHIDDEN was fixed - so speed/tflops can't be exactly compared - but can be brought back to the same size by tweaking NHIDDEN. also since for efficiency of finishing this process I take the snapshot of a single iteration (always 2nd) the data isn't exact and can fluctuate a bit. But the point of this exercise is to get a feel of which topology is superior.


| Nodes | Size | DP | TP | PP | MBS |  GBS | Mem  | Sec/it | TFLOPs | Notes |
| ----: | ---: | -: | -: | -: | --: |  --: | ---: | -----: | -----: | ----: |
|    48 | 200B | 12 |  8 |  4 |   1 | 2040 | 47GB | 189.06 |  91.67 | 02-20 |
|    45 | 200B |  9 |  8 |  5 |   1 | 2043 | 44GB | 208.40 |  88.84 | 02-20 |
|    48 | 194B |  8 |  8 |  6 |   1 | 2048 | 39GB | 183.64 |  92.38 | 02-20 |
|    42 | 191B |  6 |  8 |  7 |   1 | 2046 | 39GB | 202.99 |  94.20 | 02-20 |
|    48 | 200B |  6 |  8 |  8 |   1 | 2046 | 36GB | 185.75 |  93.59 | 02-20 |
|    45 | 205B |  5 |  8 |  9 |   1 | 2045 | 37GB | 199.14 |  94.23 | 02-20 |
|    40 | 200B |  4 |  8 | 10 |   1 | 2048 | 35GB | 221.21 |  94.39 | 02-20 |
|    44 | 195B |  4 |  8 | 11 |   1 | 2048 | 32GB | 197.15 |  92.67 | 02-20 |
|    48 | 183B |  4 |  8 | 12 |   1 | 2048 | 30GB | 172.40 |  90.84 | 02-20 |
|       |      |    |    |    |     |      |      |        |        |       |

* Sec/it throughput at iteration 2

As you can see the 80GB is totally unnecessary for MBS=1 as we are bound by compute of each gpu and we barely use half the gpu memory and trying to pack more on each gpu slows the ensemble down. This is of course thanks to ZeRO which shards all fp32 optim+grad+params over all gpus - so the more gpus you use the less memory is needed to accomodate the same model size, regardless of DP/TP/PP topology. (with MBS=1 that is so that the activations don't take too much memory)

This table doesn't take into account batch size rampup which needs to be divisible by DP as it progressed from 32, 64, ... so really we have an additional constraint of `DP % 4 = 0` and `GBS % 32 = 0`.

which means from the above list only a few configs are suitable, and these are:

| Nodes | Size | DP | TP | PP | MBS |  GBS | Mem  | Sec/it | TFLOPs | Notes |
| ----: | ---: | -: | -: | -: | --: |  --: | ---: | -----: | -----: | ----: |
|    48 | 194B |  8 |  8 |  6 |   1 | 2048 | 39GB | 183.64 |  92.38 | 02-20 |
|    40 | 200B |  4 |  8 | 10 |   1 | 2048 | 35GB | 221.21 |  94.39 | 02-20 |
|    44 | 195B |  4 |  8 | 11 |   1 | 2048 | 32GB | 197.15 |  92.67 | 02-20 |
|    48 | 183B |  4 |  8 | 12 |   1 | 2048 | 30GB | 172.40 |  90.84 | 02-20 |
|       |      |    |    |    |     |      |      |        |        |       |

Increasing MBS will speed up things a bit and we have a ton of spare memory to accommodate a larger MBS, but have to ensure we get the batch size ramp up sorted out. So if the rampup steps are in increments of 32 with DP=4 highest MBS is 8. and `log2(MBS) % 2 = 0`.


| Nodes | Size | DP | TP | PP | MBS |  GBS | Mem  | Sec/it | TFLOPs | Notes |
| ----: | ---: | -: | -: | -: | --: |  --: | ---: | -----: | -----: | ----: |
|    48 | 194B |  8 |  8 |  6 |   1 | 2048 | 39GB | 183.64 |  92.38 | 02-20 |
|    48 | 194B |  8 |  8 |  6 |   2 | 2048 | 45GB | 172.36 |  98.43 | 02-20 |
|    48 | 194B |  8 |  8 |  6 |   4 | 2048 | 56GB | 173.92 |  97.55 | 02-20 |
|    48 | 194B |  8 |  8 |  6 |   8 | 2048 | 75GB | 192.42 |  88.17 | 02-20 |
|       |      |    |    |    |     |      |      |        |        |       |


| Nodes | Size | DP | TP | PP | MBS |  GBS | Mem  | Sec/it | TFLOPs |                  Notes |
| ----: | ---: | -: | -: | -: | --: |  --: | ---: | -----: | -----: | ---------------------: |
|    40 | 200B |  4 |  8 | 10 |   1 | 2048 | 35GB | 221.21 |  94.39 |                  02-20 |
|    40 | 200B |  4 |  8 | 10 |   2 | 2048 | 43GB | 207.92 | 100.43 |                  02-20 |
|    40 | 200B |  4 |  8 | 10 |   4 | 2048 | 55GB | 208.18 | 100.30 |                  02-20 |
|    40 | 200B |  4 |  8 | 10 |   8 | 2048 | 76GB | 229.69 |  90.91 | 02-20 too close to OOM |
|       |      |    |    |    |     |      |      |        |        |                        |


| Nodes | Size | DP | TP | PP | MBS |  GBS | Mem  | Sec/it | TFLOPs | Notes |
| ----: | ---: | -: | -: | -: | --: |  --: | ---: | -----: | -----: | ----: |
|    44 | 195B |  4 |  8 | 11 |   1 | 2048 | 32GB | 197.15 |  92.67 | 02-20 |
|    44 | 195B |  4 |  8 | 11 |   2 | 2048 | 41GB | 186.65 |  97.89 | 02-20 |
|    44 | 195B |  4 |  8 | 11 |   4 | 2048 | 53GB | 185.79 |  98.34 | 02-20 |
|    44 | 195B |  4 |  8 | 11 |   8 | 2048 | 75GB | 206.62 |  88.42 | 02-20 |
|       |      |    |    |    |     |      |      |        |        |       |


| Nodes | Size | DP | TP | PP | MBS |  GBS | Mem  | Sec/it | TFLOPs | Notes |
| ----: | ---: | -: | -: | -: | --: |  --: | ---: | -----: | -----: | ----: |
|    48 | 183B |  4 |  8 | 12 |   1 | 2048 | 30GB | 172.40 |  90.84 | 02-20 |
|    48 | 183B |  4 |  8 | 12 |   2 | 2048 | 39GB | 161.96 |  96.69 | 02-20 |
|    48 | 183B |  4 |  8 | 12 |   4 | 2048 | 50GB | 163.32 |  95.89 | 02-20 |
|       |      |    |    |    |     |      |      |        |        |       |

The models are slightly different in size so can't compare absolute numbers.

But clearly MBS=2 is about the best, MBS=4 is close by.

If we utilize all 48 nodes then we have PP6 and PP12 as contenders.


## tile and wave quantization


A100 80GB has 108 SMs

https://docs.nvidia.com/deeplearning/performance/dl-performance-matrix-multiplication/index.html#tile-quant

```
nhidden % 128 = 0
```

https://docs.nvidia.com/deeplearning/performance/dl-performance-matrix-multiplication/index.html#wave-quant

```
nhidden % 108 = 0
```

TP=8:

```
nhidden % 8 = 0
```

Combining all 3:

```
nhidden = 108*8*c = 864*c
```

which gives 864*16 = 13824 (187B) => so let's try to compare with 14400 (200B)

XXX: This is a total guestimate - need proper math

| Nodes | Size | NHIDDEN | DP | TP | PP | MBS |  GBS | Mem  | Sec/it | TFLOPs | Notes |
| ----: | ---: |    ---: | -: | -: | -: | --: |  --: | ---: | -----: | -----: | ----: |
|    40 | 200B |   14400 |  4 |  8 | 10 |   1 | 2048 | 35GB | 221.21 |  94.39 | 02-20 |
|    40 | 187B |   13824 |  4 |  8 | 10 |   1 | 2048 | 33GB | 160.29 | 120.05 | 02-20 |
|    40 | 187B |   13824 |  4 |  8 | 10 |   2 | 2048 | 39GB | 151.07 | 127.38 | 02-20 |
|    40 | 187B |   13824 |  4 |  8 | 10 |   4 | 2048 | 53GB | 147.43 | 130.53 | 02-20 |
|    40 | 187B |   13824 |  4 |  8 | 10 |   4 | 2048 | 73GB | 152.51 | 126.18 | 02-20 |
|       |      |         |    |    |    |     |      |      |        |        |       |


## TFLOPs calculation improved

Until now we used an estimated TFLOPs calculator which was under-reporting the real TFLOPs. And we couldn't compare those to the TFLOPs reported by [Megatron-LM](https://github.com/NVIDIA/Megatron-LM#readme).

Deepak Narayanan fixed this here: https://github.com/bigscience-workshop/Megatron-DeepSpeed/pull/251

So from here on all the TLOPs reports will be about 3% higher - so can't exactly compare to the earlier numbers in this document.


## 48 node contenders

So we have 2 set ups that fit well into 48 nodes - and that's PP=6/DP=8 or PP=12/DP=4


| Nodes | Size | DP | TP | PP | MBS |  GBS | Mem  | Sec/it | TFLOPs | Notes |
| ----: | ---: | -: | -: | -: | --: |  --: | ---: | -----: | -----: | ----: |
|    48 | 181B |  4 |  8 | 12 |   1 | 2048 | GB   |        |        | 02-20 |
|    48 | 181B |  4 |  8 | 12 |   2 | 2048 | GB   |        |        | 02-20 |
|    48 | 181B |  4 |  8 | 12 |   4 | 2048 | 49GB | 123.69 | 130.34 | 02-20 |
|    48 | 181B |  4 |  8 | 12 |   8 | 2048 | 69GB | 129.26 | 124.72 | 02-20 |
|       |      |    |    |    |     |      |      |        |        |       |


| Nodes | Size | DP | TP | PP | MBS |  GBS | Mem  | Sec/it | TFLOPs | Notes |
| ----: | ---: | -: | -: | -: | --: |  --: | ---: | -----: | -----: | ----: |
|    48 | 181B |  8 |  8 |  6 |   1 | 2048 | GB   |        |        | 02-20 |
|    48 | 181B |  8 |  8 |  6 |   2 | 2048 | GB   |        |        | 02-20 |
|    48 | 181B |  8 |  8 |  6 |   4 | 2048 | GB   |        |        | 02-20 |
|    48 | 181B |  8 |  8 |  6 |   8 | 2048 | GB   |        |        | 02-20 |
|       |      |    |    |    |     |      |      |        |        |       |