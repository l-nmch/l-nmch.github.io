---
date: "2026-08-11T00:22:00+02:00"
title: "De l'IA sur des cartes sorties de nulle part"
description: "Héberger des modèles d'IA générative sur des GPU étranges"
tags: ["GenAI", "Z440", "NVIDIA", "AI", "GPU", "Inference", "LLM", "llama.cpp"]
---

---

## Dans quel but ?

Depuis que je m'intéresse à l'IA j'aime bien faire mumuse avec des GPU étranges. Après avoir fait [Un setup IA moins cher que Mario Kart World](/posts/cheap-ai-server/), je me suis demandé si on pouvait héberger des modèles d'IA performants avec des GPU vraiment pas chers. Et spoiler, oui on peut. Mais avec quelques points noirs.

## Le setup

Le setup reste plus ou moins le même que pour l'article précédent. Pour rappel:

- Alimentation HP 700W  
- [Intel Xeon E5-1620 v3 @ 3.50GHz](https://www.intel.fr/content/www/fr/fr/products/sku/82763/intel-xeon-processor-e51620-v3-10m-cache-3-50-ghz/specifications.html)
- 32 Go RAM DDR4 ECC 2133MHz
- [SSD Samsung EVO 250 Go](https://www.samsung.com/fr/memory-storage/sata-ssd/870-evo-250gb-sata-3-2-5-ssd-mz-77e250b-eu/)
- [Quadro K2200 4 Go](https://www.nvidia.com/content/dam/en-zz/Solutions/design-visualization/documents/75509_DS_NV_Quadro_K2200_US_NV_HR.pdf)
- [GTX 1050 Ti 4 Go](https://www.gigabyte.com/fr/Graphics-Card/GV-N105TG1-GAMING-4GD)

Mais on va y ajouter 2 cartes un peu suspectes :

- [NVIDIA Tesla P4](https://www.techpowerup.com/gpu-specs/tesla-p4.c2879): 90€
- [NVIDIA Tesla T10](https://www.techpowerup.com/gpu-specs/tesla-t10-16-gb.c4036): 300€

La [NVIDIA Tesla P4](https://www.techpowerup.com/gpu-specs/tesla-p4.c2879) est une [NVIDIA GeForce GTX 1080](https://www.techpowerup.com/gpu-specs/geforce-gtx-1080.c2839) avec un form-factor réduit (HHHL), un cooling passif, et qui ne consomme que 75W, donc pas d'alimentation externe !

Elle date de 2016 et propose **8 Go GDDR5** avec un bus mémoire d'une vitesse de **192.3 GB/s** et **5.704 TFLOPS** en FP32. Selon mes sources (moi et mes multiples personnalités), ce GPU était utilisé principalement à des fins de VDI (Virtual Desktop Infrastructure) et de transcodage.

![Techpowerup Tesla P4](/img/tpu-nvidia-p4.png)

La [NVIDIA Tesla T10](https://www.techpowerup.com/gpu-specs/tesla-t10-16-gb.c4036) elle est un peu plus spéciale. Bien que comparable à une [NVIDIA GeForce RTX 2080 Ti](https://www.techpowerup.com/gpu-specs/geforce-rtx-2080-ti.c3305), elle dispose de moins de **coeurs**, moins de **TMUs**, moins de **ROPS** et d'un bus mémoire plus petit. Elle a un form-factor single-slot PCIe (FHFL), un cooling passif, et consomme 150W via un port externe 8-PIN PCIe.

Elle date de 2020 et propose **16 Go GDDR6** avec un bus mémoire d'une vitesse de **403.2 GB/s** et **9.999 TFLOPS** en FP32. Après plusieurs heures de recherches, j'ai trouvé que cette carte était utilisée par *Nvidia* pour *GeForce Now*. Elle servait donc pour du cloud gaming ! 

> Les cartes Nvidia sous architecture Turing (RTX 20xx) sont les premières cartes à proposer des *Tensor Cores*, des coeurs dédiés à accélérer des calculs spécifiques à l'IA. C'est en grande partie ce qui a permis l'arrivée du [DLSS](https://www.nvidia.com/fr-fr/geforce/technologies/dlss/). 

![Techpowerup Tesla T10](/img/tpu-nvidia-t10.png)

### Pourquoi ces cartes sont étranges ?

Les cartes Nvidia *GTX*, *RTX*, ou *Quadro* sont relativement connues du public. Elles sont utilisées pour du gaming, du rendu vidéo, de la CAO et d'autres tâches communes. Mais les cartes *Tesla* elles sont dédiées à des tâches bien plus spécifiques, notamment de la virtualisation via des solutions comme [vGPU](https://docs.nvidia.com/vgpu/index.html) ou [MIG](https://www.nvidia.com/fr-fr/technologies/multi-instance-gpu/), pour être utilisées dans plusieurs machines virtuelles en même temps (ce qui explique que la *Tesla T10* ait été utilisée pour *GeForce Now*). Elles sont aussi souvent dédiées à des tâches de *Machine Learning*. Elles sont basées sur les mêmes puces utilisées pour les cartes grand public mais sont destinées à finir dans des serveurs (d'où le cooling passif).

### Le refroidissement

Maintenant qu'on connaît un peu mieux ces cartes, il faut les refroidir. La workstation *HP Z440* n'a jamais été faite pour accueillir ce type de cartes. Il va donc falloir se démerder autrement parce qu'une carte sans cooling qui monte à **97°C** c'est vraiment pas une bonne idée. On doit donc passer à du cooling actif.

Et qui dit cooling actif dit ventilateurs !

> ⚠️ Les modifications qui ont été faites ici ne sont pas recommandées et sont risquées. Jouer avec de l'électricité même à 12V peut vous coûter la vie donc évitez de jouer aux cons. Je n'incite personne à reproduire les modifications suivantes et décline toute responsabilité de perte de garantie, de casse matérielle, ou de dégât physique.

Pour alimenter nos ventilateurs on a besoin d'une source d'électricité, et quoi de mieux qu'un port d'alimentation *SATA* pas utilisé ? Ce dernier peut nous fournir du **3.3V à 1.5A**, **5V à 4.5A**, **12V à 4.5A** ce qui est parfait pour alimenter nos ventilos.

> Pour plus d'information: https://tadeubento.com/2025/sata-connector-power-rating-and-hard-drives/

J'ai donc bidouillé un adaptateur *SATA* -> Ventilateur *3 Pin* branché sur le rail **12V** (Bon ok il n'y a que 2 pins sur le port du ventilateur d'utilisées mais le 3ème ne sert qu'au PWM / réguler la vitesse ce dont on n'a pas besoin ici.)

![SATA TO 3 PIN](/img/cooling-sata-adapter.png)

Ensuite il faut refroidir notre *Tesla P4*. J'ai donc imprimé un shroud en 3D pour accepter un ventilateur **30mm** alimenté en **12V à 0.1A** et pousser de l'air directement sur le bloc de cuivre du GPU. C'est pas la meilleure façon de refroidir cette carte mais j'avais la flemme de designer quelque chose d'autre et j'avais pas d'autre ventilateur assez petit.

![Tesla P4 Cooling](/img/cooling-p4.png)

Maintenant pour la *Tesla T10* on sort l'artillerie lourde. J'ai imprimé un autre shroud en 3D pour accepter un ventilateur **30mm** alimenté en **12V à 0.40A** mais cette fois ci en mode *blower*. On respecte donc le flux d'air de la carte.

![Tesla T10 Cooling](/img/cooling-t10.png)

J'utilise ensuite un splitter pour ventilateur pour répartir les **12V** sur les 2 cartes. En additionnant les **0.1A** de la *Tesla P4* au **0.40A** de la *Tesla T10* on arrive à **0.50A** ce qui reste dans la norme du **12V à 4.5A** de notre port d'alimentation *SATA* et on arrive à 6W de cooling pour nos cartes. Ça paraît peu, et pourtant c'est amplement suffisant pour notre setup.

### Le software

D'un point de vue hardware on est tout bon ! 4 GPUs, tous refroidis, et de quoi installer un OS.

Comme d'habitude quand il s'agit d'avoir un bon support driver côté GPU on doit utiliser [Ubuntu](https://ubuntu.com).

Pour l'installation des drivers rien de plus simple:

```bash
ubuntu-drivers list --gpgpu
```

Cette commande nous permet de déterminer automatiquement la version des drivers Nvidia la plus adaptée à nos cartes:

```
nvidia-driver-580-server, (kernel modules provided by linux-modules-nvidia-580-server-generic)
nvidia-driver-580, (kernel modules provided by linux-modules-nvidia-580-generic)
```

Il nous suffit juste de les installer:

```bash
apt-get update
apt-get install -y nvidia-driver-580-server
reboot
```

On vérifie que nos cartes sont bien vues par l'OS:

```bash
nvidia-smi
```

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.173.02             Driver Version: 580.173.02     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Tesla T10                      Off |   00000000:02:00.0 Off |                    0 |
| N/A   46C    P0             40W /  150W |    5121MiB /  15360MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   1  Tesla P4                       Off |   00000000:04:00.0 Off |                    0 |
| N/A   40C    P0             23W /   75W |       0MiB /   7680MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   2  Quadro K2200                   Off |   00000000:06:00.0 Off |                  N/A |
| 34%   47C    P0              2W /   39W |       0MiB /   4096MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA GeForce GTX 1050 Ti     Off |   00000000:09:00.0 Off |                  N/A |
|  0%   44C    P0            N/A  /  120W |       0MiB /   4096MiB |      2%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
```

Et c'est tout bon, on peut utiliser nos cartes !

On va maintenant installer [nvtop](https://github.com/Syllo/nvtop), un *htop* pour GPU.

```bash
apt-get install -y nvtop
```

## Héberger un modèle

Bon, j'ai menti. Nos cartes sont pas vraiment prêtes à être utilisées. Les architectures *Maxwell* (Quadro K2200), *Pascal* (Tesla P4 & 1050TI), et *Turing* (Tesla T10) ne sont plus vraiment supportées par beaucoup de logiciels, la carte la plus récente ici a 6 ans !

On va donc devoir rebuild certains d'entre eux, à commencer par [llama.cpp](https://llama.app/). C'est un moteur d'inférence très puissant qui nous permet d'héberger des modèles d'IA générative sur tout type de CPU et GPU, et ça de manière plutôt simple. C'est ce qui est utilisé par [Ollama](https://ollama.com) en arrière plan pour vous faciliter la vie.

### Build llama.cpp pour nos cartes

Build un software à la main peut souvent faire peur, pas ici. La compilation a été grandement simplifiée et peut se faire en moins de 5 commandes:

1. Installer les pré-requis:

Avant de compiler *llama.cpp* on a besoin de 2-3 paquets

```bash
apt-get install -y curl cmake git nvidia-cuda-toolkit
```

2. Clonage du repository:

```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp
```

3. Compilation:

```bash
cmake -B build -DGGML_CUDA=ON -DLLAMA_CURL=ON -DCMAKE_CUDA_ARCHITECTURES=<CUDA SM VERSION> # 50 pour Maxwell | 61 pour Pascal | 75 pour Turing
cmake --build build -j$(nproc) # Utilise tous les coeurs dispo pour compiler llama.cpp
```

Après ~10min on retrouve tous les binaires nécessaires pour héberger des modèles d'IA dans `~/llama.cpp/build/bin`

### Télécharger un modèle

Parce qu'utiliser *llama.cpp* sans modèle serait un peu con, on va en télécharger un.

En se rendant sur [HuggingFace](https://hf.co) on trouve plusieurs MILLIONS de modèles. Alors comment faire un choix ?

*Llama.cpp* ne supporte qu'un format de fichier spécifique, le `.gguf`. On peut donc limiter notre choix à ce format de fichier qu'on peut utiliser comme filtre. Ensuite il faut un modèle qui tienne dans notre setup. Et là je vais en dédier un article parce que ça paraît simple mais c'est un bordel sans nom, genre vraiment.

Ici j'ai choisi ce modèle :

- [unsloth/gemma-4-E4B-it-qat-GGUF](https://huggingface.co/unsloth/gemma-4-E4B-it-qat-GGUF)

Assez petit pour tenir confortablement dans la vRAM de nos deux cartes même sans optimisation, assez costaud pour ne pas répondre n'importe quoi. Bref, la taille parfaite pour ce qu'on veut en faire ici : essayer de faire dire de la merde à un modèle.

Notre modèle est split en plusieurs fichiers:

- `nom_du_modèle_lettres_bizarres.gguf`: Le modèle en lui-même
- `mmproj-FP16.gguf`: Ses fonctions de multi-modalités (compréhension d'image / audio)

## Benchmarking

### Méthodologie

On va donc installer un super outil qui va nous permettre de lancer des benchmarks: [llama-benchy](https://github.com/eugr/llama-benchy)

```bash
snap install astral-uv
uvx llama-benchy
```

```
usage: llama-benchy [-h] [--version] --base-url BASE_URL [--api-key API_KEY] [--model MODEL] [--served-model-name SERVED_MODEL_NAME] [--tokenizer TOKENIZER] [--pp PP [PP ...]]
                    [--tg TG [TG ...]] [--exact-tg] [--depth DEPTH [DEPTH ...]] [--runs RUNS] [--no-cache] [--post-run-cmd POST_RUN_CMD] [--book-url BOOK_URL]
                    [--latency-mode {api,generation,none}] [--no-warmup] [--skip-coherence] [--adapt-prompt] [--no-adapt-prompt] [--enable-prefix-caching]
                    [--concurrency CONCURRENCY [CONCURRENCY ...]] [--save-result SAVE_RESULT] [--format {md,json,csv}] [--save-total-throughput-timeseries]
                    [--save-all-throughput-timeseries] [--exit-on-first-fail] [--no-results-on-fail] [--extra-body EXTRA_BODY] [--emit-progress PATH]
llama-benchy: error: the following arguments are required: --base-url
```

### Tesla P4

#### Commande

*Llama.cpp* propose plusieurs binaires pour lancer des modèles, celui qui nous intéresse ici c'est `llama-server`. Il sert à héberger un modèle sous le standard d'API [OpenAI](https://github.com/openai/openai-openapi). On l'utilise car il nous propose une API et une interface graphique simple à utiliser.

Allez, on en parle depuis un moment, lançons un modèle maintenant:

```bash
CUDA_VISIBLE_DEVICES=1 ~/pascal/llama.cpp/build/bin/llama-server --host 0.0.0.0 --port 8080 -m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf
```

> Ici le modèle tourne sur la *Tesla P4*

Bon on va quand même expliquer un peu ce merdier parce que là on dirait qu'un mec a vomi sur mon clavier.

- `CUDA_VISIBLE_DEVICES=1`: Sert à "sélectionner" le bon GPU (Utiliser plusieurs architectures et plusieurs build de *llama.cpp* ça peut très rapidement foutre la merde, donc on choisit bien notre carte.)
- `~/pascal/llama.cpp/build/bin/llama-server`: Localisation de notre binaire `llama-server` sur notre machine
- `--host 0.0.0.0 --port 8080`: J'ai vraiment besoin de l'expliquer ?
- `-m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf`: Localisation de notre modèle
- `--mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf`: Localisation de ses fonctions de multi-modalités

C'est déjà un peu plus clair !

Maintenant on va sur `http://ip-de-notre-z440:8080/` et on discute avec notre modèle:

![llama.cpp chat P4](/img/llama.cpp-chat-p4.png)

Ok c'est cool, mais est-ce que c'est optimisé ? Pour rappel l'idée est de tirer des bonnes performances de ces cartes. Passons au vrai bench.

#### Bench

Parfait ! Maintenant on test notre setup actuel:

```bash
uvx llama-benchy --base-url http://ip-de-notre-z440:8080/v1 --served-model-name /root/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --model google/gemma4-E4B-it --pp 512 2048 4096 --tg 512 2048 4096
```

- `--base-url`: L'url de *llama.cpp*
- `--served-model-name`: Le "nom" du modèle hébergé par *llama.cpp*
- `--model`: Le "nom" du modèle à chercher sur [HuggingFace](https://hf.co) pour les tests
- `--pp`: Les différentes tailles de prompt (en token)
- `--tg`: Les différentes tailles de réponses (en token)

`llama-benchy` ne va pas juste tester chaque valeur de `--pp` avec la valeur de `--tg` du même rang. Il teste en fait **toutes les combinaisons possibles** entre nos tailles de prompt et nos tailles de réponse : `pp512/tg512`, `pp512/tg2048`, `pp512/tg4096`, `pp2048/tg512`, et ainsi de suite jusqu'à `pp4096/tg4096`. Avec 3 valeurs de chaque côté, ça nous fait donc 9 combinaisons de test. C'est voulu : un prompt court suivi d'une longue réponse (résumé, génération de code...) et un prompt long suivi d'une réponse courte (RAG, classification...) ne sollicitent pas du tout le GPU de la même façon, donc autant les mesurer toutes.

Et pour chacune de ces 9 combinaisons, le test est répété 3 fois (c'est la valeur par défaut du paramètre `--runs`), en plus d'un premier essai à vide (le *warmup*) qui est jeté et ne compte pas dans les résultats. Le but est d'éviter qu'un résultat foireux (montée en fréquence du GPU pas terminée, un process qui traîne en arrière-plan, etc.) ne fausse la mesure : `llama-benchy` nous donne donc une moyenne ± un écart-type calculés sur ces 3 essais, plutôt qu'un chiffre isolé auquel on ne peut pas vraiment faire confiance.

Pendant que le test tourne on va regarder à quoi ressemble notre `nvtop`:

![nvtop p4](/img/nvtop-p4.png)

Après ~20min notre bench est fini:

```
| model                |   test |            t/s |     peak t/s |        ttfr (ms) |     est_ppt (ms) |    e2e_ttft (ms) |
|:---------------------|-------:|---------------:|-------------:|-----------------:|-----------------:|-----------------:|
| google/gemma4-E4B-it |  pp512 | 626.26 ± 16.91 |              |   757.76 ± 33.61 |   755.08 ± 33.61 |   842.09 ± 33.99 |
| google/gemma4-E4B-it |  tg512 |   36.12 ± 0.23 | 36.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it |  pp512 | 605.81 ± 19.03 |              |   716.45 ± 13.87 |   713.77 ± 13.87 |   802.20 ± 13.58 |
| google/gemma4-E4B-it | tg2048 |   35.69 ± 0.06 | 36.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it |  pp512 |  585.52 ± 8.36 |              |   814.39 ± 11.67 |   811.70 ± 11.67 |   900.43 ± 10.46 |
| google/gemma4-E4B-it | tg4096 |   35.92 ± 0.22 | 36.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  598.03 ± 8.90 |              | 3118.10 ± 153.00 | 3115.42 ± 153.00 | 3208.05 ± 152.01 |
| google/gemma4-E4B-it |  tg512 |   34.46 ± 0.05 | 35.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  589.45 ± 8.52 |              |  3153.12 ± 37.64 |  3150.44 ± 37.64 |  3242.11 ± 37.53 |
| google/gemma4-E4B-it | tg2048 |   34.21 ± 0.07 | 35.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  584.55 ± 5.87 |              |  3314.65 ± 59.98 |  3311.97 ± 59.98 |  3403.88 ± 60.48 |
| google/gemma4-E4B-it | tg4096 |   34.23 ± 0.04 | 35.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  539.74 ± 1.34 |              |  6937.52 ± 64.75 |  6934.84 ± 64.75 |  7028.14 ± 64.68 |
| google/gemma4-E4B-it |  tg512 |   33.93 ± 0.04 | 34.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  537.94 ± 1.73 |              | 6871.97 ± 104.42 | 6869.28 ± 104.42 | 6963.09 ± 103.85 |
| google/gemma4-E4B-it | tg2048 |   33.79 ± 0.26 | 34.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  536.15 ± 2.54 |              | 6829.99 ± 249.02 | 6827.31 ± 249.02 | 6920.72 ± 248.83 |
| google/gemma4-E4B-it | tg4096 |   33.79 ± 0.09 | 34.00 ± 0.00 |                  |
```

Ce qu'on retiens:

- **~35 t/s AVG** en Text Generation
- **~537 t/s AVG** en Prompt Processing
- **~3701,19 ms AVG** en Time To First Token
- **~6.1 Gi** de vRAM utilisé

Pour un modèle de cette taille sur une carte à moins de 100€, bah c'est pas mauvais ! Mais on va quand même essayer d'optimiser ça.

#### Optimisation

`llama-server` propose tout un tas d'options pour aller chercher plus de perf sans changer une seule ligne de matériel. L'idée derrière tout ça se résume en deux axes :

- **Consommer moins de vRAM au repos.** Le modèle occupe une taille fixe, mais le KV-cache lui grossit avec le contexte — chaque token de conversation ajoute sa part. Moins on en consomme à taille de contexte égale, plus il nous en reste pour justement... augmenter la taille max du contexte.
- **Aller plus vite à contexte égal**, sans rien sacrifier côté qualité de réponse.

**Flash Attention** — `-fa on` active un kernel d'attention optimisé qui calcule les scores d'attention sans jamais matérialiser la matrice complète en mémoire. Résultat : moins de vRAM utilisée par le calcul d'attention lui-même, et généralement un peu plus de vitesse, surtout à mesure que le contexte grandit. C'est aussi un prérequis technique pour la suite : llama.cpp exige Flash Attention active pour pouvoir quantifier le KV-cache.

**Quantification du KV-cache** — `-ctk q4_0 -ctv q4_0` quantifie les *Keys* et les *Values* du KV-cache de FP16 vers de l'INT4. Comme le KV-cache est justement la partie qui grossit avec le contexte, le quantifier a un effet direct et immédiat sur la vRAM disponible : à contexte égal, on en consomme beaucoup moins, ce qui nous permet en retour d'augmenter la taille max du contexte sans faire exploser la vRAM.

**Une fenêtre de contexte plus grande** — Avec toute la vRAM récupérée grâce aux deux points précédents, `-c 65535` nous permet de monter la taille de contexte max supportée par le serveur. Sans ces optimisations, ce genre de valeur ferait tout simplement crasher `llama-server` par manque de vRAM.

**Batching** — Il reste aussi `-b` et `-ub`, qui contrôlent la taille des batchs utilisés pour traiter les prompts (le `-ub` en particulier découpe le traitement du prompt en plus petits morceaux). Les augmenter permet généralement de mieux saturer le GPU pendant la phase de prompt processing, au prix d'un peu plus de vRAM — à ajuster selon ce qu'il vous reste de libre une fois les optimisations au-dessus en place.

Voilà pour la théorie, passons à la pratique :

```bash
CUDA_VISIBLE_DEVICES=1 ~/pascal/llama.cpp/build/bin/llama-server --host 0.0.0.0 --port 8080 -m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf -fa on -ctk q4_0 -ctv q4_0 -c 65535
```

#### Bench

Et on relance exactement le même bench qu'avant, sur la même carte, pour comparer à armes égales :

```bash
uvx llama-benchy --base-url http://ip-de-notre-z440:8080/v1 --served-model-name /root/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --model google/gemma4-E4B-it --pp 512 2048 4096 --tg 512 2048 4096
```

```
| model                |   test |            t/s |     peak t/s |        ttfr(ms) |     est_ppt (ms) |    e2e_ttft (ms) |
|:---------------------|-------:|---------------:|-------------:|-----------------:|-----------------:|-----------------:|
| google/gemma4-E4B-it |  pp512 |  632.73 ± 2.66 |              |   740.38 ± 34.31 |   737.04 ± 34.31 |   836.29 ± 34.28 |
| google/gemma4-E4B-it |  tg512 |   31.88 ± 0.10 | 32.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it |  pp512 | 613.01 ± 14.06 |              |    785.73 ± 5.08 |    782.39 ± 5.08 |    883.85 ± 5.24 |
| google/gemma4-E4B-it | tg2048 |   30.77 ± 0.22 | 31.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it |  pp512 | 592.09 ± 20.13 |              |   774.32 ± 44.09 |   770.98 ± 44.09 |   872.55 ± 43.08 |
| google/gemma4-E4B-it | tg4096 |   31.33 ± 0.40 | 31.67 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  593.44 ± 3.50 |              |  3100.50 ± 11.47 |  3097.16 ± 11.47 |  3206.30 ± 10.84 |
| google/gemma4-E4B-it |  tg512 |   29.07 ± 0.02 | 30.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  594.06 ± 3.14 |              | 3079.92 ± 152.22 | 3076.58 ± 152.22 | 3185.74 ± 153.86 |
| google/gemma4-E4B-it | tg2048 |   29.05 ± 0.17 | 29.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  586.12 ± 2.48 |              |  3169.53 ± 42.82 |  3166.19 ± 42.82 |  3275.80 ± 41.64 |
| google/gemma4-E4B-it | tg4096 |   29.12 ± 0.16 | 29.67 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  539.86 ± 3.08 |              |  6959.97 ± 90.95 |  6956.63 ± 90.95 |  7067.44 ± 91.00 |
| google/gemma4-E4B-it |  tg512 |   28.37 ± 0.02 | 29.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  540.97 ± 1.74 |              | 6869.72 ± 181.18 | 6866.38 ± 181.18 | 6976.98 ± 182.31 |
| google/gemma4-E4B-it | tg2048 |   28.04 ± 0.03 | 29.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  540.54 ± 2.77 |              | 6954.39 ± 109.84 | 6951.05 ± 109.84 | 7062.22 ± 109.84 |
| google/gemma4-E4B-it | tg4096 |   28.02 ± 0.14 | 28.33 ± 0.47 |                  |                  |                  |
```

Ce qu'on retiens, en comparant au baseline *P4* :

- **~29,52 t/s AVG** en Text Generation, contre ~35 t/s en baseline — donc **~16% plus lent**
- **~581,42 t/s AVG** en Prompt Processing, contre ~537 t/s en baseline — donc **~8% plus rapide**
- **~3707,46 ms AVG** en Time To First Token, quasiment identique au baseline
- **~4.5 Gi** de vRAM utilisé, contre ~6.1 Gi en baseline — soit **~26% de vRAM en moins**, avec un contexte 16 fois plus grand

Même histoire que sur la *T10*, et cette fois sans la moindre ambiguïté puisqu'on a retiré le MTP de l'équation : quantifier le KV-cache libère de la vRAM et permet un contexte bien plus grand, mais ça se paie clairement en vitesse de génération.

#### Et avec le MTP ?

Petite parenthèse avant de foncer dans les chiffres : c'est quoi le MTP ? Le *Multi-Token Prediction*, c'est une tête supplémentaire greffée sur le modèle, qui lui permet de proposer plusieurs tokens d'un coup à chaque passe au lieu d'un seul. Le serveur vérifie ensuite ces propositions en une seule passe, et ne garde que celles qui s'avèrent correctes. Contrairement au *speculative decoding* classique, qui repose sur un second modèle "draft" complet et distinct pour proposer ces tokens, le MTP utilise une tête intégrée au modèle principal — pas de second modèle à charger en mémoire à côté.

Même question que pour la *T10* : est-ce que rajouter le MTP permet de regagner un peu de la vitesse perdue ?

```bash
CUDA_VISIBLE_DEVICES=1 ~/pascal/llama.cpp/build/bin/llama-server --host 0.0.0.0 --port 8080 -m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf --model-draft ~/models/gemma-4-E4B-it-qat/mtp-gemma-4-E4B-it.gguf -fa on -ctk q4_0 -ctv q4_0 -c 65535
```

```
| model                |   test |            t/s |     peak t/s |        ttfr (ms) |     est_ppt (ms) |    e2e_ttft (ms) |
|:---------------------|-------:|---------------:|-------------:|-----------------:|-----------------:|-----------------:|
| google/gemma4-E4B-it |  pp512 |  632.95 ± 9.44 |              |   711.71 ± 40.98 |   707.82 ± 40.98 |   807.84 ± 41.80 |
| google/gemma4-E4B-it |  tg512 |   32.09 ± 0.06 | 33.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it |  pp512 | 621.82 ± 21.10 |              |   734.46 ± 32.46 |   730.56 ± 32.46 |   830.64 ± 33.05 |
| google/gemma4-E4B-it | tg2048 |   31.61 ± 0.51 | 32.00 ± 0.82 |                  |                  |                  |
| google/gemma4-E4B-it |  pp512 | 615.18 ± 13.59 |              |    728.85 ± 2.69 |    724.95 ± 2.69 |    824.72 ± 2.91 |
| google/gemma4-E4B-it | tg4096 |   31.54 ± 0.45 | 31.67 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  598.03 ± 8.90 |              | 3118.10 ± 153.00 | 3115.42 ± 153.00 | 3208.05 ± 152.01 |
| google/gemma4-E4B-it |  tg512 |   29.34 ± 0.07 | 30.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  589.45 ± 8.52 |              |  3153.12 ± 37.64 |  3150.44 ± 37.64 |  3242.11 ± 37.53 |
| google/gemma4-E4B-it | tg2048 |   29.02 ± 0.05 | 29.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp2048 |  584.55 ± 5.87 |              |  3314.65 ± 59.98 |  3311.97 ± 59.98 |  3403.88 ± 60.48 |
| google/gemma4-E4B-it | tg4096 |   28.97 ± 0.03 | 29.33 ± 0.47 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  539.74 ± 1.34 |              |  6937.52 ± 64.75 |  6934.84 ± 64.75 |  7028.14 ± 64.68 |
| google/gemma4-E4B-it |  tg512 |   28.45 ± 0.06 | 29.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  537.94 ± 1.73 |              | 6871.97 ± 104.42 | 6869.28 ± 104.42 | 6963.09 ± 103.85 |
| google/gemma4-E4B-it | tg2048 |   28.15 ± 0.06 | 29.00 ± 0.00 |                  |                  |                  |
| google/gemma4-E4B-it | pp4096 |  536.15 ± 2.54 |              | 6829.99 ± 249.02 | 6827.31 ± 249.02 | 6920.72 ± 248.83 |
| google/gemma4-E4B-it | tg4096 |   28.13 ± 0.10 | 29.00 ± 0.00 |                  |                  |                  |
```

Avec le MTP : **~29,70 t/s** en génération, contre **~29,52 t/s** sans. Une différence de **0,6%** — encore plus faible que sur la *T10*, et clairement dans le bruit de mesure. Même verdict : sur la *Pascal*, le MTP ne rattrape quasiment rien.

Et côté vRAM, rebelote : toujours **~4.5 Gi**, avec ou sans MTP, pour exactement la même raison que sur la *T10* — la tête MTP est greffée sur le modèle principal, pas un modèle draft séparé chargé à part, donc son empreinte est trop petite pour se voir à l'échelle où on mesure ici.

Sur la *P4* en tout cas, le verdict est clair : le MTP ne coûte quasiment rien en vRAM, mais ne rapporte quasiment rien en vitesse non plus, une fois le KV-cache quantifié. Reste à voir si la *T10* raconte la même histoire.

### Tesla T10

Passons à la carte qui coûte plus cher : 300€, plus de 3 fois le prix de la *Tesla P4*. Mais sur le papier, tout est meilleur : **16 Go** de vRAM au lieu de 8, un bus mémoire à **403.2 GB/s** au lieu de 192.3, plus de cœurs, et surtout, des *Tensor Cores* — l'architecture *Turing* les a, l'architecture *Pascal* de la P4 non.

C'est justement ce dernier point qui m'intéresse. Sur la P4, on a vu que quantifier le KV-cache en INT4 ralentissait la génération, probablement faute d'accélération matérielle pour ce genre de calcul. La T10, elle, a exactement le genre de silicium dédié qui pourrait éviter ce problème. Voyons voir si la théorie tient.

#### Commande

Même modèle, même méthode, on change juste de carte :

```bash
CUDA_VISIBLE_DEVICES=0 ~/turing/llama.cpp/build/bin/llama-server --host 0.0.0.0 --port 8080 -m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf
```

Seuls changements par rapport au lancement sur la *P4* : `CUDA_VISIBLE_DEVICES=0` pour cibler la *Tesla T10* cette fois, et le binaire `llama.cpp` compilé pour *Turing* (`~/turing/...` plutôt que `~/pascal/...`, vu que ce sont deux architectures CUDA différentes). Le reste des flags a déjà été expliqué plus haut.

#### Bench

Pendant que le test tourne on va regarder à quoi ressemble notre `nvtop`:

![nvtop t10](/img/nvtop-t10.png)

Après ~20min notre bench est fini:

```
| model                |   test |             t/s |     peak t/s |       ttfr (ms) |    est_ppt (ms) |   e2e_ttft (ms) |
|:---------------------|-------:|----------------:|-------------:|----------------:|----------------:|----------------:|
| google/gemma4-E4B-it |  pp512 | 1865.19 ± 15.40 |              |   254.97 ± 8.77 |   251.48 ± 8.77 |   298.96 ± 8.77 |
| google/gemma4-E4B-it |  tg512 |    75.49 ± 0.04 | 76.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it |  pp512 | 1838.42 ± 27.07 |              |   258.27 ± 5.63 |   254.77 ± 5.63 |   302.53 ± 5.61 |
| google/gemma4-E4B-it | tg2048 |    75.19 ± 0.46 | 75.33 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it |  pp512 | 1865.61 ± 13.04 |              |  257.17 ± 10.23 |  253.68 ± 10.23 |  301.61 ± 10.15 |
| google/gemma4-E4B-it | tg4096 |    74.67 ± 0.54 | 75.00 ± 0.82 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 | 2202.63 ± 14.41 |              |  830.65 ± 20.84 |  827.16 ± 20.84 |  877.01 ± 20.99 |
| google/gemma4-E4B-it |  tg512 |    72.25 ± 0.06 | 73.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 |  2178.48 ± 4.94 |              |  857.42 ± 12.54 |  853.93 ± 12.54 |  903.92 ± 12.52 |
| google/gemma4-E4B-it | tg2048 |    71.74 ± 0.22 | 72.33 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 |  2147.50 ± 5.75 |              |  856.32 ± 25.08 |  852.83 ± 25.08 |  902.75 ± 25.26 |
| google/gemma4-E4B-it | tg4096 |    71.58 ± 0.02 | 72.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 |  2207.94 ± 4.77 |              | 1689.38 ± 11.35 | 1685.89 ± 11.35 | 1737.32 ± 11.35 |
| google/gemma4-E4B-it |  tg512 |    70.53 ± 0.02 | 71.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 |  2176.68 ± 4.39 |              | 1707.47 ± 16.01 | 1703.98 ± 16.01 | 1755.38 ± 16.03 |
| google/gemma4-E4B-it | tg2048 |    69.78 ± 0.11 | 70.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 |  2165.45 ± 7.28 |              |  1710.31 ± 6.47 |  1706.82 ± 6.47 |  1758.25 ± 6.64 |
| google/gemma4-E4B-it | tg4096 |    69.74 ± 0.07 | 70.00 ± 0.00 |                 |                 |                 |
```

Ce qu'on retient :

- **~72,33 t/s AVG** en Text Generation
- **~2072,99 t/s AVG** en Prompt Processing
- **~982,00 ms AVG** en Time To First Token
- **6.5 Gi** de vRAM utilisé

Et là ça n'a plus rien à voir. Face à la *P4* en baseline (~35 t/s en TG, ~537 t/s en PP, ~3701 ms de TTFT), la *T10* fait plus du double en génération, presque 4x en prompt processing, et presque 4x plus rapide sur le time to first token. Pour 3.3x le prix, c'est largement au-dessus du delta tarifaire — cohérent avec les specs papier (bande passante mémoire 2.1x supérieure, plus de cœurs, et une architecture plus récente). La vRAM, elle, est quasi identique (6.5 contre 6.1 Gi), ce qui confirme que le petit écart vient bien d'un overhead driver/architecture et pas d'autre chose.

Reste la vraie question : est-ce que les *Tensor Cores* de la T10 lui permettent de cumuler vRAM en moins **et** vitesse en plus une fois les optimisations appliquées, là où la P4 a dû choisir entre les deux ?

#### Optimisation

Mêmes optimisations que sur la *P4* (Flash Attention, KV-cache en q4_0, contexte à 65535), on ne change que la carte :

```bash
CUDA_VISIBLE_DEVICES=0 ~/turing/llama.cpp/build/bin/llama-server --host 0.0.0.0 --port 8080 -m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf -fa on -ctk q4_0 -ctv q4_0 -c 65535
```

#### Bench

```bash
uvx llama-benchy --base-url http://ip-de-notre-z440:8080/v1 --served-model-name /root/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --model google/gemma4-E4B-it --pp 512 2048 4096 --tg 512 2048 4096
```

```
| model                |   test |             t/s |     peak t/s |       ttfr (ms) |    est_ppt (ms) |   e2e_ttft (ms) |
|:---------------------|-------:|----------------:|-------------:|----------------:|----------------:|----------------:|
| google/gemma4-E4B-it |  pp512 | 1988.38 ± 36.14 |              |   237.59 ± 7.58 |   236.00 ± 7.58 |   287.23 ± 7.72 |
| google/gemma4-E4B-it |  tg512 |    67.42 ± 0.10 | 68.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it |  pp512 | 1951.59 ± 29.81 |              |   243.16 ± 7.08 |   241.57 ± 7.08 |   293.13 ± 7.21 |
| google/gemma4-E4B-it | tg2048 |    66.54 ± 0.47 | 67.33 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it |  pp512 | 1919.20 ± 30.58 |              |   250.24 ± 4.15 |   248.65 ± 4.15 |   300.43 ± 4.32 |
| google/gemma4-E4B-it | tg4096 |    67.30 ± 0.17 | 68.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 | 2250.24 ± 12.67 |              |  833.73 ± 17.48 |  832.14 ± 17.48 |  885.65 ± 17.62 |
| google/gemma4-E4B-it |  tg512 |    64.83 ± 0.08 | 65.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 | 2221.61 ± 25.12 |              |  829.26 ± 30.35 |  827.67 ± 30.35 |  881.16 ± 30.47 |
| google/gemma4-E4B-it | tg2048 |    64.16 ± 0.53 | 64.67 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 |  2171.53 ± 8.33 |              |   854.46 ± 3.10 |   852.87 ± 3.10 |   906.73 ± 3.10 |
| google/gemma4-E4B-it | tg4096 |    63.93 ± 0.50 | 64.33 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 | 2186.28 ± 13.02 |              | 1694.95 ± 60.25 | 1693.36 ± 60.25 | 1749.37 ± 59.86 |
| google/gemma4-E4B-it |  tg512 |    61.58 ± 0.42 | 61.67 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 |  2170.55 ± 5.64 |              | 1710.09 ± 12.77 | 1708.50 ± 12.77 | 1764.26 ± 13.04 |
| google/gemma4-E4B-it | tg2048 |    60.56 ± 0.21 | 61.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 | 2173.51 ± 27.94 |              |  1734.22 ± 7.05 |  1732.63 ± 7.05 |  1788.50 ± 6.83 |
| google/gemma4-E4B-it | tg4096 |    60.25 ± 0.14 | 61.00 ± 0.00 |                 |                 |                 |
```

Ce qu'on retient, en comparant au baseline *T10* :

- **~64,06 t/s AVG** en Text Generation, contre ~72,33 t/s en baseline — donc **~11% plus lent**
- **~2114,77 t/s AVG** en Prompt Processing, contre ~2072,99 t/s en baseline — donc **~2% plus rapide**
- **~984,05 ms AVG** en Time To First Token, quasi identique au baseline
- **4.8 Gi** de vRAM utilisé, contre 6.5 Gi en baseline — soit **~26% de vRAM en moins** (même ratio que sur la *P4*, mais en absolu la *T10* consomme quand même plus que la *P4*, aussi bien en baseline qu'optimisé)

Même verdict que sur la *P4*, sans surprise cette fois puisqu'on a retiré la seule variable qui pouvait brouiller le signal : quantifier le KV-cache réduit bien l'empreinte mémoire — ce qui laisse de la place pour un contexte plus grand — mais ça se paie en vitesse de génération. Pas de repas gratuit.

Ce qui répond à la question posée plus haut : les *Tensor Cores* n'évitent pas le problème vu sur la *P4*, ils l'atténuent seulement. **-11%** sur la *T10* contre **-16%** sur la *P4* : moins pire, mais toujours pas gratuit. Cohérent avec le constat déjà fait sur la vRAM identique avec ou sans MTP — ce n'est pas la puissance de calcul brute qui manque pour déquantifier le KV-cache à chaque token, donc plus de *Tensor Cores* n'aide qu'à la marge.

#### Et avec le MTP ?

Reste une question ouverte : peut-on se rapprocher des vitesses d'origine en ajoutant le MTP par-dessus, sans renoncer aux gains de vRAM ? Même commande, on rajoute juste `--model-draft` :

```bash
CUDA_VISIBLE_DEVICES=0 ~/turing/llama.cpp/build/bin/llama-server --host 0.0.0.0 --port 8080 -m ~/models/gemma-4-E4B-it-qat/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf --mmproj ~/models/gemma-4-E4B-it-qat/mmproj-F16.gguf --model-draft ~/models/gemma-4-E4B-it-qat/mtp-gemma-4-E4B-it.gguf -fa on -ctk q4_0 -ctv q4_0 -c 65535
```

```
| model                |   test |             t/s |     peak t/s |       ttfr (ms) |    est_ppt (ms) |   e2e_ttft (ms) |
|:---------------------|-------:|----------------:|-------------:|----------------:|----------------:|----------------:|
| google/gemma4-E4B-it |  pp512 | 2069.27 ± 71.73 |              |  224.31 ± 11.39 |  221.16 ± 11.39 |  274.02 ± 11.57 |
| google/gemma4-E4B-it |  tg512 |    68.16 ± 0.16 | 68.67 ± 0.47 |                 |                 |                 |
| google/gemma4-E4B-it |  pp512 | 1971.28 ± 26.19 |              |  241.20 ± 10.83 |  238.06 ± 10.83 |  290.88 ± 10.77 |
| google/gemma4-E4B-it | tg2048 |    67.72 ± 0.16 | 68.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it |  pp512 | 1962.64 ± 11.68 |              |   240.62 ± 7.49 |   237.48 ± 7.49 |   290.59 ± 7.34 |
| google/gemma4-E4B-it | tg4096 |    67.41 ± 0.59 | 68.00 ± 0.82 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 |  2238.91 ± 9.44 |              |   824.99 ± 5.05 |   821.85 ± 5.05 |   876.51 ± 5.05 |
| google/gemma4-E4B-it |  tg512 |    65.55 ± 0.01 | 66.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 |  2221.51 ± 7.36 |              |  859.13 ± 14.64 |  855.98 ± 14.64 |  911.05 ± 14.70 |
| google/gemma4-E4B-it | tg2048 |    64.83 ± 0.05 | 65.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp2048 | 2188.13 ± 14.61 |              |  837.90 ± 33.00 |  834.75 ± 33.00 |  889.73 ± 33.18 |
| google/gemma4-E4B-it | tg4096 |    64.65 ± 0.21 | 65.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 |  2196.95 ± 2.31 |              | 1666.21 ± 20.25 | 1663.07 ± 20.25 | 1720.12 ± 20.14 |
| google/gemma4-E4B-it |  tg512 |    62.70 ± 0.04 | 63.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 |  2201.92 ± 4.87 |              | 1670.53 ± 39.07 | 1667.38 ± 39.07 | 1724.42 ± 39.29 |
| google/gemma4-E4B-it | tg2048 |    61.86 ± 0.15 | 62.00 ± 0.00 |                 |                 |                 |
| google/gemma4-E4B-it | pp4096 | 2183.82 ± 10.99 |              | 1725.97 ± 14.33 | 1722.83 ± 14.33 | 1780.36 ± 14.30 |
| google/gemma4-E4B-it | tg4096 |    61.04 ± 0.01 | 62.00 ± 0.00 |                 |                 |                 |
```

Avec le MTP : **~64,88 t/s** en génération, contre **~64,06 t/s** sans. Une différence de **1,3%** — dans la marge d'erreur des écarts-types qu'on voit sur le tableau. Autrement dit, non : le MTP ne compense quasiment pas la perte de vitesse causée par la quantification du KV-cache, en tout cas pas dans ces proportions. Le prompt processing bouge un peu plus (~2137 contre ~2115 t/s, +1%), mais rien qui change la conclusion.

Côté vRAM non plus, rien ne bouge : toujours **4.8 Gi**, MTP activé ou pas. Logique en y réfléchissant : contrairement au *speculative decoding* classique, qui charge un second modèle draft complet à côté du modèle principal (et qui lui coûte réellement plusieurs centaines de Mo à plusieurs Go de vRAM en plus), le MTP ici n'est qu'une petite tête de prédiction supplémentaire greffée sur le *même* modèle, pas un modèle séparé chargé en mémoire. Face aux quelques Gio déjà occupés par les poids du modèle et le KV-cache, son empreinte est négligeable — invisible à l'échelle du dixième de Gio à laquelle on mesure ici.

Ce qui aurait du sens en théorie (le MTP réduit le nombre de passes séquentielles, donc devrait limiter l'impact d'un decode plus coûteux à chaque passe) ne se vérifie pas vraiment ici. Possible que le gain du MTP soit déjà en grande partie mangé par le coût de la vérification des tokens proposés, qui elle aussi doit lire le KV-cache quantifié. Sans profiler plus finement, difficile d'aller plus loin — mais le constat reste : sur ce setup, on ne récupère pas la vitesse perdue avec du MTP. Le vrai correctif serait un format de quantification supporté nativement par le hardware plutôt que déquantifié en logiciel à chaque token, ce qui n'existe pas encore dans notre pipeline actuel.

Verdict pour les deux cartes, donc : le MTP ne coûte quasiment rien en vRAM, mais ne rapporte quasiment rien en vitesse non plus, une fois le KV-cache quantifié. Sur ce setup précis, il n'y a pas de round trip gratuit entre les deux.

## Conclusion

J'ai pris ces deux cartes pour leur prix, leur consommation électrique, et leur taille — le genre de critères qui reviennent tout le temps chez moi, parce que mon but, comme d'habitude, c'est de faire beaucoup avec un rien.

On arrive bien à faire tourner des modèles sur ces cartes, et plutôt proprement. Avoir ce genre de vitesses, sur ce genre de cartes, à ce prix, c'est franchement intéressant : elles sont petites, elles consomment peu, et elles restent relativement accessibles au vu des prix actuels du marché du GPU. Pour de l'usage local, ça a vraiment du sens. Et au-delà de ce qu'on a testé ici, le rythme auquel le monde de l'IA avance en ce moment — speculative decoding, quantification, MoE, Turboquant, et j'en passe — ne fait que rendre ce genre de setup encore plus pertinent avec le temps.

Est-ce que je recommanderais ces cartes pour autant ? Non.

Elles sont pas chères, mais elles restent très limitées niveau vRAM et surtout bande passante mémoire — le facteur qui compte le plus pour la vitesse de génération, comme on l'a vu tout du long de cet article. Et si vous n'avez pas déjà un boîtier ou une baie pensée pour ce genre de cartes serveur, il faut être prêt à imprimer des shrouds en 3D et à bidouiller de l'électrique pour les refroidir correctement. Un chouette projet à mener, mais pas franchement une base sur laquelle construire une infra fiable au quotidien.

*Cet article a été co-écrit avec Claude. Tout ce qui y est écrit peut donc contenir des erreurs, des raccourcis ou des approximations.*
