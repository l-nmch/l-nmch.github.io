---
date: "2026-08-13T03:00:00+02:00"
title: "Devenir le futur Anthropic avec un GPU de 2020"
description: "Entraîner un LLM from scratch — dataset, pretraining, fine-tuning, quantization — sur une Tesla T10 sortie de la flotte GeForce Now"
tags: ["GenAI", "LLM", "PyTorch", "Docker", "NVIDIA", "T10", "Training", "GGUF"]
---

---

## Pourquoi ce projet ?

Dans [le post précédent](/posts/testing-weird-ai-configs/), j'ai fini par faire tourner de l'inférence sur une Tesla T10 récupérée pour 300€ — une carte qui, dans une autre vie, streamait du cloud gaming chez GeForce Now. Sympa, mais soyons honnêtes deux secondes : faire tourner un modèle que quelqu'un d'autre a entraîné, c'est le service minimum syndical. N'importe qui avec `ollama pull` sait faire ça en trente secondes.

Ce qui me démangeait, c'était la partie d'avant. D'où sort le modèle. Comment on passe d'un tas de texte brut à un truc qui aligne des phrases qui tiennent debout. Quand on utilise Claude, ChatGPT ou Mistral au quotidien, on tape un prompt, ça sort une réponse, et entre les deux c'est de la magie noire pour à peu près tout le monde qui s'en sert.

Ce projet, c'est l'anti-magie noire. Est-ce que je suis capable de faire tourner *tout* le pipeline moi-même, du texte brut jusqu'au modèle quantifié prêt à déployer, sans passer par nanoGPT ou un `AutoModel.from_pretrained()` qui fait le sale boulot à ma place ?

Spoiler : oui. Mais pas sans me prendre un bug bien vicieux dans la gueule au passage. J'y reviens, il le mérite amplement.

Le pipeline complet tient en quatre étapes :

```mermaid
flowchart TB
    A["Texte brut<br/>(2,1M histoires)"] -->|tokenisation| B["Dataset<br/>tokenisé"]
    B -->|"pretraining<br/>(deviner le mot suivant)"| C["Modèle de base<br/>sait continuer du texte"]
    C -->|"SFT<br/>(instruction → réponse)"| D["Modèle instruct<br/>sait suivre une consigne"]
    D -->|quantization| E["GGUF quantifié<br/>léger, déployable"]
```

**Dataset → Pretraining → SFT → PTQ.** Quatre sections, un article. Et tout ça sur la T10, en Docker, pour ne pas transformer hpc1 en poubelle de dépendances Python à chaque expérience foireuse.

---

## Le setup

Rien de neuf côté hardware par rapport au post précédent — toujours la même workstation HP Z440 et sa Tesla T10 (Turing, compute capability 7.5, 15,6 Go de VRAM). Ce qui change, c'est que cette fois tout passe par [Docker](https://docker.com) + le [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/index.html), pour pouvoir tout jeter et recommencer sans laisser de merde sur le système.

Installation de Docker CE (Ubuntu 26.04 n'étant pas encore dans les dépôts officiels au moment de l'écriture, le suffixe `noble` de la 24.04 fait très bien le café) :

```bash
apt-get update
apt-get install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  noble stable" | tee /etc/apt/sources.list.d/docker.list

apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Puis le NVIDIA Container Toolkit, pour que les conteneurs puissent voir le GPU :

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed "s#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g" | \
  tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

apt-get update
apt-get install -y nvidia-container-toolkit

nvidia-ctk runtime configure --runtime=docker
systemctl restart docker
```

Et la vérification qui va bien :

```bash
docker run --rm --gpus all nvidia/cuda:12.6.3-base-ubuntu24.04 nvidia-smi -L
# GPU 0: Tesla T10 (UUID: ...)
```

Pour l'entraînement, une image basée sur l'image officielle PyTorch+CUDA, avec juste ce qu'il faut en plus (`docker/Dockerfile`) :

```dockerfile
FROM pytorch/pytorch:2.6.0-cuda12.4-cudnn9-runtime

WORKDIR /workspace

RUN pip install --no-cache-dir \
    datasets \
    tokenizers \
    tqdm \
    numpy \
    safetensors

COPY src/ /workspace/src/
```

```bash
docker build -t llm-from-scratch -f docker/Dockerfile .

docker run --rm --gpus '"device=0"' llm-from-scratch \
  python -c "import torch; print(torch.cuda.get_device_name(0))"
# Tesla T10
```

Petit détail qui a son importance pour la suite : la T10 c'est du **Turing**, donc des Tensor Cores en **fp16**, mais pas de bf16 natif (arrivé seulement avec Ampere). Tout l'entraînement tourne donc en `torch.autocast` fp16 + `GradScaler`, pas en bf16, le réflexe habituel sur une carte plus récente.

---

