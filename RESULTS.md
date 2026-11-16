# Results

Meta Quest 3S (Snapdragon XR2 Gen 2, Adreno 740), Horizon OS, LFM3-Omni-2B Q8_0, llama.cpp `9f0e509`, November 2026.
Milliseconds per 80 ms audio frame, headset awake, 4 threads unless noted.

## Stages

| build | total | backbone | depthformer | codec |
|---|---|---|---|---|
| stock, on battery | 245.3 | 152.4 | 74.9 | 17.2 |
| steps 1 to 2, CPU only, plugged in | 108.5 | 75.7 | 24.9 | 7.8 |
| steps 1 to 4, plugged in | 77.4 | 50.1 | 20.0 | 7.2 |
| steps 1 to 5 (Q8 only), plugged in | 70.0 | 50.7 | 19.3 | 0.1 |
| all six steps, plugged in | 68.1 | 50.7 | 17.3 | 0.1 |

## Battery

On battery the headset caps the CPU at 1651 MHz and holds the GPU at 456 MHz.

| build | default | CPU level 5, GPU level 4 |
|---|---|---|
| stock | 245.3 | 189.7 |
| steps 1 to 2, CPU only | 128.9 | 120.0 |
| steps 1 to 4 | 88.6 | 81.0 |
| steps 1 to 5 (Q8 only) | 78.5 (1.02x) | 72.8 (1.10x) |

The performance levels raise the CPU to 2054 MHz and the GPU to 545 MHz.

## Quality

| build | mel distance to stock, worst of 6 sentences | transcription errors | KL of text logits |
|---|---|---|---|
| steps 1 to 5 (Q8 only) | 3.69 | 0 of 123 words | 0.0015 |
| all six steps | 3.52 | 0 of 123 words | 0.0015 |
| all six steps, 5 held-out sentences | 3.46 | 0 of 98 words | 0.0015 |

Gates: mel at most 4.5, KL at most 0.002, duration within 15%. On a 40 second sentence the Q4 step comes out 15%
shorter than stock with a mel distance of 5.04, while the Q8-only build stays at 4.10. Use the Q8-only patch if you
read long paragraphs.

## What each piece is worth

| removed from the full build | ms per frame | cost of removing it |
|---|---|---|
| nothing | 68.1 | |
| Q4 depthformer weights | 70.0 | +1.9 |
| fused q/k/v and gate/up matrices | 68.6 | +0.5, inside the noise |

## Start-up

| | seconds |
|---|---|
| warm start | 7.5 |
| first start, empty OpenCL cache | 50 to 53 |
| first start with `opencl-lazy-cold-start.patch` | 17 to 23 |

The driver compiles 292 OpenCL programs one after the other and the voice uses 29 of them. Shipping the program
cache with the app (6 MB) makes the first start equal to a warm one on the same driver.

## What did not work

| idea | result |
|---|---|
| pinning threads to cpu2-5 | slower on an awake headset, removing it gave +38% |
| 5 and 6 threads | 81.5 and 98.6 ms per frame |
| 3 threads with the backbone on GPU, before the async codec | slower |
| audio encoders of the mmproj on the GPU | broken audio |
| codec in blocks | +2.7%, inconsistent across sentences |
| GPU gemv reading the weights directly, split-K gemv | no gain, the kernel is bandwidth bound |
| CPU and GPU reading the same weights together | 40.9 GB/s in total, but the sync costs more than it saves |
| Q4 on attention and output heads of the depthformer | 0.35 ms, worse mel |
| Q4 on the backbone | 59.4 ms per frame, KL 60 times over the gate |
| Q6 on the least sensitive backbone layers | about 1 ms inside the gates, not adopted |
