---
title: Running distributed pytorch training on Kubernetes and checkpointing to HDFS
date: 2023-01-19 12:47:38
tags: 
 - Machine Learning
 - PyTorch
 - Distributed
---

Recently I'm running some experiments on our company's cluster. We need to submit our jobs to Kubernetes cluster, and our checkpoints need to go to HDFS.