## Le dataset

Pour un pretraining from scratch qui tienne dans une soirée plutôt que dans un mois, il fallait un dataset petit et simple. J'ai pris **[TinyStories](https://huggingface.co/datasets/roneneldan/TinyStories)** : un corpus de contes générés par GPT-3.5/4 avec un vocabulaire volontairement restreint, conçu par Microsoft Research pile pour prouver qu'un tout petit modèle peut sortir du texte cohérent.

Petite parenthèse pour ceux qui n'ont jamais mis les mains là-dedans : un LLM ne lit pas du texte lettre par lettre. Avant de rentrer quoi que ce soit dans le modèle, il faut découper le texte en petits morceaux appelés **tokens** — parfois un mot entier, parfois juste un bout de mot, parfois un seul caractère. L'outil qui fait ce découpage s'appelle un **tokenizer**, et le **vocab_size** c'est simplement le nombre de tokens différents qu'il connaît. La méthode la plus courante pour le construire s'appelle le **BPE** (Byte Pair Encoding) : en gros, on part des caractères bruts, et on fusionne petit à petit les paires qui reviennent le plus souvent dans le corpus, jusqu'à obtenir un vocabulaire de la taille voulue. Plus ce vocabulaire est petit, plus le modèle croise souvent les mêmes tokens pendant l'entraînement — pratique quand le corpus est volontairement simple comme TinyStories.

2 119 719 histoires pour l'entraînement, 21 990 pour la validation. J'ai entraîné mon propre tokenizer BPE byte-level dessus (vocab_size 8192, via la lib `tokenizers` de HuggingFace — écrire un BPE trainer complètement à la main aurait été un chantier à part entière, donc là je fais une exception) puis tout tokenisé dans deux gros fichiers binaires `uint16`, chargeables en `memmap` pendant l'entraînement sans tout faire tenir en RAM. Le script (`src/prepare_data.py`) au complet :

```python
from pathlib import Path

import numpy as np
from datasets import load_dataset
from tokenizers import Tokenizer, decoders, models, pre_tokenizers, trainers

DATA_DIR = Path("/workspace/data")
VOCAB_SIZE = 8192


def train_tokenizer(dataset):
    tokenizer = Tokenizer(models.BPE(unk_token="<unk>"))
    tokenizer.pre_tokenizer = pre_tokenizers.ByteLevel(add_prefix_space=False)
    tokenizer.decoder = decoders.ByteLevel()

    trainer = trainers.BpeTrainer(
        vocab_size=VOCAB_SIZE,
        special_tokens=["<unk>", "<pad>", "<eos>"],
    )

    def text_iterator():
        for example in dataset:
            yield example["text"]

    tokenizer.train_from_iterator(text_iterator(), trainer=trainer, length=len(dataset))
    return tokenizer


def tokenize_split(tokenizer, dataset, eos_id, out_path):
    ids = []
    for example in dataset:
        ids.extend(tokenizer.encode(example["text"]).ids)
        ids.append(eos_id)
    arr = np.array(ids, dtype=np.uint16)
    arr.tofile(out_path)
    return len(arr)


def main():
    DATA_DIR.mkdir(parents=True, exist_ok=True)

    ds = load_dataset("roneneldan/TinyStories")
    tokenizer = train_tokenizer(ds["train"])
    tokenizer.save(str(DATA_DIR / "tokenizer.json"))
    eos_id = tokenizer.token_to_id("<eos>")

    n_train = tokenize_split(tokenizer, ds["train"], eos_id, DATA_DIR / "train.bin")
    n_val = tokenize_split(tokenizer, ds["validation"], eos_id, DATA_DIR / "val.bin")
    print(f"train.bin: {n_train:,} tokens, val.bin: {n_val:,} tokens")


if __name__ == "__main__":
    main()
```

```bash
docker run --rm --gpus '"device=0"' \
  -v $(pwd)/src:/workspace/src -v $(pwd)/data:/workspace/data \
  llm-from-scratch \
  python src/prepare_data.py
```

```
train.bin: 466 602 707 tokens (890 Mo)
val.bin:     4 689 785 tokens (9 Mo)
```

Téléchargement HuggingFace, entraînement du tokenizer, encodage des 2,1M histoires : tout ça, c'est du CPU pur et dur. Aucune matmul, donc la T10 reste à 0% pendant que je regarde une barre de progression pendant 16 minutes. Ça fait partie du folklore : on croit lancer un truc GPU et en fait on attend juste que Python finisse une boucle for.

---

## Le modèle : un GPT decoder-only écrit à la main

Ici, pas de nanoGPT, pas de `transformers.GPT2Model`. Un transformer decoder-only classique, construit brique par brique dans un seul fichier `model.py`.

