---
layout: post
title: "What actually blocks a privileged pod? Pod Security Admission vs. Kyverno, tested"
date: 2026-10-25
---

I'm working through a hands-on Kubernetes security curriculum, and the first lab was a small hardening baseline on a local `kind` cluster. The question I wanted to answer with evidence: if someone with permission to create pods tries to deploy something dangerous, what actually stops them?

The full code is in [k8s-secure-baseline](https://github.com/nixhacks/k8s-secure-baseline), and CI re-runs the tests on every push.

## Two admission layers, one cluster

I used two controls that overlap on purpose:

- **Pod Security Admission (PSA)**, built into Kubernetes, enforcing the `restricted` profile in one namespace (`secure-apps`).
- **Kyverno**, a policy engine, enforcing four cluster-wide rules: no privileged containers, no hostPath volumes, images only from approved registries, and no `:latest` tag.

A script submits three bad pods (privileged, hostPath, `:latest`) and one hardened pod, and asserts the results. The outcome: five rejections and two acceptances.

## What the rejections look like

In the `default` namespace, which has no PSA enforcement, Kyverno stopped a privileged pod:

```
disallow-privileged-containers:
  no-privileged: 'validation error: Privileged containers are not allowed.
```

In `secure-apps`, PSA stopped the same pod, and it listed five violations at once: privileged, `allowPrivilegeEscalation`, unrestricted capabilities, `runAsNonRoot`, and the seccomp profile. That list taught me more than the rule name did. "Restricted" demands a lot more than "not privileged."

Why both layers? PSA is coarse and built in, and it works per namespace. Kyverno lets me write cluster-wide rules in my own words with readable error messages, and cover things PSA doesn't, like registries and tags. Neither replaces the other.

## The finding I didn't expect

I applied a default-deny NetworkPolicy to `secure-apps`, then tested it. First, a control: from a namespace with no policy, a pod reached the API server. Then the same test inside `secure-apps`. It also reached the API server.

The policies were correct and present. The default networking in my kind cluster simply didn't enforce them. So I can't claim network isolation from this lab. A NetworkPolicy object is a request, and something has to enforce it. Next I'll swap in Cilium and add a test that fails if egress isn't blocked.

## Two CI races

The tests passed on my Mac and failed in CI, twice, for the same reason: on a fresh cluster, things happen in a different order.

1. Right after installing Kyverno, applying policies failed with `connection refused`. The pods were running, but the webhook wasn't accepting calls yet. The fix: wait for the deployments, then retry the apply.
2. Every new namespace gets a `default` ServiceAccount, created a moment after the namespace. My manifest tried to create it at the same moment and hit `AlreadyExists`. The fix: wait for the account, then patch it, not declare it.

Locally, the namespace already existed from earlier runs, so neither bug could show up. A fresh-cluster CI run caught both.

## kube-bench on kind

kube-bench scored 16 PASS, 8 FAIL, 35 WARN. I treat the numbers as indicative, because kind isn't a hardened distribution. Some failures, like kubelet file permissions, are not applicable to a container-based node. I closed the default ServiceAccount token findings in the namespaces I own. They still fail in system namespaces, which need their tokens, and I'd rather leave them failing than break Kyverno to improve a score.

My own policies also blocked kube-bench at first. It needs a hostPath volume and an image from docker.io. I added a documented exception for that one namespace. It was a small reminder that controls work even on the tools you trust.

## What's next

Cilium for real network enforcement, runtime detection, and signed-image verification. I'll write up each one the same way: what I built, how I tested it, and where it fell short.
