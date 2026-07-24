---
layout: post
title: "Building the Pixel Experience: A Guide to AOSP Kernel Customization & Bazel Builds"
date: 2026-07-24
category: android
tags: [aosp kernel customization, bazel builds]
description: "One to two sentence summary."
---

# Building the Pixel Experience: A Guide to AOSP Kernel Customization & Bazel Builds

## Prerequisites
Before we begin, ensure your Linux machine (Ubuntu 22.04 is highly recommended) meets these requirements: at least a 16-core CPU, 64GB of RAM, and 500GB+ of NVMe storage. You don't need to install Bazel manually; AOSP now includes `bazelisk` in its tree to manage the correct version automatically.

## Step 1: Syncing the Bazel-Ready Source
We need to grab a branch that actively supports Bazel, such as `android14-bazel`.
```bash
mkdir aosp-bazel && cd aosp-bazel
repo init -u https://android.googlesource.com/platform/manifest -b android14-bazel
repo sync -j8
```

## Step 2: Customizing the Kernel via the Config GUI
Modern Android uses Kleaf (Bazel for kernels). We can launch the `menuconfig` GUI through the Bazel build wrapper:
```bash
cd common
tools/bazel run //common:kernel_x86_64_menuconfig
```
Navigate to **General setup**, then **Local version - append to kernel release**, type `-BazelCustom`, save, and exit.

## Conclusion
Configuring the kernel via `menuconfig` and compiling AOSP using Bazel is a glimpse into the future of Android system development. Getting comfortable with this workflow today will give you an edge as Google phases out legacy Make builds entirely.