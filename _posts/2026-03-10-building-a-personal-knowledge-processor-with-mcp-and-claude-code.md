---
layout: post
title: "Building a Personal Knowledge Processor with MCP and Claude Code"
date: 2026-03-10
---

There is a gap between raw notes and something useful.

After every lab session, every Cisco module, every HVAC testbed experiment, I end up with a folder full of half-formed markdown files, terminal screenshots, and log dumps. The ideas are there. The structure is not. Turning that pile into a write-up worth keeping has always been the friction point.

I built something to close that gap. It is called knowledgeops-mcp, and it is a Dockerized MCP server that gives Claude Code access to a notes folder. Drop in your raw material, describe what you want, and Claude Code produces a structured write-up with a self-quiz section and a short project blurb. The full source and setup instructions are in the [repository](https://github.com/desvert/knowledgeops-mcp).

---

## Why MCP

If you have used Claude Code, you know it is capable of a lot on its own. What MCP adds is the ability to expose your own tools as first-class capabilities the model can call natively.

Without MCP, getting Claude Code to process a folder of local files means either pasting content into the prompt manually or writing a wrapper script that handles everything itself. Neither scales well. The first is tedious. The second puts all the orchestration logic in the script, which means the model is just a function call at the end of a pipeline rather than an active participant in the process.

With MCP, you define the tools and Claude Code decides how and when to use them. It can scan a folder, inspect what it finds, read the contents, reason over everything, and save the result, with full visibility at each step. The model is doing the thinking. The server is doing the file I/O.

This project is built on the same pattern as my earlier [AI-Assisted SOC Triage Lab](https://github.com/desvert/ai-soc-mcp-lab), where I used MCP to give Claude Code access to network analysis tools like tshark, and to parse Suricata alert logs. Same architecture, different domain. The more I use this pattern, the more it feels like the right way to wire AI tooling into a local workflow.

---

## What the Server Does

The MCP server exposes three tools:

**knowledgeops_scan_folder** returns an inventory of everything in a folder, grouped by file type, with sizes. No content yet, just a picture of what is there.

**knowledgeops_read_folder** reads everything and hands it back to Claude Code. Text files and logs come back as plain text. PDFs are extracted page by page. Images and screenshots are base64-encoded so Claude Code can process them directly through vision, no OCR dependency needed.

**knowledgeops_save_outputs** writes the finished write-up and blurb to an `output/` subfolder inside the notes folder.

That is the entire server. It contains no AI logic, no calls to the Anthropic API, and no decision-making. Claude Code receives the raw content and handles everything from there using your subscription.

---

## The Knowledgeops Concept

The name comes from "knowledge operations," which is a slightly pretentious way of saying: treat your notes like structured data, not a personal journal.

The problem with raw notes is that they are optimized for capture, not retrieval. Bullet points that made sense at 11pm during a lab session are opaque two weeks later. Screenshots without context are worse. The write-up that comes out of this tool is structured for the future reader, who is usually a future version of yourself trying to remember what you actually learned.

There are two outputs. The write-up is a full markdown document whose structure Claude Code determines based on the content. Study notes tend to become structured reference documents with comparison tables. Lab logs tend to become narrative write-ups that walk through what happened and what was found. Both end with a self-quiz section in flashcard format, which turns out to be the most immediately useful part when a topic comes up again weeks later.

The blurb is short: three to five sentences, specific about tools and techniques, written for a LinkedIn post or a GitHub README intro. Humble by design. It takes the same session and produces something shareable in about thirty seconds.

---

## Where This Fits

I am finishing a Bachelor's in Cybersecurity Technology while working full time as a commercial HVAC technician. The coursework, the lab work, the portfolio projects, and the Cisco certification prep all generate notes, and none of it sits neatly in one place.

This tool is part of a broader workflow I am building for myself: capture raw, process into structure, publish in layers. The structured write-up goes into a private notes archive. A cleaned version becomes the technical README for a GitHub repo. The blurb becomes a LinkedIn post. The blog post, like this one, tells the story behind the build.

The goal is not to have AI write my notes for me. The goal is to make it cheap enough to do the processing step that I actually do it, rather than leaving a folder of raw captures that gradually becomes useless.

---

## What Came Out of Building It

The main lesson from this project was about architecture decisions and their downstream consequences. The first version of this tool called the Anthropic API from inside the server. It worked, but it created a billing split between the Claude Code subscription and the API, and it mixed two different data handling policies in a single workflow. Untangling that required rethinking where the AI reasoning should live.

Putting the reasoning inside Claude Code and keeping the server as pure file I/O solved both problems. It also made the server simpler, which made it easier to reason about and easier to trust.

The second lesson was about path handling. Anything that reads from and writes to disk on your behalf deserves explicit path validation, even for a personal tool running locally. The server validates that every path stays within the mounted notes directory before touching anything. It takes about ten lines of code and removes a category of potential problems entirely.

A third post will go deeper on the API versus subscription question and what Anthropic's September 2025 privacy policy update means for how you think about which tool to use for what kind of work.

---

The repository is at [github.com/desvert/knowledgeops-mcp](https://github.com/desvert/knowledgeops-mcp). Setup takes about five minutes if you have Docker and Claude Code installed.