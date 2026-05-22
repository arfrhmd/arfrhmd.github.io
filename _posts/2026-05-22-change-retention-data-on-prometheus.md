---
layout: post
title: Ubah retensi data penyimpanan di Prometheus
date: 2026-05-22 08:29:00
description: Umumnya Prometheus menyimpan data selama 15 hari, karena ada kebutuhan yang mengharuskan penyimpanan data prometheus lebih lama
tags: prometheus docker
categories: monitoring
---

Umumnya Prometheus menyimpan data selama 15 hari, karena saya ada kebutuhan yang mengharuskan penyimpanan data prometheus lebih lama. Saya cukup tambahkan command berikut:

```bash
--storage.tsdb.retention.time=30d
```

Command ini berguna untuk mengubah retensi data penyimpanan menjadi 30 hari. Kalau diterapkan di Docker menjadi seperti ini:

```Dockerfile
services:
  prometheus:
    image: prom/prometheus:v2.37.0
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
```