Pour ceux qui n'ont jamais ouvert le capot d'un transformer : le cœur du mécanisme, c'est l'**attention**. C'est ce qui permet à chaque mot de "regarder" les mots qui le précèdent et de décider lesquels comptent pour deviner la suite. Dans "le chat mange sa ___", le modèle doit comprendre que "chat" et "mange" pèsent plus lourd que "le" pour deviner "gamelle" plutôt que n'importe quoi d'autre. **Multi-tête**, ça veut dire qu'il fait ce calcul plusieurs fois en parallèle, avec des "angles" différents à chaque fois, pour capter plusieurs types de relations entre les mots simultanément. **Causale**, ça veut dire qu'un mot ne peut regarder QUE ce qui vient avant lui, jamais ce qui vient après — sinon le modèle trichererait en lisant la réponse avant de l'avoir devinée, et l'entraînement ne voudrait plus rien dire.

Le détail de ce qui a été codé à la main, sans raccourci :

- **Embeddings** token + position
- **Attention causale multi-tête**, codée à la main : projections QKV, scores `QKᵀ/√head_dim`, masque triangulaire causal précalculé, softmax — pas de kernel fusionné, pas de flash-attention, du PyTorch pur et dur
- **MLP** classique derrière chaque bloc d'attention, pour digérer ce que l'attention a repéré (expansion ×4, GELU, projection retour) — rien de plus exotique que deux couches linéaires avec une activation entre les deux
- **Pre-norm** partout (LayerNorm *avant* l'attention/le MLP plutôt qu'après) — c'est ce qui entraîne le plus stablement un petit modèle sans passer des heures à tuner le warmup
- **Weight tying** entre l'embedding d'entrée et la tête de sortie — les deux partagent littéralement la même matrice de poids, ce qui réduit le nombre de paramètres et aide un petit modèle à mieux généraliser

Config et attention, le cœur du fichier `src/model.py` :

```python
@dataclass
class GPTConfig:
    vocab_size: int = 8192   # aligné sur le tokenizer entraîné dans prepare_data.py
    block_size: int = 256    # longueur de contexte max
    n_layer: int = 8
    n_head: int = 8
    n_embd: int = 512
    dropout: float = 0.0


class CausalSelfAttention(nn.Module):
    def __init__(self, config: GPTConfig):
        super().__init__()
        self.n_head = config.n_head
        self.head_dim = config.n_embd // config.n_head

        self.qkv_proj = nn.Linear(config.n_embd, 3 * config.n_embd)
        self.out_proj = nn.Linear(config.n_embd, config.n_embd)

        # position i ne peut regarder que les positions <= i
        mask = torch.tril(torch.ones(config.block_size, config.block_size, dtype=torch.bool))
        self.register_buffer("causal_mask", mask, persistent=False)

    def forward(self, x):
        B, T, C = x.shape
        qkv = self.qkv_proj(x)
        q, k, v = qkv.split(C, dim=2)

        q = q.view(B, T, self.n_head, self.head_dim).transpose(1, 2)
        k = k.view(B, T, self.n_head, self.head_dim).transpose(1, 2)
        v = v.view(B, T, self.n_head, self.head_dim).transpose(1, 2)

        att = (q @ k.transpose(-2, -1)) / (self.head_dim ** 0.5)
        att = att.masked_fill(~self.causal_mask[:T, :T], float("-inf"))
        att = F.softmax(att, dim=-1)

        out = att @ v
        out = out.transpose(1, 2).contiguous().view(B, T, C)
        return self.out_proj(out)
```

Le reste (`MLP`, `Block` qui assemble attention + MLP en pre-norm avec résiduelles, `GPT` qui empile les blocs) suit la même logique, sans surprise. Config finale : 8 couches, 8 têtes, dimension 512, contexte de 256 tokens. **29,5 millions de paramètres.** Un poids plume comparé à n'importe quel LLM du commerce (GPT-4 ou Claude tournent avec plusieurs ordres de grandeur de paramètres en plus), mais amplement suffisant pour ce qu'on lui demande.

Petit test de sanité avant de se lancer :

```bash
docker run --rm --gpus '"device=0"' \
  -v $(pwd)/src:/workspace/src -w /workspace/src \
  llm-from-scratch \
  python -c "
import torch
from model import GPT, GPTConfig

config = GPTConfig()
model = GPT(config).cuda()
print('Parameters:', sum(p.numel() for p in model.parameters()))

idx = torch.randint(0, config.vocab_size, (4, 128), device='cuda')
targets = torch.randint(0, config.vocab_size, (4, 128), device='cuda')
logits, loss = model(idx, targets)
print('loss:', loss.item())
"
```

```
Parameters: 29,545,472
loss: 9.20398998260498
```

À l'initialisation, sans le moindre entraînement, la loss était de 9.20 — cohérent avec `ln(8192) ≈ 9.01`, la loss théorique d'un modèle qui prédit au hasard sur les 8192 tokens du vocabulaire. Toujours satisfaisant de voir les maths tomber juste avant même d'avoir entraîné quoi que ce soit.

