---
date: "2026-08-11T00:22:00+02:00"
title: "AI on Cards That Came Out of Nowhere"
description: "Hosting generative AI models on weird GPUs"
tags: ["GenAI", "Z440", "NVIDIA", "AI", "GPU", "Inference", "LLM", "llama.cpp"]
---

---

## What's the Point?

Ever since I got into AI, I've enjoyed messing around with weird GPUs. After putting together [An AI Setup Cheaper Than Mario Kart World?](/en/posts/cheap-ai-server/), I wondered whether decent AI models could run on genuinely cheap GPUs. Spoiler: yes, they can. With a few dark spots along the way.

## The Setup

The setup is more or less the same as in the previous article. As a reminder:

- HP 700W PSU  
- [Intel Xeon E5-1620 v3 @ 3.50GHz](https://www.intel.fr/content/www/fr/fr/products/sku/82763/intel-xeon-processor-e51620-v3-10m-cache-3-50-ghz/specifications.html)
- 32GB RAM DDR4 ECC 2133MHz
- [Samsung EVO 250GB SSD](https://www.samsung.com/fr/memory-storage/sata-ssd/870-evo-250gb-sata-3-2-5-ssd-mz-77e250b-eu/)
- [Quadro K2200 4GB](https://www.nvidia.com/content/dam/en-zz/Solutions/design-visualization/documents/75509_DS_NV_Quadro_K2200_US_NV_HR.pdf)
- [GTX 1050 Ti 4GB](https://www.gigabyte.com/fr/Graphics-Card/GV-N105TG1-GAMING-4GD)

But we're adding 2 slightly suspicious cards to the mix:

- [NVIDIA Tesla P4](https://www.techpowerup.com/gpu-specs/tesla-p4.c2879): €90
- [NVIDIA Tesla T10](https://www.techpowerup.com/gpu-specs/tesla-t10-16-gb.c4036): €300

The [NVIDIA Tesla P4](https://www.techpowerup.com/gpu-specs/tesla-p4.c2879) is essentially an [NVIDIA GeForce GTX 1080](https://www.techpowerup.com/gpu-specs/geforce-gtx-1080.c2839) in a reduced form factor (HHHL), passively cooled, and drawing only 75W — no external power connector needed!

It dates back to 2016 and packs **8GB of GDDR5** with a **192.3 GB/s** memory bus and **5.704 TFLOPS** in FP32. According to my sources (myself, and my multiple personalities), this GPU was mainly used for VDI (Virtual Desktop Infrastructure) and transcoding.

![Techpowerup Tesla P4](/img/tpu-nvidia-p4.png)

The [NVIDIA Tesla T10](https://www.techpowerup.com/gpu-specs/tesla-t10-16-gb.c4036) is a bit more special. While comparable to an [NVIDIA GeForce RTX 2080 Ti](https://www.techpowerup.com/gpu-specs/geforce-rtx-2080-ti.c3305), it has fewer **cores**, fewer **TMUs**, fewer **ROPs**, and a smaller memory bus. It's a single-slot PCIe form factor (FHFL), passively cooled, and draws 150W through an external 8-pin PCIe connector.

It dates back to 2020 and packs **16GB of GDDR6** with a **403.2 GB/s** memory bus and **9.999 TFLOPS** in FP32. After several hours of digging, I found out this card was used by *Nvidia* for *GeForce Now*. So it used to serve cloud gaming!

> Nvidia cards on the Turing architecture (RTX 20xx) were the first to feature *Tensor Cores*, dedicated cores that accelerate AI-specific computations. That's largely what made [DLSS](https://www.nvidia.com/fr-fr/geforce/technologies/dlss/) possible.

![Techpowerup Tesla T10](/img/tpu-nvidia-t10.png)

### Why These Cards Are Weird

Nvidia's *GTX*, *RTX*, or *Quadro* cards are fairly well known to the public. They're used for gaming, video rendering, CAD, and other everyday tasks. *Tesla* cards, on the other hand, are dedicated to much more specific jobs — mainly virtualization through solutions like [vGPU](https://docs.nvidia.com/vgpu/index.html) or [MIG](https://www.nvidia.com/en-us/technologies/multi-instance-gpu/), letting a single card serve several virtual machines at once (which explains why the *Tesla T10* ended up powering *GeForce Now*). They're also often used for *Machine Learning* workloads. They're built on the same silicon as consumer cards, but are meant to live in servers — hence the passive cooling.

### Cooling

Now that we know these cards a bit better, they need cooling. The *HP Z440* workstation was never designed to host this kind of card. So we'll have to improvise, because a card with no cooling climbing to **97°C** is genuinely a bad idea. Time to go active.

And active cooling means fans!

> ⚠️ The modifications made here aren't recommended and are risky. Messing with electricity, even at 12V, can cost you your life, so don't be an idiot about it. I'm not encouraging anyone to reproduce the following modifications, and I decline all responsibility for lost warranties, broken hardware, or physical injury.

To power our fans we need a source of electricity, and what better than an unused *SATA* power connector? It can supply **3.3V at 1.5A**, **5V at 4.5A**, **12V at 4.5A** — perfect for powering our fans.

> For more information: https://tadeubento.com/2025/sata-connector-power-rating-and-hard-drives/

So I hacked together a *SATA* -> 3-pin fan adapter wired to the **12V** rail (okay, only 2 pins on the fan connector actually get used, but the 3rd one is just for PWM / speed control, which we don't need here).

![SATA TO 3 PIN](/img/cooling-sata-adapter.png)

Next up, cooling our *Tesla P4*. I 3D-printed a shroud to hold a **30mm** fan, powered at **12V/0.1A**, blowing air straight onto the GPU's copper block. Not the best way to cool this card, but I couldn't be bothered designing anything fancier and didn't have another fan small enough.

![Tesla P4 Cooling](/img/cooling-p4.png)

Now for the *Tesla T10*, time to bring out the heavy artillery. I printed another 3D shroud for a **30mm** fan, powered at **12V/0.40A**, but this time in *blower* mode — respecting the card's actual airflow direction.

![Tesla T10 Cooling](/img/cooling-t10.png)

I then used a fan splitter to distribute the **12V** across both cards. Adding the *Tesla P4*'s **0.1A** to the *Tesla T10*'s **0.40A** gets us to **0.50A**, well within the **12V at 4.5A** the *SATA* power connector can supply, for a total of 6W of cooling across both cards. Doesn't sound like much, yet it's plenty for this setup.

### The Software

Hardware-wise, we're all set! 4 GPUs, all cooled, ready for an OS.

As usual, when it comes to solid GPU driver support, [Ubuntu](https://ubuntu.com) is the way to go.

Installing the drivers couldn't be simpler:

```bash
ubuntu-drivers list --gpgpu
```

This command automatically figures out the most suitable Nvidia driver version for our cards:

```
nvidia-driver-580-server, (kernel modules provided by linux-modules-nvidia-580-server-generic)
nvidia-driver-580, (kernel modules provided by linux-modules-nvidia-580-generic)
```

All that's left is installing them:

```bash
apt-get update
apt-get install -y nvidia-driver-580-server
reboot
```

Let's check the OS actually sees our cards:

```bash
nvidia-smi
```

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.173.02             Driver Version: 580.173.02     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Tesla T10                      Off |   00000000:02:00.0 Off |                    0 |
| N/A   46C    P0             40W /  150W |    5121MiB /  15360MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   1  Tesla P4                       Off |   00000000:04:00.0 Off |                    0 |
| N/A   40C    P0             23W /   75W |       0MiB /   7680MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   2  Quadro K2200                   Off |   00000000:06:00.0 Off |                  N/A |
| 34%   47C    P0              2W /   39W |       0MiB /   4096MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA GeForce GTX 1050 Ti     Off |   00000000:09:00.0 Off |                  N/A |
|  0%   44C    P0            N/A  /  120W |       0MiB /   4096MiB |      2%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
```

And we're good — our cards are ready to use!

Now let's install [nvtop](https://github.com/Syllo/nvtop), an *htop* for GPUs.

```bash
apt-get install -y nvtop
```

## Hosting a Model

Okay, I lied. Our cards aren't really ready to use yet. The *Maxwell* (Quadro K2200), *Pascal* (Tesla P4 & 1050 Ti), and *Turing* (Tesla T10) architectures aren't really supported by much software anymore — the newest card here is 6 years old!

So we're going to have to rebuild some of it ourselves, starting with [llama.cpp](https://llama.app/). It's a very capable inference engine that lets us host generative AI models on pretty much any CPU or GPU, and does so fairly simply. It's what [Ollama](https://ollama.com) uses under the hood to make your life easier.

### Building llama.cpp for Our Cards

Building software by hand can often be intimidating — not here. The build process has been massively simplified and can be done in under 5 commands:

1. Install the prerequisites:

Before compiling *llama.cpp* we need a couple of packages

```bash
apt-get install -y curl cmake git nvidia-cuda-toolkit
```

2. Clone the repository:

```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp
```

3. Build:

```bash
cmake -B build -DGGML_CUDA=ON -DLLAMA_CURL=ON -DCMAKE_CUDA_ARCHITECTURES=<CUDA SM VERSION> # 50 for Maxwell | 61 for Pascal | 75 for Turing
cmake --build build -j$(nproc) # Uses all available cores to compile llama.cpp
```

After ~10 minutes, all the binaries needed to host AI models show up in `~/llama.cpp/build/bin`

### Downloading a Model

Since using *llama.cpp* without a model would be pretty pointless, let's download one.

Head over to [HuggingFace](https://hf.co) and you'll find several MILLION models. So how do you even pick one?

*Llama.cpp* only supports one specific file format, `.gguf`. So that narrows things down to a usable filter right away. Beyond that, we need a model that actually fits our setup. And that alone deserves its own article, because it sounds simple but it's an absolute mess, genuinely.

Here's the model I picked:

- [unsloth/gemma-4-E4B-it-qat-GGUF](https://huggingface.co/unsloth/gemma-4-E4B-it-qat-GGUF)

Small enough to comfortably fit in the vRAM of both cards even without optimization, beefy enough not to spout complete nonsense. In short, the perfect size for what we're doing here: trying to get a model to say something dumb.

Our model comes split across several files:

- `weird_letters_model_name.gguf`: The model itself
- `mmproj-FP16.gguf`: Its multi-modality functions (image / audio understanding)

## Benchmarking

### Methodology

Let's install a great tool that'll let us run benchmarks: [llama-benchy](https://github.com/eugr/llama-benchy)

```bash
snap install astral-uv
uvx llama-benchy
```

```
usage: llama-benchy [-h] [--version] --base-url BASE_URL [--api-key API_KEY] [--model MODEL] [--served-model-name SERVED_MODEL_NAME] [--tokenizer TOKENIZER] [--pp PP [PP ...]]
                    [--tg TG [TG ...]] [--exact-tg] [--depth DEPTH [DEPTH ...]] [--runs RUNS] [--no-cache] [--post-run-cmd POST_RUN_CMD] [--book-url BOOK_URL]
                    [--latency-mode {api,generation,none}] [--no-warmup] [--skip-coherence] [--adapt-prompt] [--no-adapt-prompt] [--enable-prefix-caching]
                    [--concurrency CONCURRENCY [CONCURRENCY ...]] [--save-result SAVE_RESULT] [--format {md,json,csv}] [--save-total-throughput-timeseries]
                    [--save-all-throughput-timeseries] [--exit-on-first-fail] [--no-results-on-fail] [--extra-body EXTRA_BODY] [--emit-progress PATH]
llama-benchy: error: the following arguments are required: --base-url
```

### Tesla P4

#### Command

*Llama.cpp* ships several binaries for running models; the one we care about here is `llama-server`. It hosts a model under the [OpenAI](https://github.com/openai/openai-openapi) API standard. We use it because it gives us both an API and a simple web UI.

Alright, we've been talking about it long enough — let's actually launch a model:

```bash
CUDA_VISIBLE_DEVICES=1 ~/pascal/llama.cpp/build/bin/llama-server --host 0.0.0.0 --port 8080 -m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf
```

> Here the model runs on the *Tesla P4*

Alright, let's actually break down this mess, because right now it looks like someone vomited on my keyboard.

- `CUDA_VISIBLE_DEVICES=1`: "Selects" the right GPU (mixing multiple architectures and multiple *llama.cpp* builds can go sideways fast, so we make sure to pick the right card)
- `~/pascal/llama.cpp/build/bin/llama-server`: Location of our `llama-server` binary on the machine
- `--host 0.0.0.0 --port 8080`: Do I really need to explain this one?
- `-m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf`: Location of our model
- `--mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf`: Location of its multi-modality functions

Already a bit clearer!

Now let's head to `http://our-z440-ip:8080/` and chat with our model:

![llama.cpp chat P4](/img/llama.cpp-chat-p4.png)

Okay, that's cool, but is it optimized? As a reminder, the whole point here is squeezing decent performance out of these cards. Let's move to the real benchmark.

#### Bench

Perfect! Now let's benchmark our current setup:

```bash
uvx llama-benchy --base-url http://ip-de-notre-z440:8080/v1 --served-model-name /root/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --model google/gemma4-E4B-it --pp 512 2048 4096 --tg 512 2048 4096
```

- `--base-url`: The *llama.cpp* URL
- `--served-model-name`: The "name" of the model served by *llama.cpp*
- `--model`: The model "name" to look up on [HuggingFace](https://hf.co) for the tests
- `--pp`: The different prompt sizes (in tokens)
- `--tg`: The different response sizes (in tokens)

`llama-benchy` doesn't just pair up each `--pp` value with the `--tg` value at the same rank. It actually tests **every possible combination** between our prompt sizes and response sizes: `pp512/tg512`, `pp512/tg2048`, `pp512/tg4096`, `pp2048/tg512`, and so on up to `pp4096/tg4096`. With 3 values on each side, that's 9 test combinations. That's intentional: a short prompt followed by a long response (summarization, code generation...) and a long prompt followed by a short response (RAG, classification...) stress the GPU in completely different ways, so we might as well measure all of them.

And for each of these 9 combinations, the test is repeated 3 times (the default value of the `--runs` parameter), on top of an initial dry run (the *warmup*) that gets thrown away and doesn't count toward the results. The goal is to avoid a single fluke result (GPU clock ramp-up not finished, some background process hogging resources, etc.) skewing the measurement — so `llama-benchy` gives us a mean ± standard deviation computed over these 3 runs, rather than one isolated number we can't really trust.

While the test runs, let's take a look at what our `nvtop` looks like:

![nvtop p4](/img/nvtop-p4.png)

After ~20 minutes, our benchmark is done:

```
| model                |   test |            t/s |     peak t/s |        ttfr (ms) |     est_ppt (ms) |    e2e_ttft (ms) |
|:---------------------|-------:|---------------:|-------------:|-----------------:|-----------------:|-----------------:|
| google/gemma4-E4B-it |  pp512 | 626.26 ± 16.91 |              |   757.76 ± 33.61 |   755.08 ± 33.61 |   842.09 ± 33.99 |
| google/gemma4-E4B-it |  tg512 |   36.12 ± 0.23 | 36.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it |  pp512 | 605.81 ± 19.03 |              |   716.45 ± 13.87 |   713.77 ± 13.87 |   802.20 ± 13.58 |
| google/gemma4-E4B-it | tg2048 |   35.69 ± 0.06 | 36.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it |  pp512 |  585.52 ± 8.36 |              |   814.39 ± 11.67 |   811.70 ± 11.67 |   900.43 ± 10.46 |
| google/gemma4-E4B-it | tg4096 |   35.92 ± 0.22 | 36.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  598.03 ± 8.90 |              | 3118.10 ± 153.00 | 3115.42 ± 153.00 | 3208.05 ± 152.01 |
| google/gemma4-E4B-it |  tg512 |   34.46 ± 0.05 | 35.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  589.45 ± 8.52 |              |  3153.12 ± 37.64 |  3150.44 ± 37.64 |  3242.11 ± 37.53 |
| google/gemma4-E4B-it | tg2048 |   34.21 ± 0.07 | 35.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  584.55 ± 5.87 |              |  3314.65 ± 59.98 |  3311.97 ± 59.98 |  3403.88 ± 60.48 |
| google/gemma4-E4B-it | tg4096 |   34.23 ± 0.04 | 35.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  539.74 ± 1.34 |              |  6937.52 ± 64.75 |  6934.84 ± 64.75 |  7028.14 ± 64.68 |
| google/gemma4-E4B-it |  tg512 |   33.93 ± 0.04 | 34.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  537.94 ± 1.73 |              | 6871.97 ± 104.42 | 6869.28 ± 104.42 | 6963.09 ± 103.85 |
| google/gemma4-E4B-it | tg2048 |   33.79 ± 0.26 | 34.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  536.15 ± 2.54 |              | 6829.99 ± 249.02 | 6827.31 ± 249.02 | 6920.72 ± 248.83 |
| google/gemma4-E4B-it | tg4096 |   33.79 ± 0.09 | 34.00 ± 0.00 |                  |
```

What we take away:

- **~35 t/s AVG** in Text Generation
- **~537 t/s AVG** in Prompt Processing
- **~3701.19 ms AVG** in Time To First Token
- **~6.1 Gi** of vRAM used

For a model this size on a sub-€100 card, that's honestly not bad! But let's try to optimize it further anyway.

#### Optimization

`llama-server` offers a whole bunch of options for squeezing out more performance without changing a single piece of hardware. The idea behind all of it boils down to two axes:

- **Consume less vRAM at rest.** The model itself takes up a fixed size, but the KV-cache grows with context — every token of conversation adds its share. The less we consume at equal context size, the more headroom we have to... increase the max context size.
- **Go faster at equal context**, without sacrificing anything on response quality.

**Flash Attention** — `-fa on` enables an optimized attention kernel that computes attention scores without ever materializing the full matrix in memory. Result: less vRAM used by the attention computation itself, and generally a bit more speed, especially as context grows. It's also a technical prerequisite for what comes next: llama.cpp requires Flash Attention to be enabled in order to quantize the KV-cache.

**KV-cache quantization** — `-ctk q4_0 -ctv q4_0` quantizes the *Keys* and *Values* of the KV-cache from FP16 down to INT4. Since the KV-cache is precisely the part that grows with context, quantizing it has a direct and immediate effect on available vRAM: at equal context, we consume much less, which in turn lets us raise the max context size without blowing up vRAM.

**A larger context window** — With all the vRAM reclaimed from the two points above, `-c 65535` lets us raise the max context size the server supports. Without these optimizations, a value like this would simply crash `llama-server` for lack of vRAM.

**Batching** — There's also `-b` and `-ub`, which control the batch sizes used to process prompts (`-ub` in particular splits prompt processing into smaller chunks). Bumping them up generally saturates the GPU better during the prompt processing phase, at the cost of a bit more vRAM — tune to whatever headroom is left once the optimizations above are in place.

So much for theory, let's get practical:

```bash
CUDA_VISIBLE_DEVICES=1 ~/pascal/llama.cpp/build/bin/llama-server --host 0.0.0.0 --port 8080 -m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf -fa on -ctk q4_0 -ctv q4_0 -c 65535
```

#### Bench

And we run the exact same benchmark as before, on the same card, for an apples-to-apples comparison:

```bash
uvx llama-benchy --base-url http://ip-de-notre-z440:8080/v1 --served-model-name /root/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --model google/gemma4-E4B-it --pp 512 2048 4096 --tg 512 2048 4096
```

```
| model                |   test |            t/s |     peak t/s |        ttfr(ms) |     est_ppt (ms) |    e2e_ttft (ms) |
|:---------------------|-------:|---------------:|-------------:|-----------------:|-----------------:|-----------------:|
| google/gemma4-E4B-it |  pp512 |  632.73 ± 2.66 |              |   740.38 ± 34.31 |   737.04 ± 34.31 |   836.29 ± 34.28 |
| google/gemma4-E4B-it |  tg512 |   31.88 ± 0.10 | 32.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it |  pp512 | 613.01 ± 14.06 |              |    785.73 ± 5.08 |    782.39 ± 5.08 |    883.85 ± 5.24 |
| google/gemma4-E4B-it | tg2048 |   30.77 ± 0.22 | 31.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it |  pp512 | 592.09 ± 20.13 |              |   774.32 ± 44.09 |   770.98 ± 44.09 |   872.55 ± 43.08 |
| google/gemma4-E4B-it | tg4096 |   31.33 ± 0.40 | 31.67 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  593.44 ± 3.50 |              |  3100.50 ± 11.47 |  3097.16 ± 11.47 |  3206.30 ± 10.84 |
| google/gemma4-E4B-it |  tg512 |   29.07 ± 0.02 | 30.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  594.06 ± 3.14 |              | 3079.92 ± 152.22 | 3076.58 ± 152.22 | 3185.74 ± 153.86 |
| google/gemma4-E4B-it | tg2048 |   29.05 ± 0.17 | 29.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  586.12 ± 2.48 |              |  3169.53 ± 42.82 |  3166.19 ± 42.82 |  3275.80 ± 41.64 |
| google/gemma4-E4B-it | tg4096 |   29.12 ± 0.16 | 29.67 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  539.86 ± 3.08 |              |  6959.97 ± 90.95 |  6956.63 ± 90.95 |  7067.44 ± 91.00 |
| google/gemma4-E4B-it |  tg512 |   28.37 ± 0.02 | 29.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  540.97 ± 1.74 |              | 6869.72 ± 181.18 | 6866.38 ± 181.18 | 6976.98 ± 182.31 |
| google/gemma4-E4B-it | tg2048 |   28.04 ± 0.03 | 29.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  540.54 ± 2.77 |              | 6954.39 ± 109.84 | 6951.05 ± 109.84 | 7062.22 ± 109.84 |
| google/gemma4-E4B-it | tg4096 |   28.02 ± 0.14 | 28.33 ± 0.47 |                  |                  |                  |
```

What we take away, comparing to the P4 baseline:

- **~29.52 t/s AVG** in Text Generation, vs ~35 t/s baseline — so **~16% slower**
- **~581.42 t/s AVG** in Prompt Processing, vs ~537 t/s baseline — so **~8% faster**
- **~3707.46 ms AVG** in Time To First Token, nearly identical to baseline
- **~4.5 Gi** of vRAM used, vs ~6.1 Gi baseline — so **~26% less vRAM**, with a context 16x larger

Same story as on the *T10*, and this time with zero ambiguity since we've taken MTP out of the equation: quantizing the KV-cache frees up vRAM and allows for a much larger context, but it clearly costs generation speed.

#### And With MTP?

Quick detour before diving into the numbers: what is MTP anyway? *Multi-Token Prediction* is an extra head grafted onto the model, letting it propose several tokens at once per pass instead of just one. The server then verifies these proposals in a single pass, and only keeps the ones that turn out correct. Unlike classic *speculative decoding*, which relies on a complete, separate "draft" model to propose these tokens, MTP uses a head built into the main model itself — no second model to load alongside it in memory.

Same question as for the *T10*: does adding MTP claw back some of the lost speed?

```bash
CUDA_VISIBLE_DEVICES=1 ~/pascal/llama.cpp/build/bin/llama-server --host 0.0.0.0 --port 8080 -m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf --model-draft ~/models/gemma-4-E4B-it-qat/mtp-gemma-4-E4B-it.gguf -fa on -ctk q4_0 -ctv q4_0 -c 65535
```

```
| model                |   test |            t/s |     peak t/s |        ttfr (ms) |     est_ppt (ms) |    e2e_ttft (ms) |
|:---------------------|-------:|---------------:|-------------:|-----------------:|-----------------:|-----------------:|
| google/gemma4-E4B-it |  pp512 |  632.95 ± 9.44 |              |   711.71 ± 40.98 |   707.82 ± 40.98 |   807.84 ± 41.80 |
| google/gemma4-E4B-it |  tg512 |   32.09 ± 0.06 | 33.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it |  pp512 | 621.82 ± 21.10 |              |   734.46 ± 32.46 |   730.56 ± 32.46 |   830.64 ± 33.05 |
| google/gemma4-E4B-it | tg2048 |   31.61 ± 0.51 | 32.00 ± 0.82 |                  |                  |                  |
| google/gemma4-E4B-it |  pp512 | 615.18 ± 13.59 |              |    728.85 ± 2.69 |    724.95 ± 2.69 |    824.72 ± 2.91 |
| google/gemma4-E4B-it | tg4096 |   31.54 ± 0.45 | 31.67 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  598.03 ± 8.90 |              | 3118.10 ± 153.00 | 3115.42 ± 153.00 | 3208.05 ± 152.01 |
| google/gemma4-E4B-it |  tg512 |   29.34 ± 0.07 | 30.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  589.45 ± 8.52 |              |  3153.12 ± 37.64 |  3150.44 ± 37.64 |  3242.11 ± 37.53 |
| google/gemma4-E4B-it | tg2048 |   29.02 ± 0.05 | 29.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  584.55 ± 5.87 |              |  3314.65 ± 59.98 |  3311.97 ± 59.98 |  3403.88 ± 60.48 |
| google/gemma4-E4B-it | tg4096 |   28.97 ± 0.03 | 29.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  539.74 ± 1.34 |              |  6937.52 ± 64.75 |  6934.84 ± 64.75 |  7028.14 ± 64.68 |
| google/gemma4-E4B-it |  tg512 |   28.45 ± 0.06 | 29.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  537.94 ± 1.73 |              | 6871.97 ± 104.42 | 6869.28 ± 104.42 | 6963.09 ± 103.85 |
| google/gemma4-E4B-it | tg2048 |   28.15 ± 0.06 | 29.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  536.15 ± 2.54 |              | 6829.99 ± 249.02 | 6827.31 ± 249.02 | 6920.72 ± 248.83 |
| google/gemma4-E4B-it | tg4096 |   28.13 ± 0.10 | 29.00 ± 0.00 |                  |                  |                  |
```

With MTP: **~29.70 t/s** in generation, vs **~29.52 t/s** without. A difference of **0.6%** — even smaller than on the *T10*, and clearly within measurement noise. Same verdict: on *Pascal*, MTP barely recovers anything.

And on the vRAM side, same story: still **~4.5 Gi**, with or without MTP, for exactly the same reason as on the *T10* — the MTP head is grafted onto the main model, not a separate draft model loaded on the side, so its footprint is too small to show up at the scale we're measuring here.

On the *P4* at least, the verdict is clear: MTP costs almost nothing in vRAM, but doesn't gain almost anything in speed either, once the KV-cache is quantized. Remains to be seen whether the *T10* tells the same story.

### Tesla T10

On to the pricier card: €300, more than 3 times the *Tesla P4*'s price. But on paper, everything is better: **16GB** of vRAM instead of 8, a memory bus at **403.2 GB/s** instead of 192.3, more cores, and above all, *Tensor Cores* — the *Turing* architecture has them, the P4's *Pascal* architecture doesn't.

That last point is exactly what interests me. On the P4, we saw that quantizing the KV-cache to INT4 slowed down generation, likely for lack of hardware acceleration for this kind of computation. The T10, on the other hand, has exactly the kind of dedicated silicon that could sidestep the issue. Let's see if the theory holds.

#### Command

Same model, same method, we're just swapping cards:

```bash
CUDA_VISIBLE_DEVICES=0 ~/turing/llama.cpp/build/bin/llama-server --host 0.0.0.0 --port 8080 -m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf
```

The only changes from the P4 launch: `CUDA_VISIBLE_DEVICES=0` to target the *Tesla T10* this time, and the `llama.cpp` binary compiled for *Turing* (`~/turing/...` instead of `~/pascal/...`, since these are two different CUDA architectures). The rest of the flags were already explained above.

#### Bench

While the test runs, let's take a look at what our `nvtop` looks like:

![nvtop t10](/img/nvtop-t10.png)

After ~20 minutes, our benchmark is done:

```
| model                |   test |             t/s |     peak t/s |       ttfr (ms) |    est_ppt (ms) |   e2e_ttft (ms) |
|:---------------------|-------:|----------------:|-------------:|----------------:|----------------:|----------------:|
| google/gemma4-E4B-it |  pp512 | 1865.19 ± 15.40 |              |   254.97 ± 8.77 |   251.48 ± 8.77 |   298.96 ± 8.77 |
| google/gemma4-E4B-it |  tg512 |    75.49 ± 0.04 | 76.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it |  pp512 | 1838.42 ± 27.07 |              |   258.27 ± 5.63 |   254.77 ± 5.63 |   302.53 ± 5.61 |
| google/gemma4-E4B-it | tg2048 |    75.19 ± 0.46 | 75.33 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it |  pp512 | 1865.61 ± 13.04 |              |  257.17 ± 10.23 |  253.68 ± 10.23 |  301.61 ± 10.15 |
| google/gemma4-E4B-it | tg4096 |    74.67 ± 0.54 | 75.00 ± 0.82 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 | 2202.63 ± 14.41 |              |  830.65 ± 20.84 |  827.16 ± 20.84 |  877.01 ± 20.99 |
| google/gemma4-E4B-it |  tg512 |    72.25 ± 0.06 | 73.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 |  2178.48 ± 4.94 |              |  857.42 ± 12.54 |  853.93 ± 12.54 |  903.92 ± 12.52 |
| google/gemma4-E4B-it | tg2048 |    71.74 ± 0.22 | 72.33 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 |  2147.50 ± 5.75 |              |  856.32 ± 25.08 |  852.83 ± 25.08 |  902.75 ± 25.26 |
| google/gemma4-E4B-it | tg4096 |    71.58 ± 0.02 | 72.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 |  2207.94 ± 4.77 |              | 1689.38 ± 11.35 | 1685.89 ± 11.35 | 1737.32 ± 11.35 |
| google/gemma4-E4B-it |  tg512 |    70.53 ± 0.02 | 71.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 |  2176.68 ± 4.39 |              | 1707.47 ± 16.01 | 1703.98 ± 16.01 | 1755.38 ± 16.03 |
| google/gemma4-E4B-it | tg2048 |    69.78 ± 0.11 | 70.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 |  2165.45 ± 7.28 |              |  1710.31 ± 6.47 |  1706.82 ± 6.47 |  1758.25 ± 6.64 |
| google/gemma4-E4B-it | tg4096 |    69.74 ± 0.07 | 70.00 ± 0.00 |                 |                 |                 |
```

What we take away:

- **~72.33 t/s AVG** in Text Generation
- **~2072.99 t/s AVG** in Prompt Processing
- **~982.00 ms AVG** in Time To First Token
- **6.5 Gi** of vRAM used

And here it's a completely different ballgame. Against the P4 baseline (~35 t/s TG, ~537 t/s PP, ~3701 ms TTFT), the T10 more than doubles generation speed, nearly 4x's prompt processing, and comes in nearly 4x faster on time to first token. For 3.3x the price, that's well above the price delta — consistent with the specs on paper (2.1x higher memory bandwidth, more cores, and a newer architecture). vRAM, meanwhile, is nearly identical (6.5 vs 6.1 Gi), confirming that the small gap really does come down to driver/architecture overhead and nothing else.

That leaves the real question: do the T10's Tensor Cores let it get both less vRAM usage *and* more speed once the optimizations are applied, where the P4 had to pick one or the other?

#### Optimization

Same optimizations as on the P4 (Flash Attention, KV-cache in q4_0, context at 65535), only the card changes:

```bash
CUDA_VISIBLE_DEVICES=0 ~/turing/llama.cpp/build/bin/llama-server --host 0.0.0.0 --port 8080 -m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf -fa on -ctk q4_0 -ctv q4_0 -c 65535
```

#### Bench

```bash
uvx llama-benchy --base-url http://ip-de-notre-z440:8080/v1 --served-model-name /root/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --model google/gemma4-E4B-it --pp 512 2048 4096 --tg 512 2048 4096
```

```
| model                |   test |             t/s |     peak t/s |       ttfr (ms) |    est_ppt (ms) |   e2e_ttft (ms) |
|:---------------------|-------:|----------------:|-------------:|----------------:|----------------:|----------------:|
| google/gemma4-E4B-it |  pp512 | 1988.38 ± 36.14 |              |   237.59 ± 7.58 |   236.00 ± 7.58 |   287.23 ± 7.72 |
| google/gemma4-E4B-it |  tg512 |    67.42 ± 0.10 | 68.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it |  pp512 | 1951.59 ± 29.81 |              |   243.16 ± 7.08 |   241.57 ± 7.08 |   293.13 ± 7.21 |
| google/gemma4-E4B-it | tg2048 |    66.54 ± 0.47 | 67.33 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it |  pp512 | 1919.20 ± 30.58 |              |   250.24 ± 4.15 |   248.65 ± 4.15 |   300.43 ± 4.32 |
| google/gemma4-E4B-it | tg4096 |    67.30 ± 0.17 | 68.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 | 2250.24 ± 12.67 |              |  833.73 ± 17.48 |  832.14 ± 17.48 |  885.65 ± 17.62 |
| google/gemma4-E4B-it |  tg512 |    64.83 ± 0.08 | 65.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 | 2221.61 ± 25.12 |              |  829.26 ± 30.35 |  827.67 ± 30.35 |  881.16 ± 30.47 |
| google/gemma4-E4B-it | tg2048 |    64.16 ± 0.53 | 64.67 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 |  2171.53 ± 8.33 |              |   854.46 ± 3.10 |   852.87 ± 3.10 |   906.73 ± 3.10 |
| google/gemma4-E4B-it | tg4096 |    63.93 ± 0.50 | 64.33 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 | 2186.28 ± 13.02 |              | 1694.95 ± 60.25 | 1693.36 ± 60.25 | 1749.37 ± 59.86 |
| google/gemma4-E4B-it |  tg512 |    61.58 ± 0.42 | 61.67 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 |  2170.55 ± 5.64 |              | 1710.09 ± 12.77 | 1708.50 ± 12.77 | 1764.26 ± 13.04 |
| google/gemma4-E4B-it | tg2048 |    60.56 ± 0.21 | 61.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 | 2173.51 ± 27.94 |              |  1734.22 ± 7.05 |  1732.63 ± 7.05 |  1788.50 ± 6.83 |
| google/gemma4-E4B-it | tg4096 |    60.25 ± 0.14 | 61.00 ± 0.00 |                 |                 |                 |
```

What we take away, comparing to the T10 baseline:

- **~64.06 t/s AVG** in Text Generation, vs ~72.33 t/s baseline — so **~11% slower**
- **~2114.77 t/s AVG** in Prompt Processing, vs ~2072.99 t/s baseline — so **~2% faster**
- **~984.05 ms AVG** in Time To First Token, nearly identical to baseline
- **4.8 Gi** of vRAM used, vs 6.5 Gi baseline — so **~26% less vRAM** (same ratio as on the P4, though in absolute terms the T10 still uses more than the P4, both at baseline and optimized)

Same verdict as on the P4, no surprise this time since we removed the one variable that could muddy the signal: quantizing the KV-cache does reduce the memory footprint — leaving room for a larger context — but it costs generation speed. No free lunch.

Which answers the question from earlier: Tensor Cores don't dodge the problem seen on the P4, they only soften it. **-11%** on the T10 vs **-16%** on the P4: less bad, but still not free. Consistent with what we already saw with vRAM being identical with or without MTP — raw compute power isn't the bottleneck for dequantizing the KV-cache at every token, so more Tensor Cores only help at the margins.

#### And With MTP?

One question remains open: can we get closer to the original speeds by layering MTP on top, without giving up the vRAM gains? Same command, we just add `--model-draft`:

```bash
CUDA_VISIBLE_DEVICES=0 ~/turing/llama.cpp/build/bin/llama-server --host 0.0.0.0 --port 8080 -m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf --model-draft ~/models/gemma-4-E4B-it-qat/mtp-gemma-4-E4B-it.gguf -fa on -ctk q4_0 -ctv q4_0 -c 65535
```

```
| model                |   test |             t/s |     peak t/s |       ttfr (ms) |    est_ppt (ms) |   e2e_ttft (ms) |
|:---------------------|-------:|----------------:|-------------:|----------------:|----------------:|----------------:|
| google/gemma4-E4B-it |  pp512 | 2069.27 ± 71.73 |              |  224.31 ± 11.39 |  221.16 ± 11.39 |  274.02 ± 11.57 |
| google/gemma4-E4B-it |  tg512 |    68.16 ± 0.16 | 68.67 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it |  pp512 | 1971.28 ± 26.19 |              |  241.20 ± 10.83 |  238.06 ± 10.83 |  290.88 ± 10.77 |
| google/gemma4-E4B-it | tg2048 |    67.72 ± 0.16 | 68.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it |  pp512 | 1962.64 ± 11.68 |              |   240.62 ± 7.49 |   237.48 ± 7.49 |   290.59 ± 7.34 |
| google/gemma4-E4B-it | tg4096 |    67.41 ± 0.59 | 68.00 ± 0.82 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 |  2238.91 ± 9.44 |              |   824.99 ± 5.05 |   821.85 ± 5.05 |   876.51 ± 5.05 |
| google/gemma4-E4B-it |  tg512 |    65.55 ± 0.01 | 66.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 |  2221.51 ± 7.36 |              |  859.13 ± 14.64 |  855.98 ± 14.64 |  911.05 ± 14.70 |
| google/gemma4-E4B-it | tg2048 |    64.83 ± 0.05 | 65.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 | 2188.13 ± 14.61 |              |  837.90 ± 33.00 |  834.75 ± 33.00 |  889.73 ± 33.18 |
| google/gemma4-E4B-it | tg4096 |    64.65 ± 0.21 | 65.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 |  2196.95 ± 2.31 |              | 1666.21 ± 20.25 | 1663.07 ± 20.25 | 1720.12 ± 20.14 |
| google/gemma4-E4B-it |  tg512 |    62.70 ± 0.04 | 63.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 |  2201.92 ± 4.87 |              | 1670.53 ± 39.07 | 1667.38 ± 39.07 | 1724.42 ± 39.29 |
| google/gemma4-E4B-it | tg2048 |    61.86 ± 0.15 | 62.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 | 2183.82 ± 10.99 |              | 1725.97 ± 14.33 | 1722.83 ± 14.33 | 1780.36 ± 14.30 |
| google/gemma4-E4B-it | tg4096 |    61.04 ± 0.01 | 62.00 ± 0.00 |                 |                 |                 |
```

With MTP: **~64.88 t/s** in generation, vs **~64.06 t/s** without. A difference of **1.3%** — within the margin of error of the standard deviations shown in the table. In other words, no: MTP barely compensates for the speed loss caused by KV-cache quantization, at least not at this scale. Prompt processing moves a bit more (~2137 vs ~2115 t/s, +1%), but nothing that changes the conclusion.

vRAM doesn't budge either: still **4.8 Gi**, MTP on or off. Makes sense once you think about it: unlike classic *speculative decoding*, which loads a complete second draft model alongside the main one (and genuinely costs several hundred MB to several GB of extra vRAM), MTP here is just a small extra prediction head grafted onto the *same* model, not a separate model loaded in memory. Against the several GB already occupied by the model's weights and the KV-cache, its footprint is negligible — invisible at the tenth-of-a-GB scale we're measuring at here.

What would make sense in theory (MTP reduces the number of sequential passes, so it should limit the impact of a more expensive decode per pass) doesn't really hold up here. It's possible the MTP gain is already largely eaten up by the cost of verifying the proposed tokens, which also has to read the quantized KV-cache. Without finer-grained profiling, it's hard to dig further — but the observation stands: on this setup, MTP doesn't recover the lost speed. The real fix would be a quantization format natively supported by the hardware instead of dequantized in software at every token, which doesn't exist yet in our current pipeline.

Verdict for both cards, then: MTP costs almost nothing in vRAM, but doesn't gain almost anything in speed either, once the KV-cache is quantized. On this particular setup, there's no free round trip between the two.

## Conclusion

I picked these two cards for their price, their power draw, and their size — the kind of criteria that keep coming up for me, because my goal, as usual, is to do a lot with next to nothing.

We do manage to run models on these cards, and fairly cleanly at that. Getting these kinds of speeds, on these kinds of cards, at this price, is genuinely interesting: they're small, they sip power, and they remain relatively affordable given current GPU market prices. For local usage, it really makes sense. And beyond what we tested here, the pace the AI world is moving at right now — speculative decoding, quantization, MoE, Turboquant, and more — only makes this kind of setup more relevant over time.

Would I recommend these cards, though? No.

They're cheap, but they remain heavily limited on vRAM and especially memory bandwidth — the factor that matters most for generation speed, as we've seen throughout this article. And unless you already have a case or a bay designed for this kind of server card, you need to be ready to 3D-print shrouds and hack together some wiring to cool them properly. A fun project to run, but not exactly a foundation to build reliable day-to-day infrastructure on.

*This article was co-written with Claude. Everything in it may therefore contain errors, shortcuts, or approximations.*
