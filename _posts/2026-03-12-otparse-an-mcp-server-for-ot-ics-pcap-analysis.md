---
layout: post
title: "otparse: An MCP Server for OT/ICS PCAP Analysis"
date: 2026-03-12
categories:
    - blog
tags:
    - ot
    - ics
    - mcp
    - docker
    - packet-analysis
    - modbus
    - bacnet
excerpt: ""
---
 
# otparse: An MCP Server for OT/ICS PCAP Analysis
 
Most of the systems I work on every day run BACnet or Modbus. Building
automation controllers, variable air volume boxes, chillers talking to a
BAS -- it's all networked, and almost none of it was designed with security
in mind. That overlap between the physical world and the network is part of
what led me toward OT/ICS security.
 
One thing that comes up constantly in ICS security work is the packet
capture. You grab traffic off an industrial network and then you need to
figure out what you're looking at: which devices were communicating, what
protocol, what function codes were used. The tools exist, but the workflow
of loading a file into Wireshark or running tshark manually and reasoning
over the output isn't fast, especially when you're also trying to learn the
protocols themselves.
 
This project is a natural extension of the [AI-assisted SOC triage lab](https://github.com/desvert/ai-soc-mcp-lab)
I wrote about last week. That lab used an MCP server called `netparse` to
give Claude Code access to tshark for general network analysis. `otparse`
applies the same pattern to OT protocols specifically.
 
The full source and setup instructions are in the
[repository](https://github.com/desvert/otparse-mcp).
 
---
 
## What It Does
 
`otparse` is a containerized MCP server. It exposes two tools:
 
- `parse_modbus_pcap` -- extracts Modbus/TCP transactions from a saved
  capture and returns decoded frames along with a basic device inventory
  built from observed source and destination IPs
- `parse_bacnet_pcap` -- does the same for BACnet/IP traffic
 
You point either tool at a PCAP file inside the container, and it hands
back structured JSON. Function codes, unit IDs, object types, service
names, invoke IDs -- the fields that matter for understanding what was
happening on the network. Claude can then reason over that output directly.
 
The design is the same as `netparse`: tshark does the protocol dissection,
the MCP server normalizes the output, and the model handles the
interpretation. The value of keeping those responsibilities separate is that
the analysis stays grounded in what the tools actually found rather than
what the model guesses.
 
---
 
## Container Design
 
The container runs with no outbound network access and a read-only
`/evidence` mount. For anything touching forensic data, those felt like
non-negotiable constraints. The evidence can't be modified, and the
container can't reach out over the network even if something in a malicious
capture tries to redirect it.
 
It also runs as a non-root user, which is straightforward to set up in
Docker and removes another category of potential problems.
 
---
 
## What Came Out of Building It
 
The most useful early lesson was about tshark attribute consistency.
Pyshark exposes protocol fields as object attributes, but the attribute
names tshark uses for the same conceptual field aren't always consistent
across dissector versions or capture types. Both parsers handle this by
walking a list of candidate attribute names and returning the first one
that exists, which made the results noticeably more reliable across
different captures.
 
The other thing worth noting: Modbus direction detection (whether a frame
is a request or a response) depends on pyshark exposing the right flags
from the dissector. In practice this doesn't always happen cleanly, and a
lot of frames will come back with `"direction": "unknown"`. That's
something to improve in a future pass.
 
---
 
## Where This Is Going
 
v0.1.0 covers Modbus and BACnet because those are the two protocols I see
most in the field. The next version adds broader ICS protocol coverage:
EtherNet/IP, S7comm, DNP3, IEC 60870-5-104, OPC-UA, IEC 61850, and
PROFINET. That pass is already in progress.
 
After that, the plan is anomaly heuristics -- unusual function codes,
write-heavy sessions, broadcast storms, that kind of thing -- and
eventually a timeline summarizer that can give a higher-level picture of
what a capture contains before diving into individual frames.
 
---
 
Repository: [https://github.com/desvert/otparse-mcp](https://github.com/desvert/otparse-mcp)
 
 