---

## Le pretraining

Petite parenthèse encore, parce que ça va revenir tout du long : la **loss**, c'est une seule note qui résume à quel point le modèle se plante en moyenne à chaque mot qu'il essaie de deviner. Plus c'est bas, mieux c'est. Elle part forcément haute (le modèle sort n'importe quoi au hasard, comme on vient de le voir) et elle est censée descendre au fur et à mesure qu'il apprend des régularités dans le texte.

Boucle d'entraînement maison (`src/train.py`) : chargement des `.bin` en `np.memmap` (pas besoin de charger 890 Mo en RAM d'un coup), échantillonnage aléatoire de fenêtres de `block_size` tokens, `AdamW`, warmup linéaire suivi d'un cosine decay, fp16 + `GradScaler`, gradient clipping. Le cœur du batching, qui décale bien x et y d'un cran (ça va compter plus tard, dans la partie SFT) :

```python
def get_batch(data, block_size):
    ix = torch.randint(len(data) - block_size - 1, (batch_size,))
    x = torch.stack([torch.from_numpy(data[i:i + block_size].astype(np.int64)) for i in ix])
    y = torch.stack([torch.from_numpy(data[i + 1:i + 1 + block_size].astype(np.int64)) for i in ix])
    return x.to(device), y.to(device)
```

Et la boucle elle-même, en version condensée :

```python
for it in range(max_iters + 1):
    x, y = get_batch(train_data, config.block_size)
    with torch.autocast(device_type="cuda", dtype=torch.float16):
        _, loss = model(x, y)

    optimizer.zero_grad(set_to_none=True)
    scaler.scale(loss).backward()
    scaler.unscale_(optimizer)
    torch.nn.utils.clip_grad_norm_(model.parameters(), grad_clip)
    scaler.step(optimizer)
    scaler.update()
```

Un smoke test à froid sur 20 itérations donnait ~48k tokens/s — de quoi estimer qu'une **epoch** complète (le modèle voit chaque exemple du dataset une fois, pas plus) sur les 466M tokens prendrait dans les 2h40.

J'ai lancé le run complet en conteneur détaché (`docker run -d`), parce que couper une session SSH au milieu d'un entraînement de plusieurs heures pour perdre le boulot, très peu pour moi :

```bash
docker run -d --name llm-pretrain --gpus '"device=0"' \
  -v $(pwd)/src:/workspace/src -v $(pwd)/data:/workspace/data -v $(pwd)/checkpoints:/workspace/checkpoints \
  -w /workspace/src \
  llm-from-scratch \
  python train.py --max_iters 28479 --eval_interval 500 --eval_iters 50

docker logs -f llm-pretrain   # pour suivre en direct
docker wait llm-pretrain      # pour bloquer jusqu'à la fin
```

Le débit réel s'est stabilisé autour de **71 000 tokens/s** une fois le warmup CUDA passé — largement au-dessus de l'estimation à froid. Une epoch complète, **1h49** au lieu des 2h40 prévues.

![Courbe de loss du pretraining](/img/llm-scratch-pretrain-loss.png)

Train et val se superposent quasiment parfaitement du début à la fin. Logique : avec 466M tokens vus une seule fois, le modèle n'a jamais l'occasion de revoir un exemple, donc zéro surapprentissage possible. La loss finale atterrit à **1.66**, contre 9.17 au départ.

Et surtout, ça marche. Un prompt "Once upon a time" balancé au modèle fraîchement pretrainé (`GPT.generate()`, échantillonnage avec température + top-k, ajouté à `model.py`) :

```bash
docker run --rm --gpus '"device=0"' \
  -v $(pwd)/src:/workspace/src -v $(pwd)/data:/workspace/data -v $(pwd)/checkpoints:/workspace/checkpoints \
  -w /workspace/src \
  llm-from-scratch \
  python sample.py --prompt "Once upon a time" --max_new_tokens 150
```

> *Once upon a time there was a little girl. She was very small and wanted to find a way out. The little girl looked all around her. She searched high and low, but she couldn't find anything. Then, she saw a big bird's nest in the tree. [...]*

Grammaticalement correct, cohérent sur plus de cent tokens, pile dans le style TinyStories. Pour 29,5M de paramètres entraînés en moins de deux heures sur une carte de cloud gaming reconvertie, je trouve ça franchement pas dégueulasse.

---

## Le fine-tuning (SFT)

