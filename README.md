[![License](https://img.shields.io/badge/License-BSD_2--Clause-orange.svg)](https://opensource.org/licenses/BSD-2-Clause)
[![License: GPL v2](https://img.shields.io/badge/License-GPL_v2-yellow.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)
[![GitHub release (latest by date)](https://img.shields.io/github/v/release/Mellanox/libxlio)](https://github.com/Mellanox/libxlio/releases)
[![Documentation](https://img.shields.io/badge/docs-NVIDIA-green.svg)](https://docs.nvidia.com/networking/software/accelerator-software/index.html#xlio)

# XLIO - NVIDIA® Accelerated IO

* [Introduction](#introduction)
* [Installation](#installation)
* [Running XLIO](#running-xlio)
* [Architecture](#architecture)
* [System Requirements](#system-requirements)

## Introduction

The NVIDIA® Accelerated IO (XLIO) software library accelerates TCP/UDP network applications, delivering high bandwidth, low latency, and reduced CPU utilization. Built as a user-space library with a kernel-bypass architecture, XLIO enables direct data transfer between application memory and the network adapters for optimal performance.

**POSIX Socket API:** XLIO exposes the standard POSIX socket API, offering zero-cost integration, no application code changes are required, to achieve significant networking performance improvements. Applications can be accelerated simply by preloading the XLIO library.

**Ultra Socket API:** The XLIO Ultra API is designed for applications seeking maximum performance. While it requires integration effort, it delivers advanced capabilities including true zero-copy data transfers and a simplified, highly-optimized data path.

**Key Features:** XLIO seamlessly integrates with cryptography-enabled NVIDIA® ConnectX® network adapters and NVIDIA® BlueField® DPUs, providing hardware-offloaded Transport Layer Security (TLS) symmetric encryption and decryption. XLIO also leverages hardware features such as Large Receive Offload (LRO), TCP Segmentation Offload (TSO), and Striding-RQ to enhance performance for both POSIX and Ultra API users.

Please visit our [documentation website](https://docs.nvidia.com/networking/software/accelerator-software/index.html#xlio) for more details.

## Installation

XLIO is distributed as part of the [NVIDIA DOCA Software Framework](https://developer.nvidia.com/networking/doca). Follow the "DOCA Installation Guide for Linux" in the [NVIDIA DOCA documentation](https://docs.nvidia.com/doca/sdk/index.html), then install XLIO using one of the methods below.

> **Note:** For more installation options, refer to the installation section in our [documentation website](https://docs.nvidia.com/networking/software/accelerator-software/index.html#xlio).

### BF-Bundle

XLIO is included by default. No additional installation required.

### DOCA-Host

Install the `doca-libxlio` meta-package:

```sh
$ <your-package-manager> install -y doca-libxlio
```
> **Note:** The `doca-libxlio` package contains the XLIO user-space library and all required driver stack components. When following the DOCA Installation Guide, you can install `doca-libxlio` instead of a full DOCA profile (e.g., `doca-all`).

### From Source

#### Build Tools

The following packages are required:
- Autoconf
- Automake
- libtool
- unzip
- patch
- libnl-devel (netlink 3)

#### DPCP Library

Build and install DPCP (Direct Packet Control Plane) from source:
```sh
$ git clone https://github.com/Mellanox/libdpcp.git
$ cd libdpcp
$ ./autogen.sh
$ ./configure --prefix=/where/to/install
$ make -j
$ make install
```

#### XLIO Build

```sh
$ git clone https://github.com/Mellanox/libxlio.git
$ cd libxlio
$ ./autogen.sh
$ ./configure --prefix=/where/to/install --with-dpcp=/path/to/dpcp --enable-utls
$ make -j
$ make install
```

**Configuration Options:**
- `--with-dpcp=/path/to/dpcp` — Specify custom DPCP installation path, or use `--with-dpcp` if DPCP is installed in a standard system location
- `--enable-utls` — Enable hardware-offloaded TLS for supported NVIDIA adapters (optional)

For additional configuration options, see [docs/configuration.md](./docs/configuration.md).

## Running XLIO

For more detailed instructions, refer to the "XLIO Run" section in our [documentation website](https://docs.nvidia.com/networking/software/accelerator-software/index.html#xlio). See also the [README](./README) file for XLIO features and parameters.

### POSIX Socket API

**Sockperf**

```sh
$ LD_PRELOAD=libxlio.so sockperf \<params\>
```

Repository: [Sockperf](https://github.com/Mellanox/sockperf)

**Nginx**

```sh
$ LD_PRELOAD=libxlio.so XLIO_NGINX_WORKERS_NUM=\<N\> nginx \<nginx_params\>
```

> **Note:** `N` is the number of Nginx workers.

### Ultra Socket API

The `xlio_ultra_api_ping_pong.c` example demonstrates a simple client-server application without complex resource management. It serves as a starting point for getting familiar with the XLIO Ultra API.

For more details, see [examples/README.md](./examples/README.md).

## Architecture

![](docs/arch.png)

### Supported Transports
* IPv4/6
* TCP
* UDP

### Execution Modes

XLIO supports two execution modes to accommodate different application requirements and architectures.

> **Note:** For detailed configuration and usage, refer to the "XLIO Library Architecture" section in our [documentation website](https://docs.nvidia.com/networking/software/accelerator-software/index.html#xlio).

#### Run to Completion Mode

In Run to Completion (R2C) mode, network operations are executed directly in the context of application threads. When the application makes a socket API call, XLIO performs all necessary work within the context of that call. XLIO can execute hardware polling and progress other sockets as needed. Available for both POSIX and XLIO Ultra API.
This mode delivers optimal performance when applications are designed with efficient socket handling patterns, such as per-thread socket ownership and sufficient execution time for XLIO operations.

#### Worker Threads Mode

Worker Threads mode provides greater flexibility and ease of use compared to Run to Completion mode, making XLIO more accessible to applications that weren't specifically designed with high-performance networking in mind. Available for POSIX API only.
This mode is ideal for applications with shared sockets across threads, single listening threads, or those that call socket APIs infrequently.

## System Requirements

For detailed specifications, refer to the "System Requirements" section in our [documentation website](https://docs.nvidia.com/networking/software/accelerator-software/index.html#xlio).

**CPU Architectures:**

* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [Arm](https://www.arm.com/)

**Network Adapters:**

* [ConnectX®](https://www.nvidia.com/en-eu/networking/ethernet-adapters/)
* [BlueField®](https://www.nvidia.com/en-eu/networking/products/data-processing-unit/)

**Operating Systems:**
* [Linux](https://en.wikipedia.org/wiki/Linux)
