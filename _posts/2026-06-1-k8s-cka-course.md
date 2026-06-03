---
layout: post
title: "The Complete K8s & CKA Certification Course"
date: 2026-06-1
categories: [k8s]
---

## Introduction

This course is designed by ***devopscube.com***. The following is my personal notes from the course material. All materials, images and codes used in this note is property of devopscube.com.

## Kubernetes Cluster Architecture

K8s is a distributed system. Meaning its components are running across multiple physical or virtual servers in the network.

A K8s cluster mainly includes:

- Control Plane components
- Worker Node components
- Add-on components

![K8s Architecture]({{ site.baseurl }}/assets/img/k8s-course/k8s-architecture.gif)

## Kubeadm Cluster Setup

**kubeadm** is a toolbox that helps you bootstrap a minimum viable Kubernetes cluster. It performs the actions necessary to get a cluster up and running