Parenthèse importante ici, probablement le truc le moins connu de tous ceux qui utilisent l'IA sans avoir jamais mis le nez dedans : ce qu'on vient de pretrainer, c'est ce qu'on appelle un modèle **base**. Si on lui donne "Quelle est la capitale de la France ?", il ne va pas répondre "Paris". Il va probablement continuer la phrase comme un exercice de grammaire scolaire, genre "... et quelle est la capitale de l'Allemagne ? Et de l'Italie ?". Un modèle base ne sait faire qu'une chose : deviner le mot suivant le plus probable, point barre. Pas de conversation, pas d'assistant, juste de l'autocomplete glorifié.

Le SFT (**S**upervised **F**ine-**T**uning), et derrière lui souvent du RLHF (qu'on ne fait pas ici, ce serait un article entier à part), c'est ce qui transforme cette machine à continuer du texte en un truc qui a l'air de "répondre". Concrètement : ChatGPT, Claude, Mistral, tout ce qu'on utilise au quotidien, c'est un modèle base qui a subi ce genre de traitement, en très, très, très poussé. Sans cette étape, pas de conversation. Juste un perroquet statistique qui continue les phrases.

Je voulais que mon modèle suive une instruction — genre "écris une histoire qui contient tels mots" — sans passer par Alpaca (le dataset d'instructions généraliste habituel), parce que son vocabulaire n'a rien à voir avec celui, volontairement riquiqui, de TinyStories. Un modèle qui n'a jamais vu le mot "photosynthèse" n'apprendra rien d'utile à essayer de répondre dessus.

Solution : construire le dataset d'instructions **à partir de TinyStories lui-même** (`src/prepare_sft_data.py`). Pour chaque histoire, je prends trois de ses propres mots au hasard, et je fabrique une paire `"Write a short story using these words: X, Y, Z." → l'histoire`. La loss est masquée sur la partie instruction, pour que le modèle n'apprenne qu'à produire l'histoire, pas à mémoriser la formulation de la consigne :

```python
def build_example(tokenizer, story_text, pad_id, eos_id):
    keywords = pick_keywords(story_text)  # 3 mots pris au hasard dans l'histoire
    prompt = f"Write a short story using these words: {', '.join(keywords)}.\n\n"

    prompt_ids = tokenizer.encode(prompt).ids
    story_ids = tokenizer.encode(story_text).ids + [eos_id]

    ids = (prompt_ids + story_ids)[:BLOCK_SIZE]
    n_prompt = min(len(prompt_ids), len(ids))
    pad_len = BLOCK_SIZE - len(ids)
    ids = ids + [pad_id] * pad_len

    labels = [IGNORE_INDEX] * BLOCK_SIZE
    for i in range(BLOCK_SIZE - 1):
        next_pos = i + 1
        if next_pos >= n_prompt and ids[next_pos] != pad_id:
            labels[i] = ids[next_pos]

    return ids, labels
```

```bash
docker run --rm --gpus '"device=0"' \
  -v $(pwd)/src:/workspace/src -v $(pwd)/data:/workspace/data \
  -w /workspace/src \
  llm-from-scratch \
  python prepare_sft_data.py
# -> 100 000 exemples, data/sft_input_ids.npy + data/sft_labels.npy

docker run -d --name llm-sft --gpus '"device=0"' \
  -v $(pwd)/src:/workspace/src -v $(pwd)/data:/workspace/data -v $(pwd)/checkpoints:/workspace/checkpoints \
  -w /workspace/src \
  llm-from-scratch \
  python train_sft.py --max_iters 3000 --eval_interval 200
```

Premier run lancé. Et là, alerte rouge : la val loss s'effondre à **0.0001** en quelques centaines d'itérations. Pour un modèle de langage, une loss aussi basse ne veut jamais rien dire de bon — soit il a mémorisé tout le dataset par cœur (peu probable en si peu de steps), soit il y a un bug quelque part. Je génère une histoire avec le checkpoint tout frais pour vérifier : des lignes vides, puis une boucle infinie de guillemets. Rien d'exploitable, même en donnant au modèle le prompt *exact* d'un exemple d'entraînement. Merde. Ça sentait le bug à plein nez, pas le miracle.

En creusant, la loss que je mesurais n'était même pas celle que je croyais mesurer. La toute première version de `build_example` faisait ça, au lieu du bloc `labels = ...` montré au-dessus :

```python
labels = list(ids)  # copie à l'identique, sans décalage — le bug
for i in range(n_prompt):
    labels[i] = IGNORE_INDEX
```

Une copie **à l'identique** de la séquence d'entrée, sans décalage d'un cran. Sauf que `model.forward()` ne décale rien tout seul : c'est à l'appelant de fournir des cibles déjà alignées sur "la sortie à la position i doit prédire le token à la position i+1" (exactement ce que fait `get_batch` dans le pretraining, via le `+1` sur l'indice de départ du tableau memmap). Résultat : je demandais au modèle de prédire, à chaque position, **le token qu'il venait tout juste de recevoir en entrée** à cette même position. Une tâche de copie complètement débile, presque résoluble par l'embedding seul vu que l'entrée et la sortie partagent leurs poids (*weight tying*, vu plus haut). D'où la loss ridiculement basse. D'où le modèle qui ne savait plus rien branler une fois livré à lui-même en génération.

Le genre de bête erreur qui ne saute pas aux yeux tant qu'on n'a pas comparé l'argmax et la vraie cible sur un exemple précis. Fix : le bloc `labels[i] = ids[next_pos]` montré plus haut, régénération du dataset, et deuxième run, mêmes commandes.

![Courbe de loss du SFT](/img/llm-scratch-sft-loss.png)

Cette fois la val loss oscille entre 1.57 et 1.83 — bruitée (l'éval ne tourne que sur un seul batch à chaque fois, contrairement au pretraining), mais dans la même fourchette que la loss finale du pretraining. Exactement ce qu'on veut voir : le modèle n'apprend pas une tâche disjointe, il apprend juste un format en plus par-dessus ce qu'il savait déjà faire.

Test sur un prompt **jamais vu à l'entraînement** :

```bash
docker run --rm --gpus '"device=0"' \
  -v $(pwd)/src:/workspace/src -v $(pwd)/data:/workspace/data -v $(pwd)/checkpoints:/workspace/checkpoints \
  -w /workspace/src \
  llm-from-scratch \
  python sample.py --checkpoint sft_best.pt \
    --prompt "Write a short story using these words: dog, ball, park.

"
```

> **Write a short story using these words: dog, ball, park.**
>
> *One day, a little dog named Max went for a walk. He saw a big, red ball in the park. Max wanted to play with the ball, so he took it home. Max was very happy. [...]*

Les trois mots imposés sont bien là. Sur un deuxième essai (`mountain, brave, castle`), le modèle utilise "brave" et "castle" mais zappe complètement "mountain", parti sur une histoire de chevalier et de dragon à la place. Suivi d'instruction réel, mais imparfait — cohérent pour un modèle de 29,5M de paramètres, et je préfère montrer ce résultat-là plutôt que de ne garder que le premier essai qui marchait pile poil.

---

## La quantization (PTQ)

Encore une parenthèse pour ceux qui utilisent des LLM en local sans trop savoir ce qui se passe sous le capot : les poids d'un modèle, c'est juste des milliards de nombres à virgule. Plus la précision de ces nombres est haute (32 bits, 16 bits...), plus le fichier est gros et plus il faut de VRAM pour le charger. La **quantization**, c'est réduire cette précision (8 bits, 4 bits...) pour diviser la taille du modèle par 2, 3, voire 4, quitte à perdre un chouïa de finesse sur chaque poids individuel. C'est littéralement ce qui permet à Ollama de faire tourner un modèle de plusieurs milliards de paramètres sur un laptop sans qu'il se transforme en grille-pain.

Dernière étape donc : exporter en **GGUF** (le format de fichier de `llama.cpp`, poids + métadonnées + tokenizer dans un seul fichier prêt à l'inférence) et quantifier. Sauf que notre architecture maison n'est évidemment pas une architecture que `llama.cpp` connaît de base — il a fallu la faire passer pour ce qu'elle est vraiment : un GPT-2 avec un autre nom.

Il se trouve que notre transformer colle presque tensor pour tensor à la convention `GPT2LMHeadModel` de HuggingFace (`wte`, `wpe`, `h.{i}.attn.c_attn`, `h.{i}.mlp.c_fc`, etc.), à un détail près : GPT-2 stocke ses poids linéaires **transposés** (héritage de l'implémentation `Conv1D` d'OpenAI). Un script d'export (`src/export_hf.py`) remappe notre `state_dict` vers cette convention, transpose ce qu'il faut, et écrit un `config.json` + `model.safetensors` + le tokenizer — un faux dossier HuggingFace, quoi. L'essentiel :

```python
hf_sd = {}
hf_sd["wte.weight"] = sd["token_emb.weight"].contiguous()
hf_sd["wpe.weight"] = sd["pos_emb.weight"].contiguous()
# lm_head est tied à wte -> pas réécrit, safetensors refuse d'écrire
# deux fois le même stockage mémoire.

for i in range(config.n_layer):
    p, h = f"blocks.{i}.", f"h.{i}."
    hf_sd[h + "attn.c_attn.weight"] = sd[p + "attn.qkv_proj.weight"].t().contiguous()
    hf_sd[h + "attn.c_proj.weight"] = sd[p + "attn.out_proj.weight"].t().contiguous()
    hf_sd[h + "mlp.c_fc.weight"]    = sd[p + "mlp.fc.weight"].t().contiguous()
    hf_sd[h + "mlp.c_proj.weight"]  = sd[p + "mlp.proj.weight"].t().contiguous()
    # + les biais et les LayerNorm, sans transposition (vecteurs 1D)

save_file(hf_sd, out_dir / "model.safetensors")
```

```bash
docker run --rm --gpus '"device=0"' \
  -v $(pwd)/src:/workspace/src -v $(pwd)/data:/workspace/data \
  -v $(pwd)/checkpoints:/workspace/checkpoints -v $(pwd)/gguf/hf_export:/workspace/hf_export \
  -w /workspace/src \
  llm-from-scratch \
  python export_hf.py --checkpoint sft_best.pt
```

Deuxième mur : le convertisseur GGUF de `llama.cpp` (déjà compilé bare-metal sur la machine depuis le post précédent) refuse notre tokenizer, parce que sa pré-tokenisation BPE ne correspond à aucune signature connue dans sa table de hash codée en dur. Logique, vu qu'on l'a entraîné nous-mêmes — jamais catalogué nulle part. Comme c'est fonctionnellement exactement le même schéma que celui de GPT-2 (`ByteLevel` BPE), un petit wrapper (`src/run_convert.py`) monkeypatch la fonction responsable pour forcer la valeur `"gpt-2"`, plutôt que de laisser le convertisseur deviner (et se vautrer) — sans toucher au checkout `llama.cpp` lui-même, partagé avec l'autre post :

```python
import sys
sys.path.insert(0, "/llama.cpp")
sys.path.insert(0, "/llama.cpp/gguf-py")

from conversion.base import TextModel
TextModel.get_vocab_base_pre = lambda self, tokenizer: "gpt-2"

# ... puis on appelle le main() de convert_hf_to_gguf.py normalement
```

Le convertisseur tourne sous Python, et le host est en Python 3.14 (Ubuntu 26.04) — trop récent pour `numpy==1.26.4` épinglé dans ses requirements. Plutôt que de patcher le host, la conversion tourne dans un petit conteneur Python 3.11 à part (`docker/Dockerfile.convert`) qui monte le repo `llama.cpp` existant en lecture seule :

```dockerfile
FROM python:3.11-slim
RUN pip install --no-cache-dir torch --index-url https://download.pytorch.org/whl/cpu
RUN pip install --no-cache-dir numpy~=1.26.4 sentencepiece "transformers==4.57.6" "protobuf>=4.21.0,<5.0.0"
WORKDIR /workspace
```

```bash
docker build -t llm-gguf-convert -f docker/Dockerfile.convert .

docker run --rm \
  -v /root/turing/llama.cpp:/llama.cpp:ro \
  -v $(pwd)/src:/workspace/src:ro \
  -v $(pwd)/gguf/hf_export:/workspace/hf_export:ro \
  -v $(pwd)/gguf/out:/workspace/out \
  -w /workspace \
  llm-gguf-convert \
  python /workspace/src/run_convert.py /workspace/hf_export \
    --outfile /workspace/out/tinystories-30m-f16.gguf --outtype f16
```

> P.S. Pendant cette étape, tentative de tuer proprement le serveur d'inférence resté ouvert avec `pkill -f llama-server` — sauf que `-f` matche la ligne de commande *complète*, y compris celle du shell qui exécute la commande `pkill -f llama-server` elle-même. Résultat : je me suis coupé ma propre session SSH en pleine manip, comme un débile. `pkill -x llama-server` (nom de process exact, pas la cmdline) a réglé le truc.

Une fois ces deux murs passés, la conversion tombe toute seule :

```
tinystories-30m-f16.gguf : 59,7 Mo
```

Puis `llama-quantize` (aussi compilé bare-metal) fait le reste :

```bash
llama-quantize tinystories-30m-f16.gguf tinystories-30m-q8_0.gguf Q8_0
llama-quantize tinystories-30m-f16.gguf tinystories-30m-q4_k_m.gguf Q4_K_M
```

| Format | Taille | vs F16 |
|---|---|---|
| F16 | 59,7 Mo | — |
| Q8_0 | 32,2 Mo | -46% |
| Q4_K_M | 20,5 Mo | -66% |

Et côté vitesse, sur la T10 :

```bash
CUDA_VISIBLE_DEVICES=0 llama-bench \
  -m tinystories-30m-f16.gguf -m tinystories-30m-q8_0.gguf -m tinystories-30m-q4_k_m.gguf \
  -p 128 -n 128 -ngl 999
```

| Format | Génération |
|---|---|
| F16 | 1 161 t/s |
| Q8_0 | 1 662 t/s |
| Q4_K_M | 1 731 t/s |

Le Q4_K_M est non seulement 3× plus léger, mais aussi **49% plus rapide** que le F16 pour générer du texte. Ça peut surprendre — moins de bits, ça devrait vouloir dire plus d'approximations, pas plus de vitesse — mais la génération token par token est limitée par la bande passante mémoire, pas par le calcul brut : moins de données à lire par token, plus c'est rapide. Et à l'usage, sur un modèle de cette taille, aucune perte de qualité perceptible entre les trois formats.

Le déploiement final, celui qui tourne réellement en pratique (`llama-cli` s'est révélé capricieux sur cette version de `llama.cpp`, `llama-server` + une requête HTTP fait le café sans souci, même méthode que dans le post précédent) :

```bash
CUDA_VISIBLE_DEVICES=0 llama-server --host 127.0.0.1 --port 8099 \
  -m tinystories-30m-q4_k_m.gguf -ngl 999 -c 256

curl http://127.0.0.1:8099/completion -H "Content-Type: application/json" -d '{
  "prompt": "Write a short story using these words: sun, garden, happy.\n\n",
  "n_predict": 120, "temperature": 0.8
}'
```

---

## Terminologie

| Terme | Définition rapide |
|---|---|
| **Token / Tokenizer** | Un LLM ne lit pas du texte brut mais des tokens (mots, bouts de mots, caractères). Le tokenizer fait ce découpage. |
| **BPE** (Byte Pair Encoding) | Algorithme de tokenisation qui fusionne itérativement les paires de caractères/tokens les plus fréquentes. |
| **Loss** | Une note qui résume à quel point le modèle se plante en moyenne à chaque prédiction. Plus c'est bas, mieux c'est. |
| **Pretraining** | Apprentissage auto-supervisé (prédiction du prochain token) sur un gros corpus brut. Donne un modèle "base", qui continue du texte mais ne "répond" à rien. |
| **SFT** (Supervised Fine-Tuning) | Fine-tuning sur des paires instruction/réponse, pour transformer un modèle base en quelque chose qui suit des consignes — la brique de base de tout "assistant" IA. |
| **PTQ** (Post-Training Quantization) | Réduction de la précision des poids (ici fp16 → Q8_0/Q4_K_M) après coup, sans réentraînement, pour réduire taille et coût mémoire. |
| **GGUF** | Format de fichier utilisé par `llama.cpp` pour stocker un modèle (poids + métadonnées + tokenizer) prêt à l'inférence. |
| **Weight tying** | Partage des poids entre l'embedding d'entrée et la couche de sortie du modèle. |

---

## Conclusion

Le pipeline complet — dataset, pretraining, SFT, quantization — tient sur une carte graphique qui, il y a quelques années à peine, servait à streamer du Fortnite à des joueurs qui n'ont probablement jamais soupçonné qu'elle finirait sa carrière à écrire des histoires pour enfants.

Est-ce que ça fait de moi le prochain Anthropic ? Évidemment pas, et c'est bien pour ça que le titre est une blague. Soyons clairs sur ce qui vient d'être fait : c'est un **jouet**. 29,5 millions de paramètres, un vocabulaire de 8192 tokens, un domaine volontairement riquiqui (des histoires pour enfants), le tout sur un seul GPU en moins de deux heures. Ce n'est ni la taille, ni l'ambition, ni sérieusement le sujet — c'est de l'exploration et du plaisir de comprendre, pas une tentative de rivaliser avec quoi que ce soit qui existe déjà.

Pour donner une idée de l'écart réel : Meta a publié les chiffres d'entraînement de [Llama 3](https://ai.meta.com/blog/meta-llama-3/) — plus de **15 000 milliards de tokens**, sur deux clusters de **24 000 GPU H100** tournant en parallèle. Rien que côté données, ça fait plus de 32 000 fois le corpus TinyStories utilisé ici. Rien que côté matériel, 24 000 GPU contre 1. Les modèles qu'on utilise tous les jours (Claude, GPT, Llama, Mistral) ne sont pas juste "plus gros" que ce qu'on vient de construire, ils appartiennent à une catégorie de dépense complètement différente — data centers entiers, des mois d'entraînement, des équipes de recherche à temps plein. Rien de tout ça n'était le but ici.

Ce qui était le but : il n'y a plus grand mystère dans le fonctionnement d'un LLM une fois qu'on a écrit soi-même chaque étage, même en miniature. L'attention n'est plus une boîte noire, la SFT n'est plus une formule magique, et la quantization n'est plus juste une ligne de commande copiée depuis un README. La mécanique de fond est rigoureusement la même à toutes les échelles — c'est justement ce qui rend l'exercice utile pour comprendre les gros modèles, sans avoir besoin d'en construire un.

Le bug du label shift résume bien l'exercice à lui tout seul. Ce genre d'erreur, on ne la rencontre jamais en appelant `.from_pretrained()`. Elle n'existe que quand on a vraiment mis les mains dans le mécanisme — et c'est exactement pour ça que ça valait le coup de le faire au moins une fois.

*Cet article a été co-écrit avec Claude. Tout ce qui y est écrit peut donc contenir des erreurs, des raccourcis ou des approximations.*
