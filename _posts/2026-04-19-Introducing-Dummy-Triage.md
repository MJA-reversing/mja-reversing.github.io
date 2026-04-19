---
layout: post
title: "Introducing Dummy Triage"
date: 2026-04-19
categories:
  - malware-analysis
tags:
  - new-tool
  - triage
---

## Dummy-triage.py

There are an overwhelming number of tools in the cyber security space. Between researchers building specialized tools and the growing use of AI-generated code, there is a growing number of open-source tools daily (even hourly with AI helping write the code). In my experience, many open-source scripts require some tweaking to fit a specific workflow, and it can be difficult to find a tool that produces exactly what you need out of the box. To address this, I built a lightweight triage tool of my own.

Dummy-triage.py is a straightforward Python script that performs the following tasks:

- File hashing (MD5, SHA256)
- String extraction and basic indicator parsing
- PE (Portable Executable) header and metadata analysis
- Structured JSON reporting

The goal of this tool is to automate the very first step of my malware analysis workflow. While many tools already perform this process, I’ve found that building and maintaining your own lightweight scripts can greatly expedite the malware analysis process. I also plan to integrate this into future analysis pipelines, so I’m sharing it in case it’s useful to others.

Future improvements include:

- ELF and Mach-O support
- Improved string filtering and extraction
- Suspicious import detection
- YARA rule integration
- Batch file processing

Please feel free to give it a try: https://github.com/MJA-reversing/dummy-triage
