---
title: "What Limits Local AI Speed on an Arduino, a Raspberry Pi and a Mac Mini"
date: 2026-10-09
draft: false
tags: ["llm", "benchmarking", "raspberry-pi", "arduino", "claude-code", "hardware-in-the-loop"]
description: "Two weeks of measurements on an Arduino UNO Q, a Raspberry Pi 5 and a Mac mini M4, with every number tied to a raw file. Memory bandwidth sets the writing speed on two of them, not the third, a faster Pi clock can write slower, and the Arduino spends about half a joule per token."
---

I wanted to know what actually limits how fast a small language model runs, and I wanted numbers I could trust. The usual answer is two rules: memory capacity decides which model fits, and memory bandwidth decides how fast it writes. I spent two weeks testing how far that goes on the three devices I had: an Arduino UNO Q, a Raspberry Pi 5 and a Mac mini with an M4. Claude Code wrote the harness and ran most of the measurements. I'm a systems electrical engineer, not an ML person, so I wanted a method that ties every number to a raw file.

The short answer: the rules hold for two of the three and fail for the third, and two cheap tests tell you which case you are in. On the Pi and the M4, writing a token is limited by memory bandwidth. On the UNO Q it is limited by how fast its small cores execute the kernel's instructions, and a written token costs it about half a joule. The devices are the demonstration, not the subject.

## The method, in five habits

1. Anchor first: reproduce a published number with the same tool, settings and model file before trusting your own, and
   explain any gap over about 10%.
2. Log the device's state every second (clock, temperature, throttle flags, load, memory, power where available) and
   give every run a verdict. A run whose state breaks the test's assumption is flagged, never averaged in.
3. Keep a claims register: one row per number you might repeat, with its status, its raw file and its caveats, and a
   script that recomputes every number from the raw files. Retractions stay on record.
4. Write the prediction before the run, so the result can prove it wrong.
5. Measure the roofline reference rates (read bandwidth, int8 multiply rate) on the device itself and place every
   workload against them. That turns "it is slow" into "here is what it is short of".

Who did what: I defined the method and decided each step of the evaluation (what to measure next, how each run could fail, which hardware, every review). Claude Code wrote the harness, the scripts and the first drafts of the notes, and started sub-agents for some runs and for the independent audits. I cannot review its code line by line, which is why every number cites a raw file, and why the audits re-read the raw files against the register (they found no register number wrong, and their wording fixes are in). Scope: three ARM devices (four Mac configurations), the CPU path through llama.cpp release b11181 (the Apple anchor uses the build its table used) plus the M4's Metal GPU. Energy is measured on the UNO Q (at its 5 V input) and on the Pi (board rails only), not yet on the same meter; accelerators and the Pi on that meter are the next posts.

Every number in this post has a row in that register, with its raw result file, its status and its caveats. The register and the raw result files are not published yet.

## The devices

