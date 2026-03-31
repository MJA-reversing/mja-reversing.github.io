---
layout: post
title: "How Malware Executes Before Entry Point: TLS Callbacks"
date: 2026-03-30
categories:
  - malware-analysis
  - reverse-engineering
tags:
  - tls-callbacks
  - pe-format
---

## Introduction

In my last blog I triaged a piece of malware in two hours with the goal of extracting as many Indicators of Compromise (IOCs) as possible. During that analysis a binary was extracted from memory. The binary contained multiple Thread-Local Storage (TLS) callbacks.

As a more experienced malware analyst I recognize this as a potential anti-analysis technique. There are legitimate uses for TLS callbacks, but I will explain what these are and how they can be used by malware authors to thwart analysts best efforts. I was unable to dig deep into the TLS callbacks (Two hours is not enough time for a full RE!), so I wanted to follow up with an explanation of what TLS callbacks are and how they can be used by malware to execute code before the entry point.

---

## What Are TLS Callbacks?

TLS callbacks are functions in a Windows program that are executed automatically when a process starts or when a new thread is created.

They are part of the Thread Local Storage (TLS) mechanism (https://learn.microsoft.com/en-us/windows/win32/procthread/thread-local-storage) and are defined in the PE file’s TLS Directory. Unlike the normal program entry point, these callbacks execute before the main function, allowing code to run very early in the program’s lifecycle.

While they are intended for initialization tasks, this early execution makes them particularly interesting in malware analysis, as they can be used to run code before a debugger reaches the program’s entry point.

---
## How does malware exploit this?

Executing code before the main function is a powerful tool that malware authors can abuse! There are a few ways malware authors will leverage this legitimate mechanism.

### Execution before entry point

Typically, a Windows program execution starts at the entry point. Often times a malware analyst will begin their analysis here. A malware author can evade analysis by placing malicious code in a TLS callback function that executes before the entry point. Here are just a few examples below:

### Using TLS callbacks as a loader

One of the most common ways malware abuses TLS callbacks is by using them as a loader for the real payload.

The TLS callback may:

- decrypt an encrypted payload stored in the binary
- allocate memory for the payload
- copy the decrypted code into memory
- transfer execution to the decrypted payload

By the time the entry point executes, the real malicious code is already running in memory. This makes the program appear harmless during basic static analysis, even though the actual payload has already been executed.

### Anti-analysis and anti-debugging

Another common abuse of TLS callbacks is leveraging the execution before an entry point as a way to thwart analysis. Because malware can run this code before the entry point, malware will abuse this to check for analysis tools or debugging tools.

Common checks include:

- detecting debuggers
- checking for virtual machines
- identifying sandbox environments
- performing timing-based detection

If any of these checks succeed, then the malware could terminate itself before reaching the entry point. An example that I have seen is a TLS callback where the malware checks to see if the first byte of the main function equals 0xCC (single-byte opcode representing a breakpoint within a debugger). If the check sees that the first bytes are 0xCC, then the main function is overwritten with 0xFF bytes. Otherwise, the code executes as normal.

This is a simple, yet effective anti-debugging technique. From an analysts perspective, it may appear that the sample isn't doing anything or is just some buggy code.

### Multiple callbacks to hide behavior

More advanced samples use multiple TLS callbacks instead of just one. Each callback can perform a different task, such as:

- running anti-debugging checks
- decrypting the payload
- injecting code into another process
- cleaning up traces

This spreads the malicious logic across several hidden execution points and makes the behavior much harder to follow during analysis.

---
## Real world example

A few weeks ago I wrote a blog about a sample that I spent 2-hours analyzing. Staying true to the rules of the exercise I ended my analysis right as my 2-hour timer went off. However, the malware sample I analyzed made use of TLS callbacks. So, this is a good opportunity to review this code and see what it is used for.

Metadata:
- md5 =    EC02D2A2BF9E988B4A6F4746106F4D2E
- sha1 =   C757982FFFA2AFEA377799357983920731437BE9
- sha256 = A5F433F76F818C90E3B65A8FA46E1CB55506F5EB7883B164D3280B4FB0931D6F
### Analysis

Fortunately for analysts, x64dbg automatically sets breakpoints on TLS callbacks. Additionally, decompilers will automatically identify TLS callbacks in most cases. This is helpful when starting analysis of a sample the utilizes TLS callback functionality.

The TLS callback itself is located at the address 0x1401F2C0. Upon further analysis, the callback itself doesn't contain an malicious code. Instead, it iterates through a callback arrays-global global structure that register Rust's runtime components.

In other words, this is legitimate code used to initialize components of Rust. However, this analysis did reveal the Rust dependencies that are used by the malware. The following were observed in this brief analysis:

- Tokio: Async runtime
- Rustls: TLS 1.3 encryption
- Hyper: HTTP client
- Rusqlite: Local data storage

As explained in the previous post, this malware is an info stealer. Specifically, it attempts to steal crypto currency wallet information among other details about the infected device. The TLS callback initializes the info stealing infrastructure in this context. Not the most exciting outcome, but these dependencies help analysts understand how the sample works. Artifacts this sample can be found here: https://github.com/mja-reversing/malware-analysis-artifacts/Rust-infostealer1 

---
## Conclusion

TLS callbacks are a small feature in the PE format, but they can have a huge impact on how malware behaves and how analysts approach a sample. Because they execute before the program’s entry point, they give malware authors an opportunity to run code earlier than most analysts expect—whether that means decrypting a payload, performing anti-debugging checks, or quietly preparing the real malicious functionality in memory.

In the real-world sample reviewed here, the TLS callback ultimately turned out to be legitimate Rust runtime initialization rather than hidden malicious logic. Even so, analyzing it was still valuable. It revealed important details about the malware’s dependencies, execution flow, and overall design. Sometimes the biggest takeaway from reversing isn’t discovering something dramatic—it’s confirming what _isn’t_ malicious and using that information to better understand the rest of the sample.
