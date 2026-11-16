# LFM3-Omni speaks in real time on a Quest 3S: +111% speed, +212% on battery

![llama.cpp](https://img.shields.io/badge/llama.cpp-9f0e509-blue) ![Meta Quest 3S](https://img.shields.io/badge/Meta%20Quest-3S-black) ![LFM3-Omni-2B](https://img.shields.io/badge/LFM3--Omni-2B%20Q8__0-purple) ![real time](https://img.shields.io/badge/real%20time-1.17x-brightgreen) ![on battery](https://img.shields.io/badge/on%20battery-1.02x-brightgreen) ![OpenCL Adreno 740](https://img.shields.io/badge/OpenCL-Adreno%20740-orange) ![license MIT](https://img.shields.io/badge/license-MIT-lightgrey)

`llama-cpp` `meta-quest` `quest-3s` `text-to-speech` `lfm` `liquid-ai` `on-device-ai` `opencl` `adreno` `snapdragon-xr2` `edge-ai` `vr`

By **Adrian Ciprian Iancu** · [GitHub](https://github.com/ArtyomITA) · [Hugging Face](https://huggingface.co/adriandj3) · [LinkedIn](https://www.linkedin.com/in/adrian-ciprian-iancu-a2a477391)

**A 300 euro VR headset now talks faster than you can listen. No cloud, no PC, no cable.**

Out of the box, [LFM3-Omni-2B](https://huggingface.co/LiquidAI/LFM3-Omni-2B) on a Quest 3S needs 1.8 seconds to say
one second of speech. With one patch to llama.cpp it needs 0.85. Same model file, same headset, same voice.

| | stock llama.cpp | this repo | gain |
|---|---|---|---|
| speed, plugged in | 0.56x real time | **1.17x real time** | **+111%** |
| speed, on battery | 0.33x real time | **1.02x real time** | **+212%** |
| time per frame, plugged in | 143.5 ms | **68.1 ms** | **-53%** |
| time per frame, on battery | 245.3 ms | **78.5 ms** | **-68%** |
| peak CPU memory | 4174 MB | **992 MB** | **-76%** |
| first launch after install | 53 s | **about 20 s** | **-62%** |

One audio frame is 80 ms, so anything under 80 ms per frame is real time. The battery rows use the Q8-only patch.
Listen for yourself: [samples/](samples/) has the same sentences from stock and from this build, in English and Italian.

## Where the time went

Snapdragon XR2 Gen 2, Adreno 740, 6 cores, Horizon OS, model in Q8_0, headset awake, 4 threads.

| step | what changed | ms per frame | real time | gain on stock | audio |
|---|---|---|---|---|---|
| start | llama.cpp `9f0e509`, stock | 143.5 | 0.56x | | reference |
| 1 | MaskGIT schedule 6,6 in the depthformer | 130.0 | 0.62x | +10% | within the quality gates |
| 2 | one shared thread pool, LM head skipped on audio frames, context sized for speech | 108.5 | 0.74x | +32% | bit-identical to step 1 |
| 3 | backbone on the Adreno GPU through OpenCL, with an f32 fix in the Q8 kernel | 86.9 | 0.92x | +65% | within the quality gates |
| 4 | depthformer weights repacked, only the heads that get sampled | 77.4 | 1.03x | +85% | within the quality gates |
| 5 | codec on its own thread while the GPU works, last depthformer layer only on the rows it needs | 70.0 | 1.14x | +105% | bit-identical to step 4 |
| 6 | depthformer feed-forward weights requantized to Q4 at load | **68.1** | **1.17x** | **+111%** | I listened: almost no difference |

Per frame after step 6: backbone 50.7 ms on the GPU, depthformer 17.3 ms on the CPU, codec 0.1 ms on the critical path.

What I learned on the way:

- **The LM head was computed and thrown away on every audio frame.** 128K logits, about 10% of the frame, for nothing.
- **Pinning threads to the fast cores loses on an awake headset.** The VR compositor lives there. Letting the
  scheduler decide gave 38% over my own pinned build.
- **The default context reserves 1.5 GB of KV for a sentence.** Sizing it for speech freed the memory and the swap.
- **The backbone is limited by memory bandwidth, not compute.** The GPU reads 1.5 GB of weights per token at 33 GB/s.
  Smarter kernels gave nothing. Reading fewer bytes is the only lever left.
- **With the codec on its own thread, 3 threads run as fast as 4** (68.6 against 68.1 ms). One core goes back to the app.
- **Ten minutes of continuous speech stay flat** at 68.2 to 68.7 ms per frame while the chip goes from 54 to 73 °C.

## Use it

```bash
git clone https://github.com/ggml-org/llama.cpp && cd llama.cpp
git checkout 9f0e509
git apply ../patches/voice-quest3s-full.patch
```

Build for Android arm64 with the NDK and OpenCL for Adreno:

```bash
cmake -B build -G Ninja -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-29 -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=OFF -DGGML_OPENMP=OFF -DGGML_NATIVE=OFF \
  -DGGML_CPU_ARM_ARCH=armv8.2-a+dotprod+fp16 \
  -DGGML_OPENCL=ON -DGGML_OPENCL_USE_ADRENO_KERNELS=ON -DGGML_OPENCL_EMBED_KERNELS=ON
cmake --build build --target llama-tts
```

On the headset:

```bash
./llama-tts -m LFM3-Omni-2B-Q8_0.gguf -mm mmproj-LFM3-Omni-2B-Q8_0.gguf -t 4 --tts-lang en -f text.txt -o out.wav
```

The patch already sets the GPU offload and the context size, so there are no extra flags.

| patch | what it is |
|---|---|
| `patches/voice-quest3s-full.patch` | all six steps, 68.1 ms per frame |
| `patches/voice-quest3s-q8-only.patch` | steps 1 to 5, weights stay Q8, 70.0 ms per frame |
| `patches/opencl-lazy-cold-start.patch` | optional, on top of either one: the first launch after install drops from 53 s to about 20 s by compiling only the OpenCL programs the voice uses, same audio bit for bit |

Both main patches include the timing counters I used to measure each stage.

## How I measured

- Every run compares two builds on the same headset in A,B,B,A order, one process per block, median milliseconds
  per frame after the first 8 frames, clock read inside the process.
- A run is valid only with the headset awake, on the charger and at full CPU frequency before and after each block.
- Every change is classed before any speed number: bit-identical codes and PCM, or inside fixed gates on mel
  distance, transcription errors, duration and KL divergence of the text logits.
- The final number comes from 5 sentences nobody looked at during the search: 68.4 ms per frame, zero transcription
  errors, within 0.6% of the search result.
- 22 official runs and 14 exploratory ones. The dead ends are in [RESULTS.md](RESULTS.md) with their numbers.

## Citation

```bibtex
@misc{iancu2026lfm3quest,
  author = {Iancu, Adrian Ciprian},
  title  = {LFM3-Omni speaks in real time on a Quest 3S},
  year   = {2026},
  url    = {https://github.com/ArtyomITA/lfm3-omni-voice-quest3s}
}
```

Patches are MIT like llama.cpp. The model belongs to Liquid AI and keeps its own license.
