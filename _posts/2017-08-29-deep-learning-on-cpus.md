---
layout: post
title: "Getting Started with Deep Learning on CPUs"
date: 2017-08-29
comments: true
---

This isn't really a comprehensive guide to getting started with deep learning in general as there are great resources already out there that cover the territory. This is aimed more for those looking to dip their toe into the waters of deep learning with whatever they happen to have lying around.

I won't lie to you -- you won't get great performance compared to GPUs on deep learning applications today if you're only using CPUs.  That being said, if for whatever reason you don't have access to a decent NVIDIA GPU, either physically or through a cloud computing service like AWS, you can still work on deep learning projects with some patience and mindfulness. I've seen a fair few simple projects trained on CPUs that nonetheless offer some interesting initial results.

This simple guide today will focus on getting better CPU performance out of Tensorflow, either directly or as a backend. If you're not using GPUs, I would definitely not recommend using a pre-compiled Tensorflow binary, such as can be installed via pip -- you're going to want to create your own optimized build from source. However, the optimizations you can make are dependent on your CPU architecture, so YMMV. I'm running on an Intel Xeon E5-1650 from a workstation I acquired and the performance gains I received were fairly substantial simply by enabling the use of optimized instructions.

You can follow the standard instructions to install from source from the [official Tensorflow documentation](https://www.tensorflow.org/install/install_sources). When you're at the step where you're configuring the build via `./configure`, don't enable OpenCL or CUDA support if you don't have a GPU. Then, when you're supposed to issue a `bazel build` command, use the following to compile with [SSE](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions) and [AVX](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions) extensions:

```
bazel build -c opt --copt=-mavx --copt=-mavx2 --copt=-mfma --copt=-mfpmath=both --copt=-msse4.2 -k //tensorflow/tools/pip_package:build_pip_package
```

If you want to double-check whether your processor supports the mentioned extensions, in Linux, check the `flags` section of your `/proc/cpuinfo` by using `cat /proc/cpuinfo | grep 'flags' | uniq`. For Mac OS X, you can use `sysctl machdep.cpu.features` in your terminal.

Using the following toy Keras model and Tensorflow installed from pip, training on the MNIST handwritten digits dataset hovered at ~90s per epoch:

```
model = Sequential()
model.add(Conv2D(32, (5, 5), activation='relu',
                 input_shape=(28, 28, 1))
model.add(MaxPooling2D(pool_size=(2, 2)))
model.add(Dropout(0.2))
model.add(Flatten())
model.add(Dense(128, activation='relu'))
model.add(Dense(self.n_classes, activation='softmax'))
```

Running with a version of Tensorflow custom-built from source with optimized instructions, training was reduced to a steady 29s per epoch. As a note, I have an EVGA GeForce GTX 960 that I repurposed from another machine, which although not a very powerful card, still reduced training further to ~11s per epoch.
