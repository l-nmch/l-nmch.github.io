---
date: '2025-06-27T23:49:00+02:00'
title: 'Automatically Updating a K3s Cluster with System Upgrade Controller'
description: "Automatic updates for your K3s cluster?"
tags: ["Kubernetes", "k3s", "Ops", "System Upgrade Controller", "DevOps", "Cluster"]
---

---

## 🧠 Why automate K3s updates?

Keeping a cluster up to date is *the bare minimum*: security patches, bug fixes, better performance, all of it.

But in real life, especially when you're running a self-hosted cluster on a pile of heterogeneous hardware, you forget, you put it off, or you just don't have the time for it. And before you know it you're way behind on versions, which is exactly when the trouble starts.

So I went looking for something simple, stable, and a bit "Talos-like" to automate this.

---

## 🧩 My cluster for [Retake.fr](https://retake.fr)

Quick reminder of the setup:

| Role       | Hardware                   | OS           |
|------------|----------------------------|--------------|
| 3 Masters  | Modified Dell Wyse 5070    | Armbian      |
| 3 Workers  | RockPi 64 (ARM64)          | Debian       |

- **K3s** is used as the lightweight Kubernetes distro
- **Goal**: automate K3s upgrades cleanly, the way Talos or Kairos would, but K3s-style.

> The Git repo with the manifests isn't public yet, because I'm in the middle of encrypting the YAML — I'll get into that in another post.

---

## ⚙️ Installing System Upgrade Controller

[System Upgrade Controller](https://github.com/rancher/system-upgrade-controller) is the tool Rancher provides for driving automatic K3s (or RKE2) upgrades.

### 📥 Installation:

```bash
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/download/v0.15.2/system-upgrade-controller.yaml
```

> It doesn't get any simpler than that.

### ✅ Check it worked:

```bash
kubectl get pods -n system-upgrade
```

```
NAME                                         READY   STATUS    RESTARTS      AGE
system-upgrade-controller-78c958df6b-gx8j4   1/1     Running   1 (1m ago)    3m
```

> Getting an install this painless for a tool like this still catches me off guard.

## 🧠 How does it work?

**system-upgrade-controller** relies on a custom Plan resource that will:

- Select nodes via their labels

- Update them serially or in parallel (concurrency)

- Cordon / drain them cleanly

- Use the rancher/k3s-upgrade image to perform the upgrade

- Do all of this through a dedicated ServiceAccount

## 🧾 A basic upgrade plan

First, check the current version on the nodes:

```bash
kubectl get nodes
```

```
NAME       STATUS   ROLES                       AGE    VERSION
dell0      Ready    control-plane,etcd,master   382d   v1.32.4+k3s1
dell1      Ready    control-plane,etcd,master   382d   v1.32.4+k3s1
dell2      Ready    control-plane,etcd,master   382d   v1.32.4+k3s1
rock64-0   Ready    <none>                      382d   v1.32.4+k3s1
rock64-1   Ready    <none>                      382d   v1.32.4+k3s1
rock64-2   Ready    <none>                      382d   v1.32.4+k3s1
```

> P.S. As I'm writing this, the latest K3s version is `v1.33.1+k3s1`.

Then we create an upgrade plan:

```yaml
# ~/upgrades/basic_plan.yml
---
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: k3s-server
  namespace: system-upgrade
  labels:
    k3s-upgrade: server             # Updates the masters
spec:
  concurrency: 1                    # Upgrade one node at a time
  version: v1.33.1+k3s1             # Latest K3s version
  nodeSelector:
    matchExpressions:
      - {key: k3s-upgrade, operator: Exists}
      - {key: k3s-upgrade, operator: NotIn, values: ["disabled", "false"]}
      - {key: node-role.kubernetes.io/control-plane, operator: Exists}
  serviceAccountName: system-upgrade
  cordon: true
  upgrade:
    image: rancher/k3s-upgrade
---
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: k3s-agent
  namespace: system-upgrade
  labels:
    k3s-upgrade: agent              # Updates the workers
spec:
  concurrency: 1
  version: v1.33.1+k3s1
  nodeSelector:
    matchExpressions:
      - {key: k3s-upgrade, operator: Exists}
      - {key: k3s-upgrade, operator: NotIn, values: ["disabled", "false"]}
      - {key: node-role.kubernetes.io/control-plane, operator: DoesNotExist}
  serviceAccountName: system-upgrade
  prepare:
    image: rancher/k3s-upgrade
    args: ["prepare", "k3s-server"]
  drain:
    force: true
    skipWaitForDeleteTimeout: 60
  upgrade:
    image: rancher/k3s-upgrade
```

And we apply it:

```bash
kubectl label node dell0 dell1 dell2 rock64-0 rock64-1 rock64-2 k3s-upgrade=enabled # Enables updates for this node
kubectl apply -f ~/upgrades/basic_plan.yml
```

## 🔄 Why this changes everything

Before, updating K3s across 6 machines meant manual grinding through it with human error waiting to happen at every step.

Now, I've got:

- upgrades that are schedulable and staged

- a controlled rollout, node by node

- the ability to pull a node out of the upgrade with a simple label

- and even the ability to run node-side commands (apt update, reboot, etc.) for more thorough ops work

## 📎 Useful links

- [System Upgrade Controller](https://github.com/rancher/system-upgrade-controller)

- [Plan examples](https://github.com/rancher/system-upgrade-controller/tree/master/examples)

- [Kairos overview](https://kairos.io/)

- [Talos overview](https://www.talos.dev/)

## 🧠 Terminology

| Term            | Quick definition                                          |
| --------------- | -------------------------------------------------------- |
| **Plan**        | CRD resource describing how the upgrade should happen     |
| **Ops**         | Operational maintenance — keeping the system running smoothly (MCO, in the original French: *Maintien en Condition Opérationnelle*) |
| **Cordon**      | Marking a node as unschedulable                           |
| **Drain**       | Evicting all pods from a node                             |
| **concurrency** | Number of nodes upgraded in parallel                      |
| **prepare**     | Optional step before the upgrade, runs commands           |

## ✅ Conclusion

Keeping a K3s cluster up to date doesn't have to be a chore.

With System Upgrade Controller, you get automatic behavior, safe upgrades, and you can actually sleep soundly at night (or close enough).

I'd recommend this setup to anyone who can't or doesn't want to move to Talos or Kairos, but still wants their ops maintenance automated.

> P.S. I didn't get into application or cluster downtime here, simply because it varies depending on the load the apps generate, and because I stay under a minute of downtime for my apps and under 2 minutes per machine.
