---
layout: post
title: "Graylog Installation on Ubuntu Server 24.04"
date: 2026-03-12
categories: [graylog]
---

## Graylog needs 3 components

- MongoDB (metadata)
- OpenSearch (log storage)
- Graylog itself (UI + processing)

> You may install Graylog and its components (MongoDB, OpenSearch) as a standalone docker container.

While in production environments, they run graylog, MongoDB, OpenSearch inside K8s cluster.

- Kubernetes Cluster
  - graylog (Deployment)
  - opensearch (StatefulSet + PVC)
  - mongo (StatefulSet + PVC)
  - Fluent Bit DaemonSet (log agents on every node)
    - sends logs to graylog service

---

- Graylog itself does not automatically pull logs from Linux servers.
- Graylog receives log data via inputs (GELF, Syslog, Beats, etc.).
- Filebeat / Beats / Fluentd / rsyslog are called log shippers / agents. They read logs on your host and send them to Graylog.
- Graylog alone can only process data that is pushed to it.

Example scenarios:

1. System logs (Linux / Ubuntu) → use Filebeat → sends /var/log/syslog → Graylog input.
2. K8s cluster logs → use Fluent Bit / Fluentd → sends stdout/stderr logs of pods → Graylog.
3. Application logs → can push directly in GELF format via your app → Graylog input.

> Graylog is the collector/processor. You need an agent/shipper on the nodes that reads logs and pushes them in a format Graylog understands.

Mental Model:

| Component | Role |
| :---: | :---: |
| Graylog | UI + processing |
| OpenSearch | Log storage engine |
| MongoDB | Metadata |
| Fluent Bit | Log shippers |
| Kubernetes | Orchestrator |

---

| Component | Role |
| :---: | :---: |
| OpenSearch | The video files themselves |
| MongoDB | User accounts, playlists, comments |
| Graylog | The YouTube website |

- OpenSearch is used for storing and searching log data at scale.
- while MongoDB stores Graylog’s metadata such as users, dashboards, and configuration.

---

### DOCKER COMMANDS

```bash
docker ps # show docker containers
sudo docker-compose down # removes docker container from the docker-compose.yaml file
sudo docker-compose up -d # starts docker container
sudo docker logs graylog-opensearch --tail 50 #
sudo docker logs -f graylog # username and password
```

### Graylog Sidecar

- Instead of manually editing YAML files on every server, the Sidecar is a small wrapper you install on your Ubuntu nodes.
- It "talks" to Graylog, allowing you to manage the Filebeat configuration directly from the Graylog Web UI.
- It’s much more "SRE-like" because it centralizes management.

### How Log Shippers (like Fluent Bit) Work?

Think of a log shipper like a conveyor belt system for data. It follows a four-stage pipeline:

- Input: The shipper "tails" (reads) a source. This could be a log file (/var/log/syslog), a system metric, or even a stream of data from Docker.
- Parser: It turns raw text into structured data (JSON). For example, it breaks a single line of text into fields like timestamp, level, and message.
- Filter/Buffer: This is the "brain." It can add metadata (like "this log came from Server A"), remove sensitive info (like passwords), or drop useless logs. If Graylog is down, the shipper buffers the data in memory or on disk so nothing is lost.
- Output: It "ships" the structured data to its destination (Graylog) using a protocol like GELF (Graylog Extended Log Format).

### What metrics can you get with Fluent Bit?

While Fluent Bit is primarily a log shipper, it has built-in Input Plugins that act like a mini-Prometheus. You can collect:

- CPU Metrics: Percentage of usage, user vs. system time.
- Memory Metrics: Total, used, and free RAM.
- Disk usage: I/O wait times and read/write speeds.
- Network: Bandwidth usage and error rates per interface.
- Thermal: If the hardware supports it, you can sometimes see temperature (useful for the "CPU Throttling" conversation we had earlier!).
- SRE Tip: If you see "System Metrics" in Graylog, you are likely using the cpu and mem input plugins in your Fluent Bit config.

### How Fluent Bit send K8s cluster logs?

In a K8s cluster, Fluent Bit is usually deployed as a DaemonSet (one pod on every single node).

How it works in K8s:

- Log Discovery: It automatically finds the directory /var/log/containers/ where K8s stores all container logs.
- The Kubernetes Filter: This is the "magic" part. Fluent Bit talks to the Kubernetes API. It sees a log and says, "I know this log came from Container X. I will now ask the API for the Pod Name, the Namespace, and the Labels."
- Enrichment: It attaches all that K8s info to the log before sending it to Graylog.

Every good Graylog setup follows this exact loop:

> Raw logs → Search → Filter → Stream → Widget → Dashboard
