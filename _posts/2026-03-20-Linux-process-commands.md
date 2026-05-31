---
layout: post
title: "Linux Process Manipulation"
date: 2026-03-20
categories: [linux]
---

## Process Manipulation Commands

```bash
ps -ef | grep [n]ginx # use the following command to see the Nginx's parent (master) and the child (worker) processes
pstree -p -s $(pgrep -o nginx) # To see the process in a tree format
```

> Here, systemd is the root parent process of the Ubuntu server, so the PID is 1.



