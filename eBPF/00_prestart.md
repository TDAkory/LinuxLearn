# Pre-Start

- [Libbpf Vs. BCC for BPF Development](https://devops.com/libbpf-vs-bcc-for-bpf-development/#:~:text=BCC%20relies%20on%20kernel%20header,relying%20on%20a%20vmheader%20file.)

## What is BCC?

BCC stands for BPF Compiler Collection and is one of the oldest ways to develop BPF applications. It helps you embed your BPF code into your user-space program in the form of a plain string. When the user-space program is executed by the kernel, BCC invokes its embedded Clang/LLVM, pulls in system-wide kernel headers and compiles the program on the spot. 

Since BCC compiles BPF programs on the host machine, it ensures that the memory layout your BPF program expects is precisely the same as that of the target host.

BPF programs are designed to be injected directly into a kernel, so BCC tools seem to be the perfect solution for developing BPF applications. However, they have proven to be bulky in the modern context due to their heavy reliance on the Clang/LLVM combination, which is resource-intensive. This has led to the need for a better, more modern solution to developing BPF apps.

## What is [Libbpf](https://github.com/libbpf)?

Libbpf is one of the hottest new BPF tools on the market. It is usually coupled with BPF CO-RE (which stands for compile once, run everywhere). BPF CO-RE enables you to generate binaries that run on multiple kernel versions.

The idea behind Libbpf is to make BPF development as similar to other forms of development as possible. With BPF CO-RE, Libbpf does this by compiling BPF programs into small binaries that can be deployed to multiple deployment hosts. Libbpf does the setup work like loading and verifying programs, creating maps, attaching to hooks, etc., which enables developers to focus on more critical tasks at hand, such as program performance and correctness.

Libbpf aims to eliminate the overheads associated with BPF app development and deployment by reducing the dependency on system-wide kernel headers as well as Clang/LLVM libraries for compilation on runtime.

## What is libbpf-bootstrap

libbpf-bootstrap is a scaffolding playground setting up as much things as possible for beginner users to let them dive straight into writing BPF programs and tinkering with them without unnecessary frustrations of initial setup. It takes into account best practices developed in BPF community over last few years and provides a modern and convenient workflow with, arguably, best BPF user experience to date. libbpf-bootstrap is relying on libbpf and uses a simple Makefile. For users needed more advanced set ups, it should be a good starting point. At the very least, if Makefile can't be used directly, it's simple enough to just transfer the logic to whichever build system needs to be used.

## What is eunomia-bpf

eunomia-bpf is a dynamic loading library/runtime and a compile toolchain framework, aim at helping you build and distribute eBPF programs easier.

## What is [Cilium](https://cilium.io/get-started/)？

Cilium is an open source project to provide networking, security, and observability for cloud native environments such as Kubernetes clusters and other container orchestration platforms.

