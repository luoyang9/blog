---
layout: post
title: Catching Human Errors in Kubernetes Manifests
---

Kubernetes is an open-source tool that allows you to deploy and manage containers in clusters. It's great for companies that want to deploy cloud applications across multiple environments. However, it requires a lot of configuration files that define the resources it manages (deployments, services, ConfigMaps, etc). All these files can get really complicated really fast, and a couple of simple human errors in the configuration can potentially lead to hours of production downtime if no one catches it. 

Validating your Kubernetes manifests using custom rules that you can set would reduce or eliminate the amount of errors that make it to production.

<!--more-->

# Background

This summer, I had the opportunity to intern on the core infrastructure team at Wish, an e-commerce platform that connects manufacturers and small businesses directly to hundreds of millions of customers. One of the team's big tasks was to migrate Wish infrastructure to Kubernetes. This involved creating configuration files for 

# Proposal

# Design & Planning

# Implementation