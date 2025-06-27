---
date: '2025-06-27T23:49:00+02:00'
title: 'Mettre à jour automatiquement un cluster K3s avec System Upgrade Controller'
description: "Des mises à jour automatiques pour son cluster K3s ?"
tags: ["Kubernetes", "k3s", "MCO", "System Upgrade Controller", "Cluster"]
---

# Mettre à jour automatiquement un cluster K3s avec System Upgrade Controller

---

## 🧠 Pourquoi automatiser les mises à jour K3s ?

Garder un cluster à jour, c’est *la base* : patches de sécu, corrections de bugs, perfs améliorées, etc.

Mais dans la vraie vie, surtout quand t’as un cluster auto-hébergé sur du matos hétérogène, bah t’oublies, tu repousses, ou t’as pas le temps. Résultat : t’es vite à la bourre sur les versions. Et c’est là que les emmerdes commencent.

Alors j’ai cherché une solution simple, stable, et un peu “Talos-like” pour automatiser ça.

---

## 🧩 Mon cluster pour [Retake.fr](https://retake.fr)

Petit rappel rapide du setup :

| Rôle       | Matériel                   | OS           |
|------------|----------------------------|--------------|
| 3 Masters  | Dell Wyse 5070 modifiés    | Armbian      |
| 3 Workers  | RockPi 64 (ARM64)          | Debian       |

- **K3s** est utilisé comme distro Kubernetes légère  
- **Objectif** : automatiser les upgrades K3s de manière propre, comme Talos/Kairos, mais en mode K3s.

> Le repo Git avec les manifests n’est pas encore public car je suis en train de chiffrer les YAML — j’en parlerai dans un autre article.

---

## ⚙️ Installation de System Upgrade Controller

[System Upgrade Controller](https://github.com/rancher/system-upgrade-controller), c’est l’outil proposé par Rancher pour piloter les mises à jour automatiques de K3s (ou RKE2).

### 📥 Installation :

```bash
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/download/v0.15.2/system-upgrade-controller.yaml
```

> Il n’y a pas plus simple.

### ✅ Vérification :

```bash
kubectl get pods -n system-upgrade
```

```
NAME                                         READY   STATUS    RESTARTS      AGE
system-upgrade-controller-78c958df6b-gx8j4   1/1     Running   1 (1m ago)    3m
```

> Avoir une installation aussi simple pour un outil comme ça, c’est juste incroyable.

## 🧠 Comment ça marche ?

**system-upgrade-controller** repose sur une ressource custom Plan qui va :

- Sélectionner des nœuds via leurs labels

- Les mettre à jour en série ou en parallèle (concurrency)

- Les cordonner / drainer proprement

- Utiliser l’image rancher/k3s-upgrade pour effectuer l’upgrade

- Tout ça via un ServiceAccount dédié

## 🧾 Plan de mise à jour basique

D’abord, on check la version actuelle des nœuds :

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

> P.S. Au moment où j’écris cet article, la version de K3s la plus récente est `v1.33.1+k3s1`.

Et on crée un plan de mise à jour :

```yaml
# ~/upgrades/basic_plan.yml
---
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: k3s-server
  namespace: system-upgrade
  labels:
    k3s-upgrade: server             # Met à jour les masters
spec:
  concurrency: 1                    # Upgrade un nœud à la fois
  version: v1.33.1+k3s1             # Dernière version de K3s
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
    k3s-upgrade: agent              # Met à jour les workers
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

Et on applique :

```bash
kubectl label node dell0 dell1 dell2 rock64-0 rock64-1 rock64-2 k3s-upgrade=enabled # Active les updates pour ce nœud
kubectl apply -f ~/upgrades/basic_plan.yml
```

## 🔄 Pourquoi ça change la donne

Avant, mettre à jour K3s sur 6 machines = galère manuelle + erreurs humaines possibles.

Maintenant, j’ai :

- des upgrades planifiables et progressifs

- un rollout contrôlé, nœud par nœud

- la possibilité de désactiver un nœud de l’upgrade via un simple label

- et même la possibilité de lancer des commandes côté nœud (apt update, reboot, etc.) pour une MCO plus poussée

## 📎 Liens utiles

- [System Upgrade Controller](https://github.com/rancher/system-upgrade-controller)

- [Exemples de plans](https://github.com/rancher/system-upgrade-controller/tree/master/examples)

- [Présentation de Kairos](https://kairos.io/)

- [Présentation de Talos](https://www.talos.dev/)

## 🧠 Terminologie

| Terme           | Définition rapide                                        |
| --------------- | -------------------------------------------------------- |
| **Plan**        | Ressource CRD qui décrit comment faire l’upgrade         |
| **MCO**         | Maintien en Condition Opérationnelle                     |
| **Cordon**      | Marquer un nœud comme non-schedulable                    |
| **Drain**       | Évacuer tous les pods d’un nœud                          |
| **concurrency** | Nombre de nœuds upgradés en parallèle                    |
| **prepare**     | Étape facultative avant l’upgrade, exécute des commandes |

## ✅ Conclusion

Maintenir un cluster K3s à jour n’a pas à être chiant.

Avec System Upgrade Controller, t’as un comportement automatique, des upgrades sûrs, et tu peux dormir tranquille (ou presque).

Je recommande ce setup à tous ceux qui ne peuvent / veulent pas passer sur Talos ou Kairos mais veulent quand même automatiser leur MCO.

> P.S. J'ai pas parlé du down time applicatif ou celui du cluster, simplement parce qu'il est différent en fonction de la load causé par les apps et parce que je suis sous la minute de downtime pour mes apps et sous les 2 min par machine.