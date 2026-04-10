# Day 1: SPDK Overview

## What SPDK is

SPDK is a set of user-space storage libraries and applications. The code is mainly organized as:

- `lib`: core SPDK libraries
- `include/spdk`: public headers
- `app`: full applications built from the libraries
- `examples`: smaller reference programs
- `doc`: architecture and feature documentation
- `test`: unit and functional tests

Source reference:

- `doc/overview.md`

## Core architectural idea

SPDK is designed around:

- user-space execution
- asynchronous I/O
- polling instead of interrupt-driven/blocking model
- message passing instead of heavy shared-state locking

The most important high-level statement in the docs is that SPDK is designed around message passing instead of locking. SPDK libraries depend on a threading abstraction, but are intentionally decoupled from one specific application framework.

This is the first mental shift compared with the Linux kernel storage path:

- Linux commonly relies on kernel threads, interrupts, scheduler decisions, and shared kernel subsystems
- SPDK prefers explicit ownership, poll loops, callbacks, and cross-thread messages

## Threading model at a high level

SPDK provides a framework for asynchronous, polled-mode, shared-nothing server applications.

Key concepts:

- reactor: one event loop thread per CPU core
- event: a message sent to a reactor
- poller: a repeatedly executed callback on a thread

The framework connects reactors with lockless queues and uses events for cross-thread communication.

Why this matters:

- event functions should be short and non-blocking
- pollers replace many interrupt-driven patterns
- concurrency is achieved through asynchronous operations and callback completion

This is the second major mental shift:

- not "sleep until work arrives"
- instead "poll, dispatch, complete, and send messages"

Source reference:

- `doc/event.md`

## Environment abstraction

SPDK uses POSIX for many operations, but abstracts system-specific functionality behind the `env` layer.

Important examples:

- PCI enumeration
- DMA-safe memory allocation

By default, SPDK uses a DPDK-based environment implementation, but the architecture is intentionally abstracted.

This means SPDK is not just "an application". It is a framework with portability and embedding in mind.

## Application model

SPDK applications usually:

- start with a small set of command-line options
- initialize framework components
- expose runtime configuration through JSON-RPC

This is different from Linux configfs-based management. In SPDK, the normal operational model is:

- start app
- configure using RPC
- inspect and modify state dynamically

## The role of bdev

The block device layer, or bdev, is the central storage abstraction for most SPDK services.

The bdev layer provides:

- a common API for block devices
- pluggable modules for different backend types
- a common interface for read, write, unmap, reset, and related operations
- lockless queueing and asynchronous completion
- JSON-RPC-based management

Conceptually, bdev in SPDK plays a role similar to the generic block layer in a traditional OS storage stack, but implemented as a user-space library.

This is important because upper-layer services like NVMf usually do not talk directly to raw hardware. They export bdevs.

Source reference:

- `doc/bdev.md`

## Big-picture stack

A useful mental model is:

- SPDK application framework
- thread/reactor/event/poller model
- service modules such as NVMf
- bdev framework
- backend bdev modules such as malloc, aio, uring, nvme, lvol, raid
- actual hardware or Linux-backed device/file

For NVMf specifically:

- NVMf receives a host command
- resolves subsystem and namespace state
- translates the operation into bdev I/O
- the backing bdev module performs the actual operation
- completion returns asynchronously

## Comparison with Linux kernel storage

Linux kernel model often looks like:

- syscall or block layer request
- kernel block subsystem
- driver
- interrupt/completion

SPDK model looks more like:

- user-space RPC/app control
- user-space storage service
- bdev abstraction
- asynchronous request submission
- poller/event-driven completion

This does not mean "no synchronization exists". It means the architecture tries to minimize contention by:

- per-thread ownership
- message passing
- local context
- non-blocking operation

## What matters most to remember after Day 1

1. SPDK is a framework of libraries plus applications, not just one target binary.
2. Message passing is a core design rule.
3. Polling is fundamental to SPDK performance and concurrency.
4. The event framework is optional, but its model explains how many SPDK apps behave.
5. The bdev layer is the common storage abstraction that many higher-level services depend on.
6. JSON-RPC is the normal runtime control plane.
7. To understand NVMf well, you must first understand thread/reactor/poller and bdev.

## Suggested next step

Day 2 should focus on:

- application startup
- subsystem initialization
- RPC control flow
- how `spdk_tgt` and `nvmf_tgt` fit on top of the framework
