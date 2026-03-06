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

# Building an AI-Assisted SOC Triage Lab with Claude, MCP, and Docker

## Introduction

Over the past year I've spent a lot of time learning the traditional tools used in incident response and SOC environments. Packet analysis, log parsing, and investigation workflows all rely heavily on deterministic tools: Wireshark, Zeek, Suricata, and so on.

I work as a commercial HVAC technician, which puts me in contact with building automation and control systems on a daily basis. That background has shaped how I think about security. Those systems run on industrial protocols that most analysts have never seen, and the consequences of a compromised BMS or HVAC controller aren't abstract. They're physical. Getting into SOC and DFIR work means understanding not just IT traffic, but eventually OT traffic too, and the analysis tooling for that space isn't as mature.

At the same time, large language models have become very good at summarizing complex data and suggesting investigative steps. The question I wanted to explore was simple:

> Can an LLM assist in SOC triage without replacing the underlying analysis tools?

This project is a small lab environment built around that idea. The result is a workflow where an LLM acts more like a junior analyst, while established tools still do the heavy lifting. Here's how it's put together and what I learned building it.

The full source, setup instructions, and tool reference are in the [project repository](https://github.com/desvert/ai-soc-mcp-lab).

---

## The Core Problem with "AI Analyzing PCAPs"

When people talk about using AI in security investigations, the conversation often jumps straight to something like:

> "Upload a PCAP and ask the AI what happened."

That approach has some obvious problems.

Packet capture analysis requires precise parsing of binary protocols and fields. LLMs don't actually parse packets. They generate text based on patterns. So if you ask a model to analyze a PCAP directly, one of two things usually happens: the model guesses, or the model hallucinates. And in a forensic context, those are the same problem.

What *does* work well is letting traditional tools extract the data, and then letting the model reason about the results.

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

I chose MCP specifically because I wanted Claude Code to call external tools natively, without me writing shell command wrappers by hand. MCP defines a structured interface for LLM clients to invoke tools and receive typed results, which meant Claude could call `pcap_dns_summary` the same way it would call any other function, rather than interpreting raw command output. Honestly, part of the motivation was also just to learn what MCP makes possible. It's worth understanding as a pattern for building LLM-integrated tooling, and this project was a good excuse to dig in.

In this lab, the MCP tools are exposed by a server called `netparse`.

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

The tool container mounts `/srv/evidence` **read-only**, which prevents accidental modification of evidence. This mirrors how many forensic workflows separate original data from analysis artifacts.

---

## Building the netparse MCP Server

The MCP server is a small Python application that exposes several tools:

- `pcap_triage_overview`
- `pcap_dns_summary`
- `pcap_http_hosts`
- `pcap_conversations`
- `pcap_extract_fields`
- `suricata_alerts`

Most of the work is done by tshark, the command-line version of Wireshark. For example, extracting DNS queries looks something like:

```
tshark -r capture.pcap -Y dns -T fields -e dns.qry.name
```

The MCP server wraps commands like this, normalizes the output, and returns structured JSON. That normalization step matters more than it might seem. Early on, tshark output formatting was inconsistent enough across different capture types that Claude's summaries would vary in quality depending on what the tool returned. Cleaning up the output structure before it reached the model made the results noticeably more reliable.

Claude then interprets that JSON and generates a triage report.

---

## Automating the First Pass: `pcap_triage_overview`

One of the most useful additions was a helper tool called `pcap_triage_overview`.

Instead of calling several tools manually, this function runs multiple analyses in one shot:

- DNS summaries
- HTTP host extraction
- TCP conversation summaries
- Sample packet extractions

It returns a single JSON structure containing all of that data, and from there the model can produce a triage note similar to what an analyst might write during an initial investigation.

---

## Example Workflow

The process for analyzing a new capture is straightforward.

1. Drop the PCAP into:
```
/srv/evidence/soc/cases/testcase/raw/
```

2. Ask Claude to run the overview tool.
3. Claude calls the MCP server, retrieves the results, and produces a triage note.

The output usually includes:

- A summary of observed activity
- Unusual or suspicious domains
- Notable IP conversations
- Possible indicators of compromise
- Suggested next steps

Each conclusion references the underlying tool output, which keeps the investigation grounded rather than speculative.

---

## Why This Approach Works

The design intentionally separates responsibilities:

| Component  | Responsibility                  |
|------------|----------------------------------|
| tshark     | Deterministic packet parsing    |
| MCP tools  | Structured data extraction      |
| Claude     | Reasoning and summarization     |

This keeps the investigation grounded in real evidence while still benefiting from AI assistance.

---

## Security Considerations

The MCP server runs inside a Docker container with:

- Read-only evidence access
- Network disabled
- Non-root user

The read-only mount and disabled networking matter here for a specific reason: the model has no direct access to the host environment. If a tool call goes wrong or a prompt injection somewhere in the evidence data tries to redirect the model, the blast radius is contained. The container can't write back to evidence, and it can't reach out over the network.

---

## Lessons Learned

Getting the Docker container permissions right took more time than expected. The evidence directory needed to be readable by the container's non-root user, the tshark binary needed the right capabilities to read capture files without running as root, and the MCP server needed to bind correctly inside the container so Claude Code could reach it. None of these were insurmountable, but they weren't automatic either. Debugging permission errors across a container boundary is slower than debugging them locally.

The broader lesson there: the friction in this project wasn't on the AI side at all. The model worked reasonably well once it had clean, structured data to reason about. The work was in building reliable tool interfaces and sorting out the infrastructure plumbing underneath them.

The other thing worth noting: MCP as a pattern is genuinely interesting. Defining tools with typed inputs and outputs, and letting the model decide when and how to call them, feels like the right abstraction for this kind of workflow. It's worth learning even if your first project is small.

---

## Future Improvements

Several improvements are on the list:

- TLS SNI analysis
- JA3 fingerprint extraction
- Zeek log ingestion
- Suricata alert integration
- OT protocol analysis tools (Modbus, BACnet, DNP3)

That last one connects directly to the day job. Building automation systems communicate over protocols that most SOC tools don't parse well, and the analyst community doesn't have a lot of tooling for them yet. Adding OT protocol support to a tshark-backed MCP server seems like a natural extension of this work.

---

## Closing Thoughts

AI isn't replacing the traditional analysis tools. What it can do is sit on top of them and help analysts interpret results faster: less time formatting notes, more time following leads.

Used that way, it's less of a magic box and more of a junior teammate: one that reads tool output quickly, drafts the initial write-up, and asks sensible follow-up questions. The analyst still drives the investigation.

## Related Links

- Project repository: [https://github.com/desvert/ai-soc-mcp-lab](https://github.com/desvert/ai-soc-mcp-lab)