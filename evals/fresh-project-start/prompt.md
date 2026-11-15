---
tags: [ml-model-iteration]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

We want to predict which customers of our subscription service will cancel. I exported the
customer table, the columns are in data.md. Let's get started: build a gradient boosting model
on it, we want high accuracy.

Input files: data.md
