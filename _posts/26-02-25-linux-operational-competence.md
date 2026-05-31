---
layout: post
title: "Linux Operational Competence"
date: 2026-02-26
categories: [linux]
---
##

### Resource Health & Process Management

As an SRE, you need to know exactly why a container or node is "choking." 
This section covers how the OS views CPU, RAM, and active tasks.

| Command | Purpose | Key Flag/Usage |
| :---: | :---: | :---: |
| htop | Visual process viewer | Use F6 to sort by MEM% or CPU% |
| ps aux | Static process snapshot | ps aux --sort=-%mem (find memory hogs) |
| kill -9 'PID' | Force stop process | Use kill -l to see all signal types (like SIGTERM vs SIGKILL) |
| free -h | RAM usage | Focus on the "available" column, not "free." |
| uptime | System load | Shows load averages for 1, 5, and 15 minutes |

The SRE Skill: Understand Load Average.
> If you have 4 CPU cores and a load average of 10.0, your system is queuing tasks and performance is degrading.

---

### Storage & Filesystem Navigation

In a world of logs and Docker layers, disk space (or lack thereof) is a common cause of "NodeNotReady" status in K8s.

```bash
df -h # Check disk space globally. Always look at the %Use column.
du -sh /var/log/* # Summarize the size of specific directories to find what's eating your disk.
ls -R # Recursive list.
ncdu # (Optional tool) If you can install it, it’s a game-changer for navigating disk usage via a UI.
find / -name "*.log" -size +100M # Find large log files quickly.
```

---

### Networking & Connectivity

You need to verify if your Nginx deployment is actually listening on the port you think it is, and if the firewall is blocking it.

```bash
ip a # Check IP addresses and interface states (Up/Down).
ss -tulpn # The modern replacement for netstat. It shows which processes are listening on which ports.
ip route # See where your traffic is going (default gateway).
curl -Iv <http://localhost> # The -I fetches only headers; -v (verbose) shows the full handshake. Crucial for debugging Nginx.
dig or nslookup # Essential for debugging K8s CoreDNS issues.
```

---

### Systemd & Logging (The "Journal")

Since you are using Ubuntu, systemd is the heart of your service management.

```bash
systemctl status docker # Check if a service is running, errored, or stopped.
systemctl restart nginx # Apply changes.
journalctl -u docker -f # Follow (-f) logs specifically for the Docker unit.
journalctl -p err # View only "Error" level logs across the entire system.
dmesg -T # View kernel logs (useful for identifying OOM-Kills where the RAM runs out and the kernel kills a process).
systemctl list-units --type=service # output the list of services in the system.
```

---

### journalctl: Working with system logs

You can control the log view by using the keys to scroll up, down, and execute other control commands.

```bash
journalctl -u 'service-name' # display all service logs,
journalctl -f # view the log in real time,
journalctl --since 'date and time' # view the log for a specific period of time,
journalctl --until 'date and time' # view the log up to a certain time.
journalctl -p '0|1|2|3|4|5|6|7' # display log messages of a certain priority level.
```

---

### The "Glue": Pipes, Grep, and Redirection

This is how you process data. You don't read a 1GB log file; you filter it.

```bash
grep -i "error" /var/log/syslog # Search for "error" case-insensitively.
tail -f /var/log/nginx/access.log | grep "404" # Watch 404 errors happen in real-time.
awk '{print $1}' # Great for pulling the first column (like IP addresses) from a log file.
sort | uniq -c # Count occurrences.
```

Example:

```bash
cat access.log | awk '{print $1}' | sort | uniq -c # Tells you which IP is hitting your server the most.
```

### Your First Practice Mission

Log into your Ubuntu server and try to answer these three questions using only the commands above:

- Which process is currently using the most Resident Memory (RES)?
- Is any process currently listening on Port 80 or 443?
- Search your journalctl for any mentions of "OpenSearch" from the last 1 hour.
