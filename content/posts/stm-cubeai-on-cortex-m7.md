---
title: "STM CubeAI on Cortex M7"
date: 2020-08-25T22:01:35+03:00
draft: false
---

# Introduction

This post is a brief performance analysis for running machine learning inference (prediction) on a Cortex-M7 microcontroller. I will concentrate here on the micro-optimizations by inspecting the code at assembly level.

The trained model is borrowed from the benchmarks done by [Stupid Projects](https://www.stupid-projects.com/machine-learning-on-embedded-part-1/) and adapted to my setup.

## Test setup

The MCU under test is an STM32H7 with 2MBytes of flash and 1MByte RAM, running at 480MHz. For an embedded microcontroller, this is truly a high-end device.

STM32CubeMX version 6.0.0 with STM's X-CUBE-AI version 5.1.2 were used to generate the base for the project. As far as I understand, the [license](https://www.st.com/content/ccc/resource/legal/legal_agreement/license_agreement/group0/eb/8e/f9/c1/9e/64/49/95/SLA0048/files/SLA0048.txt/jcr:content/translations/en.SLA0048.txt) for X-CUBE-AI doesn't prohibit publishing the analysis what I'm about to show here.

GNU Arm Embedded Toolchain version 9-2019-q4-major with -O3 optimization level is used in these tests.

Both the instruction and data caches are enabled in the test.

On the test firmware I have [FreeRTOS](https://www.freertos.org/) running, but I disable the interrupts before running the inference to even eliminate any context switches from the benchmark.

## Machine learning model

The model used in this example is a copy of the one used in [Dimitri's](https://www.stupid-projects.com/machine-learning-on-embedded-part-4/) tests. It is a model for classifying a handwritten digit. The MNIST handwritten digits database is used to get the training data. I wanted first to use this exact same model for my test to see that I get comparable results and everything is working as expected.
Running it through STMCubeMX CubeAI plugin I get the memory consumption described as follows:
[![CubeAI MNIST model memory consumption](/images/stm32_cubeai/stm_cubeai_memory_usage.png)](/images/stm32_cubeai/stm_cubeai_memory_usage.png)

In this test I use a pre-made digit [found from here](https://github.com/dimtass/stm32f746-x-cube-ai-mnist/blob/master/source/src/inc/digit.h) as an input, though for the performance analysis it doesn't matter whether it's meaningful or random data.

## Inference without memory optimizations

The first test to get started, the X-CUBE-AI inference takes about 36ms on this test setup.
[![STM CubeAI on Cortex-M7, no TCM used](/images/stm32_cubeai/ai_run_trace_stat_no_dtcm_no_itcm.png)](/images/stm32_cubeai/ai_run_trace_stat_no_dtcm_no_itcm.png)

During this test, the CubeAI library code is running and the model weights are stored in the flash. The task which is running the inference has a stack which is placed in SRAM. The heap is also placed in SRAM. The activations are also placed in SRAM.


## Inference with some memory optimizations

This one has the RAM part (activations, that 32.23kB stated by CubeAI) of the inference moved to the tightly coupled data memory DTCM so the memory reads and writes are single cycle.

[![STM CubeAI on Cortex-M7, DTCM used](/images/stm32_cubeai/ai_run_trace_stat_dtcm_used.png)](/images/stm32_cubeai/ai_run_trace_stat_dtcm_used.png)

## Inference with more memory optimizations

The microcontroller under test has a total of 192kB tightly coupled memory which is divided between the instruction (ITCM, 64kB) and data (DTCM, 128kB) regions. The memory needed for the weights (that 257.54kB stated by CubeAI) is too large to put to tightly coupled memory, but the runtime fits to DTCM. In this test whole the runtime library (NetworkRuntime512_CM7_GCC.a in my case) is placed to DTCM.

[![STM CubeAI on Cortex-M7, DTCM and ITCM used](/images/stm32_cubeai/ai_run_trace_stat_dtcm_and_itcm.png)](/images/stm32_cubeai/ai_run_trace_stat_dtcm_and_itcm.png)

So what's going on here, the results of moving the runtime library to ITCM is basically the same as when it is running from flash? Well, I stated in the test setup section that the instruction cache is enabled. I'll do one more test having the instruction cache disabled to see the effect.

## Inference with instruction cache disabled

Here's a test where the AI runtime is running from flash, instruction cache disabled. I'll let the activations alone and let them stay in ITCM.

[![STM CubeAI on Cortex-M7, ITCM used, icache disabled](/images/stm32_cubeai/ai_run_trace_stat_no_dtcm_no_icache.png)](/images/stm32_cubeai/ai_run_trace_stat_no_dtcm_no_icache.png)

That looks sensible. If the runtime is running from flash and the instruction cache is enabled (why wouldn't it be), a more realistic benchmark would be to mess with the cache, invalidate it and so on. Also running the inference interrupts disabled would not be very realistic, unless such a 36ms *stop the world* period is acceptable for the application.

## Inference with data cache disabled

Here's a test where the AI runtime is running from flash, data cache disabled, instruction cache enabled. 

[![STM CubeAI on Cortex-M7, ITCM used, icache disabled](/images/stm32_cubeai/ai_run_trace_stat_no_dtcm_no_dcache.png)](/images/stm32_cubeai/ai_run_trace_stat_no_dtcm_no_dcache.png)


## Inference with caches disabled, no tightly coupled memory

Ok now I'm running backwards, but just for fun as a worst case test I'll disable both the data and instruction caches and don't use any TCM.

[![STM CubeAI on Cortex-M7, no caches, no TCM](/images/stm32_cubeai/ai_run_trace_stat_no_caches_no_tcm.png)](/images/stm32_cubeai/ai_run_trace_stat_no_caches_no_tcm.png)

Makes sense, move along, nothing to see here.


## Final test

This is the last test. I looked at the _data cache disabled_ numbers and started wondering how the performance got so bad. In the test setup section I stated that the AI runtime FreeRTOS task stack was in the SRAM. Finally, the test with everything that is possible in the TCM and caches enabled.

[![STM CubeAI on Cortex-M7, no caches, no TCM](/images/stm32_cubeai/ai_run_trace_stat_both_caches_all_in_tcm.png)](/images/stm32_cubeai/ai_run_trace_stat_both_caches_all_in_tcm.png)

Not a very surprising result, it's similar where I started from. So what's to learn from here? - Make sure that the caches are turned on, or use the TCM if possible. On this test I let it run more cycles, but as the inference is running interrupts disabled there shouldn't be any variation in the timing. If there was, something would be terribly wrong..

## Deeper dive

Let's have closer look at what is actually happening when running the inference. This test runs the inference, waits for a second and does another. The *ERROR* in the chart is basically when the CPU is running in the IDLE task. TraceON/TraceOFF breakpoints are around the aiRun function. 

[![STM CubeAI on Cortex-M7 large trace](/images/stm32_cubeai/ai_run_chart_symbol_out.png)](/images/stm32_cubeai/ai_run_chart_symbol_out.png)

Let's dive in deeper to the action.

[![STM CubeAI on Cortex-M7 single inference](/images/stm32_cubeai/ai_run_chart_symbol_single_inference.png)](/images/stm32_cubeai/ai_run_chart_symbol_single_inference.png)

There you have it, this is machine learning.

From this view it seems that the dominant functions are forward_conv2d_nl_pool and ... memcpy! Ok, it's not the full story, but certainly an interesting place for optimization. Let's see more closer.

[![STM CubeAI on Cortex-M7 single inference, zoomed in](/images/stm32_cubeai/ai_run_conv2d_memcpy.png)](/images/stm32_cubeai/ai_run_conv2d_memcpy.png)

From a wider view it is noticeable that there are two parts of the 2D convolution. In the first part (from the green cursor to the red cursor, about 8ms) the memcpy domination is not too bad.

[![STM CubeAI on Cortex-M7 single inference](/images/stm32_cubeai/ai_run_chart_symbol_wide_single_inference.png)](/images/stm32_cubeai/ai_run_chart_symbol_wide_single_inference.png)

The 'second part' of the convolution (from the red cursor to the blue cursor, about 23ms), memcpy plays a bigger role:

[![STM CubeAI on Cortex-M7 single inference, zoomed in](/images/stm32_cubeai/ai_run_conv2d_memcpy_2.png)](/images/stm32_cubeai/ai_run_conv2d_memcpy_2.png)

And how is the memcpy implemented? It is doing a byte by byte copy:

[![Byte by byte memcpy assembly](/images/stm32_cubeai/ai_run_memcpy_libc_nano.png)](/images/stm32_cubeai/ai_run_memcpy_libc_nano.png)

I'm linking the test firmware with `-specs=nano.specs`, replacing this memcpy with an optimized version is for another post. Though if I was creating the CubeAI library I would keep the memory copying in my own hands. 

And what about the convolution? This is a snippet of what is going on in the loop:

[![2D convolution assembly](/images/stm32_cubeai/ai_run_conv2d.png)](/images/stm32_cubeai/ai_run_conv2d.png)

What can you determine from this snippet? Well, at least you can see that the FPU registers are well utilized, almost all of them. For Cortex-M7 this means that the instruction dual issue is utilized at least in the loading of registers. The repeated fused multiply accumulate instructions with the same destination register raises questions if there is a better way to arrange the instructions.

## Conclusion

For STM CubeAI, at least for the 2d convolution, there seems to be a quick win in the optimization field if the copying of memory from one place to another is done more efficiently (not byte by byte).

Next up will be having a deeper look at Tensorflow TFlite. It will be interesting to compare the tight convolution loop with CubeAI (I admit, I already had a look).

-Petri