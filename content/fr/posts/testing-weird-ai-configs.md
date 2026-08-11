---
date: "2026-08-11T00:22:00+02:00"
title: "De l'IA sur des cartes sortient de nul part"
description: "Héberger des modèles d'IA générative sur des GPU étranges"
tags: ["GenAI", "Z440", "NVIDIA", "AI", "GPU", "Inference", "LLM", "llama.cpp", "vLLM"]
---

---

## Dans quel but ?

Depuis que je m'intéresse à l'IA j'aime bien faire mumuse avec des GPU étranges. Après avoir fait [Un setup IA moins cher que Mario Kart World](/posts/cheap-ai-server/), je me suis demandé si on pouvais héberger des modèles d'IA performant avec des GPU vraiment pas cher. Et spoiler, oui on peut. Mais avec quelque point noir.

## Le setup

Le setup reste plus ou moins le même que pour l'article précedent. Pour rappel:

- Alimentation HP 700W  
- [Intel Xeon E5-1620 v3 @ 3.50GHz](https://www.intel.fr/content/www/fr/fr/products/sku/82763/intel-xeon-processor-e51620-v3-10m-cache-3-50-ghz/specifications.html)
- 32 Go RAM DDR4 ECC 2133MHz
- [SSD Samsung EVO 250 Go](https://www.samsung.com/fr/memory-storage/sata-ssd/870-evo-250gb-sata-3-2-5-ssd-mz-77e250b-eu/)
- [Quadro K2200 4 Go](https://www.nvidia.com/content/dam/en-zz/Solutions/design-visualization/documents/75509_DS_NV_Quadro_K2200_US_NV_HR.pdf)
- [GTX 1050 Ti 4 Go](https://www.gigabyte.com/fr/Graphics-Card/GV-N105TG1-GAMING-4GD)

Mais on vas y ajouter 2 cartes un peu suspecte :

- [NVIDIA Tesla P4](https://www.techpowerup.com/gpu-specs/tesla-p4.c2879)
- [NVIDIA Tesla T10](https://www.techpowerup.com/gpu-specs/tesla-t10-16-gb.c4036)

La [NVIDIA Tesla P4](https://www.techpowerup.com/gpu-specs/tesla-p4.c2879) est une [NVIDIA GeForce GTX 1080](https://www.techpowerup.com/gpu-specs/geforce-gtx-1080.c2839) avec un form-factor réduit(HHHL), un cooling passif, et qui ne consomme que 75W, donc pas d'alimentation exeterne !

Elle date de 2016 et propose **8Gi GDDR5** avec un bus mémoire d'une vitesse de **192.3 GB/s** et **5.704 TFLOPS** en FP32. Selon mes sources (moi et mes multiples personnalitées), ce GPU était utilisé principalement à des fins de VDI(Virtual Desktop Infrastructure) et de transcodage.

![Techpowerup Tesla P4](/img/tpu-nvidia-p4.png)

La [NVIDIA Tesla T10](https://www.techpowerup.com/gpu-specs/tesla-t10-16-gb.c4036) elle est un peu plus spéciale. Bien que comparable à une [NVIDIA GeForce RTX 2080 Ti](https://www.techpowerup.com/gpu-specs/geforce-rtx-2080-ti.c3305), elle dispose de moins de **coeurs**, moins de **TMUs**, moins de **ROPS** et d'un bus mémoire plus petit. Elle à un form-factor single-slot PCIe (FHFL), un cooling passif, et consomme 150W via un port externe 8-PIN PCIe.

Elle date de 2020 et propose **16Gi GDRR6** avec un bus mémoire d'une vitesse de **403.2 GB/s** et **9.999 TFLOPS** en FP32. Après plusieurs heures de recherches, j'ai trouvé que cette carte était utilisé par *Nvidia* pour *GeForce Now*. Elle servait donc pour du cloud gaming ! 

> Les cartes Nvidia sous architecture Turing (RTX 20xx) sont les permières cartes à proposer des *Tensor Cores*, des coeurs dédier à accélérer des calculs spécifiquent à l'IA. C'est en grande partie ce qui as permis l'arrivé du [DLSS](https://www.nvidia.com/fr-fr/geforce/technologies/dlss/). 

![Techpowerup Tesla T10](/img/tpu-nvidia-t10.png)

### Pourquoi ces cartes sont étranges ?

Les cartes Nvidia *GTX*, *RTX*, ou *Quadro* sont relativement connu du publique. Elles sont utilisé pour du gaming, du rendu vidéo, de la CAO et d'autre taches commune. Mais les cartes *Tesla* elles sont dédier à des taches bien plus spécifiques, notament de la virtualisation via des solutions comme [vGPU](https://docs.nvidia.com/vgpu/index.html) ou [MIG](https://www.nvidia.com/fr-fr/technologies/multi-instance-gpu/), pour être utilisés dans plusieurs machines virtuelles en même temps(ce qui explique que la *Tesla T10* ai été utilisé pour *GeForce Now*). Elles sont aussi souivent dédier à des taches de *Machine Learning*. Elles sont basées sur les mêmes puces utilisés pour les cartes grand publique mais sont déstiner à finir dans des serveurs(d'où le cooling passif).

### Le refroidissement

Maintenant qu'on connais un peu mieux ces cartes, il faut les refroidire. La workstation *HP Z440* n'a jamais été faite pour acceuillir ce type de cartes. Il vas donc falloir se démerder autrement parce qu'une carte sans cooling qui monte à **97°C** c'est vraiment pas une bonne idée. On doit donc passer à du cooling actif.

Et qui dit cooling actif dit ventilateurs !

> ⚠️ Les modifications qui ont été faite ici ne sont pas recommandées et risqués. Jouer avec de l'électricité même à 12V peut vous couter la vie donc éviter de jouer aux cons. Je n'incite personne à reproduire les modifications suivante et décline toutes résponsabilité de perte de garantie, de casse matériel, ou de dégat physique.

Pour alimenter nos ventilateurs on as besoin d'un source d'électricité, et quoi de mieux qu'un port d'alimentation *SATA* pas utilisé ? Ce dernier peut nous fournir du **3.3V à 1.5A**, **5V à 4.5A**, **12V à 4.5A** ce qui est parfait pour alimenter nos ventilos.

> Pour plus d'information: https://tadeubento.com/2025/sata-connector-power-rating-and-hard-drives/

J'ai donc bidouillé un adaptateur *SATA* -> Ventilateur *3 Pin* branché sur le rail **12V** (Bon ok il n'y as que 2 pin sur le port du ventilateur d'utilisés mais le 3ème ne sert qu'au PWM / réguler la vitesse ce dont on as pas besoin ici.)

![SATA TO 3 PIN](/img/cooling-sata-adapter.png)

Ensuite il faut refroidire notre *Tesla P4*. J'ai donc imprimé un shroud en 3D pour accepter un ventilateur **30mm** alimenté en **12V à 0.1A** et pousser de l'air directement sur le bloc de cuivre du GPU. C'est pas la meilleur façon de refroidire cette carte mais j'avais la flemme de designer quelque chose d'autre et j'avais pas d'autre ventilateur assez petit.

![Tesla P4 Cooling](/img/cooling-p4.png)

Maintant pour la *Tesla T10* on sort l'artillerie lourde. J'ai imprimé un autre shroud en 3D pour accepter un ventilateur **30mm** alimenté en **12V à 0.40A** mais cette fois ci en mode *blower*. On respecte donc le flux d'air de la carte.

![Tesla T10 Cooling](/img/cooling-t10.png)

J'utilise en suite un splriter pour ventilateur pour répartir les **12V** sur les 2 cartes. En additionnant les **0.1A** de la *Tesla P4* au **0.40A** de la *Tesla T10* on arrive à **0.50A** ce qui reste dans la norme du **12V à 4.5A** de notre port d'alimentation *SATA* et on arrive à 6W de cooling pour nos cartes. Ça parrait peut, et pourtant c'est amplement suffisant pour notre setup.