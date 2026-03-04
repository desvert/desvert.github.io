---
layout: post
title: "Building an AI-Assisted SOC Triage Lab with Claude, MCP, and Docker"
date: 2026-03-04 00:45:32
categories: 
    - blog
tags: 
    - claude
    - soc
    - automation
    - docker
    - packet-triage
    - investigation

---

# **Building an AI-Assisted SOC Triage Lab with Claude, MCP, and Docker**


## Introduction

Over the past year I’ve spent a lot of time learning the traditional tools used in incident response and SOC environments. Packet analysis, log parsing, and investigation workflows all rely heavily on deterministic tools: Wireshark, Zeek, Suricata, and so on.

At the same time, large language models have become very good at summarizing complex data and suggesting investigative steps.

The question I wanted to explore was simple:

> Can an LLM assist in SOC triage without replacing the underlying analysis tools?

This project is a small lab environment built around that idea.

The result is a workflow where an LLM acts more like a junior analyst, while established tools still do the heavy lifting. 

---

## The Core Problem with “AI Analyzing PCAPs”

When people talk about using AI in security investigations, the conversation often jumps straight to something like:

> “Upload a PCAP and ask the AI what happened.”

That approach has some obvious problems.

Packet capture analysis requires precise parsing of binary protocols and fields. LLMs don’t actually parse packets. They generate text based on patterns.

So if you ask a model to analyze a PCAP directly, one of two things usually happens:

1. The model guesses.
2. The model hallucinates.

Neither is great if you care about evidence.

What _does_ work well is letting traditional tools extract the data, and then letting the model reason about the results.

---

## Architecture Overview

The lab architecture looks like this:

```
Claude Code  
   ↓  
MCP tool server  
   ↓  
Docker container  
   ↓  
tshark  
   ↓  
evidence directory
```

The key piece connecting everything is the Model Context Protocol (MCP).

MCP allows an LLM client like Claude Code to call external tools in a structured way. Instead of writing shell commands, the model calls functions that return structured data.

In this lab, those functions are exposed by a tool server called `netparse`.

---

## Evidence Directory Design

All investigation artifacts live under a single directory:

```
/srv/evidence/soc/cases/<case>/
```

Each case has a simple structure:

```
raw/  
derived/  
reports/
```

The tool container mounts `/srv/evidence` **read-only**, which prevents accidental modification of evidence.

This mirrors how many forensic workflows separate original data from analysis artifacts.

---

## Building the netparse MCP Server

The MCP server itself is a small Python application.

It exposes several tools:

- `pcap_triage_overview`
- `pcap_dns_summary`
- `pcap_http_hosts`
- `pcap_conversations`
- `pcap_extract_fields`
- `suricata_alerts`

Most of the work is done by tshark, the command-line version of Wireshark.

For example, extracting DNS queries looks something like:

```
tshark -r capture.pcap -Y dns -T fields -e dns.qry.name
```

The MCP server wraps commands like this and returns structured JSON.

Claude then interprets that JSON and generates a triage report.

---

## Automating the First Pass: `pcap_triage_overview`

One of the most useful additions was a helper tool called `pcap_triage_overview`.

Instead of calling several tools manually, this function runs multiple analyses:

- DNS summaries
- HTTP host extraction
- TCP conversation summaries
- sample packet extractions

It returns a single JSON structure containing all of that data.

From there, the model can produce a triage note similar to what an analyst might write during an initial investigation.

---

## Example Workflow

The process for analyzing a new capture is simple.

1. Drop the PCAP into:
```
/srv/evidence/soc/cases/testcase/raw/
```

2. Ask Claude to run the overview tool.
3. Claude calls the MCP server, retrieves the results, and produces a triage note.

The output usually includes:

- summary of activity
- unusual domains
- suspicious IP conversations
- possible indicators
- suggested next steps

Importantly, each conclusion references the underlying tool output.

---

## Why This Approach Works

The design intentionally separates responsibilities:

|Component|Responsibility|
|---|---|
|tshark|deterministic packet parsing|
|MCP tools|structured data extraction|
|Claude|reasoning and summarization|

This keeps the investigation grounded in real evidence while still benefiting from AI assistance.

---

## Security Considerations

The MCP server runs inside a Docker container with:

- read-only evidence access
- network disabled
- non-root user

This prevents the model from interacting directly with the host environment.

---

## Lessons Learned

A few interesting lessons came out of building this lab.

First, most of the complexity wasn’t in the AI side. It was in building clean tool interfaces that returned reliable data.

Second, containerizing the tools made the system much easier to reason about.

And third, the LLM was actually most helpful when generating investigation notes and suggesting follow-up questions.

---

## Future Improvements

There are several improvements I’d like to add:

- TLS SNI analysis
- JA3 fingerprint extraction
- Zeek log ingestion
- Suricata alert integration
- OT protocol analysis tools

This project is also connected to some of my broader interests in OT and critical infrastructure security, where packet analysis and protocol understanding are especially important.

---

## Closing Thoughts

This lab is a small experiment, but it demonstrates something I think will become common in security workflows.

AI isn’t replacing the traditional tools.

Instead, it can sit on top of them and help analysts interpret results faster.

Used that way, it becomes less of a “magic box” and more like a junior teammate that helps organize the investigation.

## Related Links

- Project repository:
    [https://github.com/desvert/ai-soc-mcp-lab](https://github.com/desvert/ai-soc-mcp-lab)