![A three-column table comparing the Arduino UNO Q, Raspberry Pi 5 and Mac mini M4: processor cores, instruction support, GPU, memory, and our measured read bandwidth (8.9, 12.5 and 86 GB/s on the M4's CPU) and speed on one 1.2-billion-parameter model (writing 5.2, 16.6 and 100 tokens per second; reading a prompt 7.2, 145 and 478).](figs/hardware.png)

| | Arduino UNO Q | Raspberry Pi 5 (8 GB) | Mac mini M4 (16 GB) |
|---|---|---|---|
| SoC | Qualcomm QRB2210 | Broadcom BCM2712 | Apple M4 |
| CPU | 4x Cortex-A53, 2.0 GHz, in-order | 4x Cortex-A76, 2.4 GHz, out-of-order | 4 performance + 6 efficiency cores |
| int8 instructions | plain NEON, no dot-product (SDOT) | SDOT and fp16 | SDOT, fp16, int8 matrix multiply (unused by the macOS build) |
| GPU | Adreno 702 (Vulkan 1.0 in our stack) | VideoCore VII | 10-core Apple GPU (Metal) |
| NPU | none in our setup | none | 16-core Neural Engine, unused here |
| RAM | 4 GB LPDDR4X (3.6 GiB usable) | 8 GB LPDDR4X-4267 | 16 GB unified, 120 GB/s rated |
| Storage | 32 GB eMMC | 32 GB microSD | 1 TB SSD |

Facts from the vendor pages and the machines themselves, one named source per field. A note on the Pi: it does have a GPU (VideoCore VII); llama.cpp does not use it for language models here, so every Pi number below is a CPU number. The UNO Q also carries a microcontroller next to its Linux processor (an STM32U585 with 2 MB of flash and 786 KB of SRAM, from Arduino's datasheet). A model has to fit in that, a different size class from the 1.2-billion-parameter files used here, so it is not part of these tests.

## Two rules, and when they fail

![A log-log chart of memory capacity against memory bandwidth for the UNO Q (4 GB, 8.9 GB/s), the Pi 5 (8 GB, 12.5 GB/s), the Mac mini M4 (16 GB, 102 GB/s) and five larger devices taken from vendor specs, with horizontal lines for the writing speed the speed rule predicts for a 7B and a 1.2B model. The Pi writes at 92% of its bandwidth and the M4 at 83%; the UNO Q reaches only 41%.](figs/landscape.png)

Capacity. Resident size is about parameters times bytes per parameter (0.5625 for Q4_0, about 0.6 for Q4_K_M, about 1.06
for Q8_0) plus the context cache and the runtime's copies. One rule of thumb says 75% of RAM is usable and the largest model, in billions of parameters, is the usable GB divided by 0.6. That is the Q4_K_M line of the arithmetic. It hides two things: small models' "Q4_0" files
keep the vocabulary table at 6 or 8 bits, 16 to 45% of the bytes a token reads, and repacking for dot-product
kernels nearly doubles resident memory, 1366 MB on the Pi against 717 MB on the UNO Q for the same 4-bit 1.2B file.

Speed. Tokens per second while writing is at most measured read bandwidth times an efficiency (about 0.9 at the memory
wall) divided by bytes per token. With the throttled chip's own measured bandwidth, the rule predicted the Pi's 7B model within 5%. It fails on the UNO Q, which
writes at 41% of its measured bandwidth and 34% of its measured multiply rate at the same time: neither memory bandwidth nor arithmetic is the limit, the kernel's instruction count is, and the rule, at the efficiency the Pi and the M4 reach, overstates its speed about 2x.

The gray squares in the chart are bigger devices from their vendors' spec pages, not measured by us. They show that capacity and bandwidth are bought separately: some have a lot of memory and moderate bandwidth, some the reverse.

## Anchors first

![A dot plot of how far our speeds are from published numbers, in percent: the Mac mini M4's GPU within 1%, the UNO Q within about 4%, and the Pi 5 (hollow dots, throttled runs) within 8% of three published rows. Every dot is inside the 10% band the project asks to be explained.](figs/anchors.png)

Each device has a published number we reproduced first. The M4 matches llama.cpp's own Apple table within 1% on the
exact 2023 build the table used (Llama-2-7B Q4_0: 222.4 vs 221.3 tokens per second reading, 23.9 vs 24.1 writing); a fourth run on a quieter machine gave 222.5 and 24.9, within 4%.
The UNO Q matches the one published UNO Q figure within 4% (Gemma 3 1B Q4_0: 10.0 vs about 9.7 reading, 5.67 vs about 5.5
writing). The Pi lands within 10% of published Pi 5 rows for other 7B-class 4-bit files run with the same tool and settings
(Llama-2-7B Q4_0: 14.4 reading, 2.37 writing; their build and cooling are not stated, so this one is a plausibility check
rather than an exact anchor). That last one holds a miss: the prediction written before the run was 2.6 to 3.1 tokens per second,
it landed at 2.37 because the passive case throttled the run, and the prediction from the chip's hot ceilings, 2.47, was
within 5%. Writing the prediction down is what made the miss visible.

## Three machines, three limiters

![Five log-log roofline panels (UNO Q, Pi 5 with fan, M4 Linux VM, M4 CPU, M4 GPU) with each machine's measured memory and arithmetic ceilings and the 1.2B model's measured speeds placed on them. Writing reaches 94% of the memory ceiling on the Pi, 81 to 83% on the M4 and only 41% on the UNO Q.](figs/roofline.png)

One small program, the same binary on every device, measured each one's read bandwidth and int8 multiply rate. They are reference rates, not hardware limits; a share carries about 10% uncertainty. Against them, same 1.2B
model, 4 bits, four threads: the Pi's writing streams weights at 92% of its read rate (the chart plots the fan-cooled run, 94%) and the M4 CPU's at 81 to 86% (90 to 93% at 8 bits), the memory wall; the UNO Q reaches 41% of its read rate and 34% of its multiply rate, neither.

A GPU adds arithmetic more than bandwidth. On the M4, the Metal GPU reads a prompt at 1544 tokens per second against the CPU's 523, 3.0 times faster, but writes at 130 against 98, only 1.3 times faster (1.2 times in the first run), about the ratio of their read rates, 102 against 86 GB/s; in the first run its writing sat at 83% of its own read rate, the same wall. The UNO Q's GPU (Adreno 702, Vulkan 1.0) was 7.3 times slower than its CPU on a small vision model, and llama.cpp's Vulkan backend needs a newer Vulkan, so no language model ran on it.

Two tests could have proved that wrong. A chip at the memory wall cannot speed up by adding
cores once one core fills the bus: the Pi's writing goes from 14.2 tokens per second on one thread to 16.9 on four while
its prompt reading scales 3.6x; the UNO Q's writing scales 3.7 to 3.8x. And an 8-bit file reads 1.79x the bytes of
the 4-bit file per token, so a bandwidth-bound machine must slow to about 56%: the Pi slows to 56%, as predicted, the M4 CPU to 58 to 64%, the UNO Q keeps 93%. On the Arduino, nearly doubling the bytes costs almost nothing, because bytes are not what it is short of.

![Two log-log panels of speed-up against thread count, for reading a prompt and for writing. The UNO Q writes 3.72 times faster on 4 threads, near the ideal diagonal; the Pi 5 and the M4 Linux VM flatten at about 1.2 to 1.6 times; the M4 CPU peaks at 6 to 8 threads (1.9 times) and dips at 10. An open square marks the first M4 run's 10-thread collapse (0.14 times), which did not reproduce.](figs/threads.png)

![Two panels of horizontal bars showing the speed of the 8-bit (dark) and mixed 4/6-bit (light) files relative to the 4-bit file on the same machine. Writing with the 8-bit file runs at 56% on the Pi (the bytes alone predict 56%), 60 to 67% on the M4 variants and 93% on the UNO Q, which is not limited by bytes.](figs/precision.png)

## Inside the A53: count the instructions

![Stacked bars of the instructions in one pass of the Cortex-A53's inner loop per 64 multiply-accumulates: the 4-bit loop has 54 instructions of which 16 multiply, the 8-bit loop has 48, and the microbenchmark's test kernel has 16, all multiplies. The rest unpack, rescale, load and loop.](figs/kernel_mix.png)

We disassembled the exact kernel library the board runs. The 4-bit inner loop spends 54 instructions per 64
multiply-accumulates and only 16 of them multiply; the rest load, unpack and rescale. The 8-bit loop needs 48, since
nothing is unpacked. No runtime option changes it (all within 2%). A better kernel has at least about 2.5x of
room.

The same missing instruction decides how a prompt is read: reusing a weight across a block of tokens needs a blocked
kernel, which on ARM needs the dot-product instruction. A 128-token prompt therefore raises the UNO Q's arithmetic
rate only 1.23x over writing one token, against 4.2 to 4.7x on the M4 CPU and 10.5 to 11x on its GPU. In practice a smart-home
request with about 120 tokens of instructions takes 20.2 seconds on the UNO Q, 16.8 of them reading the instructions: the system prompt sets the wait, not the reply. (The latency chart computes a 72-token email from benchmark speeds; the 20.2 seconds is a measured request.)

![Horizontal stacked bars of waiting time for a 72-token email (panel A) and a 150-token answer (panel B) on seven setups, split into the wait for the first word and the time to write the rest: the UNO Q takes 19 s and 34 s, the Pi 5 about 4.5 s and 9 s, the M4 variants 0.6 to 1.8 s and 1.3 to 3.1 s.](figs/latency.png)

How much is the instruction and how much the core? We forced the Pi onto the UNO Q's kernels. At clocks matched within 4% the A76
runs the same plain code 3.5x faster on prompts and 2.8x on writing than the A53 (core and memory system); its
dot-product kernels then add 5.8x on prompts but only 1.1x on writing, because the plain loop already writes at 81% of
the Pi's bandwidth. An instruction helps only where it removes the bottleneck. That number is itself a correction:
the first split (7.0x and 1.4x) compared a hard-throttled plain-kernel run with dot-product runs that were only soft-limited; the state
logs showed the mismatch, the claim was superseded within minutes and cool reruns gave the figures above: the method caught its own error.

![Two log-scale bar panels, prompt reading and writing, showing how many times faster than the UNO Q each machine is. The Pi 5 is 3.46 times (prompts) and 2.82 times (writing) faster running the UNO Q's own plain instructions, then 5.82 times and only 1.12 times faster again with the dot-product instruction. The M4 Linux VM is split the same way.](figs/isa_split.png)

## Heat

In a passive case the Pi 5 trips its 80 C limit within a minute of continuous writing. With the real clock recorded, the
clock falls from 2383 to 1514 MHz (-36%) over ten minutes while writing falls from 16.6 to 13.2 tokens per second
(-21%); speed went as the clock to the power 0.49, about half, as a memory-bound workload should. A desk fan delays
the limit from 51 to 162 seconds and holds the loss to 6%. Measured on the throttled chip, the multiply rate fell 37% and the four-thread read rate 9%; the pinned-clock tests below show that is the clock, not the heat. The trap: the standard Linux clock reading said
2.4 GHz through all of it, because it reports the clock requested, not delivered; every Pi clock column recorded before
we noticed is marked wrong. The UNO Q never slowed through a 20-minute run at up to 74 C; its cores do too little per second to get hot.

![Speed and temperature over ten minutes of continuous writing. The passive Pi 5 hits its 80 C limit at 51 seconds and loses 21% of its speed (16.6 to 13.2 tokens per second); with a desk fan it hits the limit at 162 seconds and loses 6%; the UNO Q stays flat at 5.1 tokens per second and never reaches 80 C. A small panel shows the fanned Pi cooling to 35 C.](figs/thermals.png)

![Two groups of bars for the Pi 5: dot-product arithmetic in GOP/s and read-only memory bandwidth in GB/s, measured cold, hot after the suite on the passive Pi, and with a fan after ten minutes of load. Hot, arithmetic is 37% lower (317.5 against 507.7) and bandwidth 9% lower (11.32 against 12.48); with the fan both are back within 0.2%.](figs/heat_ceilings.png)

The direct test pins the clock at fixed steps from 1500 to 2400 MHz, twice in opposite orders, first in the passive case and then with a fan on the Pi. Two results. Temperature does nothing at a pinned clock: the chip at 51 to 63 C writes as fast as the same chip at 65 to 81 C, within about 1%, 0.4 to 0.5% slower if anything. A hot Pi is slow only because the firmware lowers its clock; the fan's value is the delay before it does and the higher clock it settles at afterwards. And over the 60% of extra clock between 1500 and 2400 MHz, prompt reading gained 51% (speed went as the clock to the power 0.9) while writing gained only 23%: reading is arithmetic and follows the clock, writing waits on memory.

## A faster clock that writes slower

![Left: writing speed rises from 13.4 to 15.8 tokens per second between 1500 and 1900 MHz, drops 6.4% to 14.8 at 2000 and rises to 16.6 at 2400. Right: arithmetic, L1 and L2 throughput per cycle stay at 100%; L3 speed drops to about 91% and the DRAM stream to about 83% at 2000 MHz.](figs/clock_step.png)

Mapped at every 100 MHz, the writing curve is not smooth. It rises from 13.4 tokens per second at 1500 MHz to 15.8 at 1900, falls 6.4% to 14.8 at 2000, and climbs
again to 16.6 at 2400; 2000, 2100 and 2200 MHz are all slower than 1900, in both passes. (The first, coarser sweep showed this as a dip at 2100 MHz, which we put down to chance and then to heat. The fan-cooled repeat and the finer grid showed it is a step.)

Where is it? Not in the arithmetic: the int8 rate per MHz is the same at every clock, and the core voltage rises smoothly, 15 mV per 100 MHz, with no jump.
A latency ladder points at the memory side. L1 and L2 behave identically at every clock, but from 2000 MHz up the shared L3 answers in 10% more CPU cycles and one thread
streams 17% fewer bytes per cycle from DRAM, a ratio of 0.83, consistent with 5 to 6, per MHz. A read-only imitation of the model's memory pattern (four threads reading 0.69 GB in 200
tensors with a barrier after each, no arithmetic) steps more (9.5% against the model's 6.4%) and keeps a nearly constant ratio to the real writing speed across all ten clocks (within about 2%). A prediction written down beforehand also held: the 8-bit file, which reads 1.79 times the bytes per token, steps by 7.1% at the same clock, more than the 4-bit file's 6.4%. So the cause sits in how the path to memory scales with the CPU clock at that operating point, which looks like a clock-ratio change; whether the firmware sets it, and which divider, I cannot see from Linux.

The practical reading: capped at 1900 MHz this Pi writes at 95 to 96% of its full-clock speed on 26% less power, and 2000 to 2200 MHz are slower than 1900 while drawing the same or more power. "More clock" is more speed only if every stage between the core and the data scales with it.

## 4-bit versus 8-bit

![Three panels against bits per weight for the 1.2B model quantized from 8.5 down to 4.8 bits, plus 4-bit files made with more care (quantization-aware, or with an importance matrix). KL divergence from the full-precision file rises at every step; the share of positions with the same top word falls from 98.5% to 79.4%; perplexity stays within 0.5% of full precision until the 4-bit files, where plain Q4_K_M and Q4_0 rise 12.6% and 17.3%.](figs/quant_ladder.png)

Speed is the bandwidth story again: 8-bit costs 7 to 10% on the UNO Q and 36 to 44% on the Pi and the Mac, and
needs 1.7x the resident memory. On quality, what I could see was smaller than what is there.
In blind votes between the 8-bit and 4-bit versions of one model I called 4 of 7 pairs equal, and our automatic
checks scored them 13 and 12 of 24: noise. Measured as llama.cpp does (perplexity and KL divergence against the
full-precision file over a 300,000-token text), the ladder is clear: 8-bit keeps the full-precision next word 98.5% of
the time, Q6_K 95.2%, Q5_K_M 92.0%, a quantization-aware 4-bit file 84.9%, Q4_K_M 83.3%, plain Q4_0 79.4%. Perplexity
stays within 0.5% of full precision down to Q5_K_M, then rises to +1.9% for the quantization-aware file and jumps to +12.6% and +17.3% for Q4_K_M and Q4_0. Plain 4-bit ranks a different word first at about one position in five of the test text: invisible in a handful of answers, but there. How the file is made matters
as much as the bit count: the same 1B weights quantized with an importance matrix keep the next word 86.2% of the time,
without one 81.0%. One model family each, on encyclopedia text.

## Same weights, different words

![A matrix of the number of identical answers, out of 36 prompts, between each pair of the M4 GPU, M4 CPU, Pi 5 and UNO Q (18 to 23 for different chips; all four agree on 14), and a strip chart of how many characters pairs of answers share before they first differ (medians 50 to 99).](figs/chips.png)

Same model file, greedy decoding, 36 prompts: any two chips write word for word the same answer on half to two thirds of
them; on one machine the text repeats exactly, 52 of 52. What sets
the words is the kernel path, not the chip: the Pi writes the same 28 of 28 answers as the M4's Linux VM, which runs the
same kernel library on different silicon, and 24 to 25 of 28 as the Mac GPU and CPU, whose kernels differ. Where
the addition order differs, texts split at a close call between two words. The verdicts do not: all three chip families
give the same pass/fail result on at least 23 of 24 checks.

## Batching: predict, then measure

![Three panels of aggregate writing speed against the number of sequences decoded together. On the Pi 5, cool and hot, the measured speed (41 tokens per second at 8 sequences, 2.6 to 3.1 times one sequence) sits far under the textbook prediction (125 cool) and the kernel-aware prediction (90 cool), with a sawtooth at 5 to 7 sequences; the UNO Q gains only 1.09 times at 4.](figs/batching.png)

Decoding several conversations at once lets a memory-bound chip read each weight once for all of them: a naive roofline promises up to 8x at 8 streams. Reading llama.cpp's repacked kernel before the run said otherwise: it handles streams in blocks
of 4 and sends leftovers through single-row passes, so the gain should be a sawtooth. The Pi measured 15.8, 18.3, 38.8
and 41.3 tokens per second in aggregate for 1, 2, 4 and 8 streams (2.6x at 8), and 5 to 7 streams were slower than 4. Every one of these runs was on the passive Pi and reached its 80 C limit, so all are flagged; the hot runs at 1500 MHz show the same shape (13.2, 15.6, 37.3 and 41.5). The sawtooth was on record before the data existed. The UNO Q, with no idle arithmetic to share, gains 1.09x at 4
streams.

## The biggest models that fit

The sharpest test of "speed follows bytes per token" is a mixture-of-experts model, which stores many experts but reads
only a few for each token. LFM2-8B-A1B (8.3B parameters, about 1.5B active per token) writes 8.9 to 9.4 tokens per
second on the passive Pi, 3.8 to 3.95x the dense 7B, for 4.05x fewer bytes read per token: 95% of the bytes ratio, against about 10 predicted before the run for the throttled clock. By file size alone it should have been slower than the 7B.
Prompt reading gains less, 2.5x.

It also showed what the capacity rule misses. The 4.7 GB file fits the Pi's 8 GB at rest, but llama.cpp's default load
builds a repacked second copy first. With the file partly in the page cache, 7 of 12 loads swapped and 2 were killed by
the kernel's out-of-memory killer; loading without repacking was clean and cost 1.77x on prompt reading, almost nothing on
writing. Fitting is not loading.

On the UNO Q the largest dense models that fit, a 3B and a 4B, run within 2% of the prediction (1.9 and 1.5 tokens per
second writing) and use the same 41 to 42% of the bandwidth as the 1.2B, so the speed rule holds there with the UNO Q's
own efficiency. They fit. At 46 seconds just to read a smart-home instruction, they are not usable.

## Is any of it useful?

For a smart-home command the 1B models name the right device almost every time (70 of 90 replies) and the right action
almost never (15 of 90): the device name lands in the action field, so the automation would not run. In a lighter
two-line format the best 1.2B model passes 4 of 5 requests and the sub-1B models stop following the format at all.
On the UNO Q that costs 20 seconds per request.

The first fix we tried was one worked example in the instructions. It did nothing: 5 of 30 right actions without it and 5 of 30 with it, 15 and 20 of 120 over twenty commands, and the prediction written beforehand, over 60%, failed. What worked was constraining the
reply itself: llama-server can force the output to follow a JSON schema built from the prompt's own device and action
lists, and that gave 28 of 30, and 103 of 120 over twenty commands, with nothing extra to read. The fix was in the
runtime, not in a bigger chip.

And most of the 20 seconds can go. llama-server can keep the processed instructions and reuse them, but with its
defaults this model reused nothing: its hybrid layers can only be rewound to a saved checkpoint, and the server's log
shows the default checkpoint landing a few tokens past the point where two commands start to differ. A micro-batch of 8
tokens (-ub 8) moves a checkpoint early enough: 109 of 121 tokens reused, the first word in 1.5 to 1.9 seconds instead of
16.3 to 17.0, five commands in 41 seconds instead of 101. A model with attention layers only reuses by default.

## Energy: what a token costs

![Left: board power over time at the UNO Q's 5 V input. Writing a 1,500-token answer holds about 2.3 W for 347 seconds (2.35 W in the first minute, 2.19 W in the last) and reading a prompt about 1.8 W for 287 seconds, against 0.455 W idle. Right: three bars of energy per token, each split into the board's idle draw and the extra draw for the work: reading a prompt 0.25 joules (3.9 tokens per joule), writing 128-token answers 0.47 (2.1) and writing 1,500-token answers 0.53 (1.9).](figs/uno_energy.png)

The UNO Q has no power sensor, so I measured it from outside. I put an inline USB-C meter between the Mac's port and the board. The Mac cannot see the meter as a device, so I chose to read its display with a desk webcam, twice a second, and had Claude Code write the reader. A reading counts only if the meter's watts equal its volts times its amps within 3%, and a run's energy is the area under the power trace. I had the board's LED animation killed (Claude Code flashed an empty sketch to its microcontroller) because it would add to the number. The first run started with Wi-Fi and Bluetooth on; I asked why not switch them off for every run, so we stopped it, kept its files marked as aborted, and started over with them off. Wi-Fi and Bluetooth cost 0.047 W at idle, 9% of it; idle with them off is 0.455 W.

Writing a 1,500-token reply, the board draws 2.28 W at 4.33 tokens per second, so each token costs 0.53 J, or 1.9 tokens per joule; three runs agree within 0.7%. At the 128-token length the usual benchmark uses, it draws 2.43 W at 5.13 tokens per second: 0.47 J, 2.1 tokens per joule. Reading a prompt draws less, 1.82 W, but runs faster, 7.18 tokens per second, so a prompt token costs 0.25 J, 0.48 times a written one. Power also falls along a long answer, by 7 to 9% between the first minute and the last (2.35 W to 2.19 W in the first run), as the speed falls with the growing context.

This board is slow rather than frugal: it adds about 1.8 to 2.0 W on top of idle to write 4 to 5 tokens per second. That fits the earlier finding that its limit is the instructions its small cores execute, not memory. A better kernel has at least about 2.5 times of speed room; if it ran at about the same power, a joule would go that much further. That is a projection, not a measurement.

I wrote the prediction first, and it missed: 1.3 to 1.8 W and 2.9 to 4.0 tokens per joule while writing, against 2.28 W and 1.90 measured. The article that gave the one published UNO Q speed I matched above also says about 1.4 W under load, for a different model, and I could not find how it measured; I did not reproduce that figure. And a caution about my own number: I have not checked the meter against a reference instrument. All I checked is that its readings agree with each other, watts against volts times amps, and my summed energy against the meter's own charge counter, within 0.8 to 2.3%.

On the Pi's board rails a 1.2B model gives about 2.5 tokens per joule over a whole run, about 2.7 to 3.0 counting only the generation windows. The cheap lever is the clock: pinned at 1900 MHz the rails averaged 4.6 W against 6.2 W at 2400 for 4% less writing speed, which is 3.45 against 2.66 tokens per joule (whole-run power), the best of the six clocks tried. The Pi's figures are its board rails only, with no power supply or USB, and the UNO Q's are the whole board at its 5 V input, so the two cannot be compared yet. The Pi needs the same meter first.

## Limitations

One board of each kind, one run per thermal configuration, no temperature log during the hot ceiling measurement. The M4 numbers are good to about 5 to 10% run to run: a rerun at 8 to 9% background instead of 18 to 20% moved prompt reading by up to 10% and Metal writing by 6.5%, four repeats scattered 11% on prompt reading, and the Mac never reached a strictly quiet state; the M4 VM rows moved more, 7 to 33%. The Pi's clock step is one chip on one firmware, and its mechanism is inferred from latency and bandwidth, not observed. The UNO Q's energy comes from one meter read by webcam and not checked against a reference instrument, and its idle reading drifted from 0.471 to 0.450 W across the session for a reason I did not find; the Pi's energy is a different measuring point. The ceilings are reference rates, not
hardware limits. The A53 explanation is inferred from instruction counts, not performance counters. Quantization: one
model each, one text. Human quality: one rater, 24 votes. Apart from the M4's Metal GPU, no GPU or NPU has run a language model here. Not covered: image, video and speech generation, phones, a voice-assistant pipeline, and sending requests from a small board to a bigger machine; small vision and speech models did run on the boards earlier in the project, but they are not part of this post. The register, the raw files and the harness are not published yet.

## Doing this on your own device

Adding a device to my harness is meant to be a profile, not a port: a transport profile plus a spec file with a named source for each field. One script installs the same llama.cpp release and Python environment, one pushes the same model files, and one suite script runs the same sequence everywhere, reference rates and kernel check included. A cost script turns bytes and operations per token plus the reference rates into the prediction you write down first; a verify script recomputes every registered number afterwards. What is device-specific today: real clock and throttle flags are read only on a Raspberry Pi; power is read from the Pi's power chip and, on the UNO Q, from an external meter read by webcam; macOS gives no clock or temperature without sudo; the roofline program is ARM NEON only; and no GPU or NPU engine is wired in beyond one Vulkan vision test.

If you measure your own board, the first thing to publish is not the tokens per second. It is the anchor you reproduced, the state log of the run, and the prediction you wrote down before it.

---

*[How this was built](/how-i-work/): Claude Code wrote the harness, the analysis scripts, the figures, the program that reads the power meter's display and the first drafts of this post, and started sub-agents for some runs and for the independent audits of the numbers. I defined the method (anchor to a published number first, log the device's state on every run, write the prediction before the run, tie every number to a raw file) and decided how to proceed at each step of the evaluation: what to measure next, what would count as a failure, which result to distrust and rerun, and to read the power meter with a webcam and switch the radios off for every run. I also set up the boards and their cooling, the meter and the camera, cast the blind votes, and reviewed every draft. Tested on the real hardware: every measurement above, on one UNO Q, one Pi 5 and one Mac mini M4 (the email times in the latency chart are computed from those speeds), and the UNO Q's power with one USB meter that a webcam read. Not tested: a second board of each kind, that meter against a reference instrument, the Pi on the same meter, and what inside the Pi's firmware changes at 2000 MHz.*
