---
title: "TensorFlow Lite on Cortex M7"
date: 2020-10-21T17:23:47+03:00
draft: false
---

# Just a brief note

I have been too busy to dive deep on to analyzing the performance of the TensorFlow Lite micro,
but as I already did some prep work I'll just post what I did. My intention still is to take the plunge
to the inner workings of TFLite, hopefully sooner than later.

This post is just a quick follow up to compare TFLite micro to STMCubeAI which I was testing on the previous post. The TensorFlow source code used in the build was taken from the master branch at around 12th of August 2020.

## Inference

This trace shows it all, it is running the inference in a loop:

[![TensorFlow Lite inference on Cortex-M7](/images/tflite/tflite_inference.png)](/images/tflite/tflite_inference.png)

Now although this loses to STMCubeAI on performance, it looks much better as there is no copying of memory all the time
as was in the case of CubeAI. Certainly this can be optimized to run much faster.

Here's the timing of the inference, it takes about 63ms compared to STMCubeAI's 36ms.

[![TensorFlow Lite inference duration on Cortex-M7](/images/tflite/tflite_inference_duration.png)](/images/tflite/tflite_inference_duration.png)

## Inner loop

The inner loop of the 2d convolution is very neat and tidy, yet it leaves much room for optimization.
Only a few registers are used, loop unrolling might help etc. Need to keep in mind branch prediction and caching among other things though, optimization is not as simple as it used to be. 

[![TensorFlow Lite convolution inner loop on Cortex-M7](/images/tflite/tflite_convolution_inner_loop.png)](/images/tflite/tflite_convolution_inner_loop.png)

## What next

These simple tests were using 32-bit float models so the ARM CMSIS NN doesn't help here, the convolution falls back to the reference design. Next up will be definitely 8-bit quantized models and running an optimized inference.

-Petri