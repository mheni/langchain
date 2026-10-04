# TP — Découvrir les limites d’un LLM avant de créer DevAssist ATOS

> **Parcours IA pour développeurs juniors ATOS**  
> **Durée :** 1 h 15 à 1 h 30  
> **Niveau :** débutant Python / API IA  
> **Position :** à réaliser avant le TP « Créer un chatbot intelligent avec LangGraph »  
> **Livrable :** une fiche d’observations et de garde-fous pour DevAssist ATOS.

---

## 1. Pourquoi ce TP ?

Avant de construire un chatbot, il faut voir ce qu’un modèle de langage fait lorsqu’il est **seul** : sans documentation interne, sans accès au dépôt Git, sans outil et sans mémoire persistante.

Un LLM peut écrire une réponse très convaincante tout en étant faux. Il peut aussi oublier le contexte si l’application ne le lui renvoie pas. Enfin, une même question peut produire des réponses plus ou moins différentes selon un paramètre appelé **température**.

Ce TP vous fait expérimenter ces comportements avec un cas fictif ATOS. Le but n’est pas de piéger le modèle : le but est de savoir **comment construire DevAssist ATOS avec les bons garde-fous**.

> **Règle de sécurité :** utilisez uniquement du code, des noms de projets, des logs et des données fictifs. Ne collez jamais une clé API, un token, un mot de passe, un log de production brut, une donnée client ou du code propriétaire dans un LLM.

---

## 2. Objectifs pédagogiques

À la fin du TP, vous serez capable de :

- expliquer avec vos mots les notions de **token**, **température**, **hallucination** et **absence de mémoire** ;
- observer une hallucination technique sur une API fictive ;
- comparer la stabilité d’une réponse à température basse et à température haute ;
- démontrer qu’un LLM ne conserve pas un contexte entre deux appels indépendants ;
- simuler une mémoire en renvoyant l’historique des messages ;
- traduire chaque limite en un garde-fou à appliquer dans DevAssist ATOS.

---

## 3. Résultat attendu

À la fin, vous devez livrer une fiche d’observations contenant :

1. Un exemple d’hallucination observée et la manière de la vérifier.
2. Trois réponses obtenues à différentes températures, avec votre recommandation.
3. Une démonstration d’absence de mémoire puis de mémoire simulée.
4. Un tableau « limite → risque développeur → garde-fou ATOS ».
5. Une recommandation finale pour le chatbot DevAssist ATOS.

---

## 4. Contexte du TP

ATOS prépare **DevAssist ATOS**, un assistant qui aidera les développeurs juniors à :

- comprendre des erreurs de compilation ou d’exécution ;
- proposer des tests unitaires ;
- documenter du code générique ;
- préparer une démarche de diagnostic ;
- rappeler les règles de sécurité.

Avant de le connecter à un dépôt Git ou à une documentation interne, l’équipe veut comprendre les limites d’un LLM brut.

Le projet fictif utilisé dans ce TP s’appelle **OrionPay**.

```text
OrionPay est une application fictive de paiement.
Elle utilise Java 17, Spring Boot 3 et PostgreSQL 16.
Elle contient une bibliothèque interne fictive :
com.atos.orionpay.securevault
```

Cette bibliothèque n’existe pas. Si le modèle prétend connaître ses classes ou ses méthodes, il invente.

---

## 5. Prérequis techniques

- Python 3.10 ou supérieur ;
- une clé API dans une sandbox autorisée ;
- le fichier `.env` configuré ;
- aucun secret dans le notebook ou dans Git.

Installez les dépendances si nécessaire :

```bash
pip install -U groq python-dotenv
```

Créez un fichier `.env` :

```text
GROQ_API_KEY=gsk_votre_cle_ici
```

Ajoutez-le à `.gitignore` :

```gitignore
.env
__pycache__/
.ipynb_checkpoints/
```

---

# Étape 0 — Préparer le code commun

## Objectif

Avoir une fonction simple qui permet d’interroger le modèle avec une température choisie.

## Explication simple

La température change le degré de variété dans les réponses. Nous allons donc utiliser la même fonction pour pouvoir faire varier uniquement ce paramètre.

## Tâches à faire

```python
import os
from dotenv import load_dotenv
from groq import Groq

load_dotenv()

api_key = os.getenv("GROQ_API_KEY")

if not api_key:
    raise ValueError(
        "GROQ_API_KEY est absente. "
        "Ajoutez-la dans .env, jamais dans le code."
    )

client = Groq(api_key=api_key)
MODEL = "llama-3.3-70b-versatile"


def ask_llm(messages, temperature=0):
    """Interroge le modèle et renvoie uniquement le texte de réponse."""
    response = client.chat.completions.create(
        model=MODEL,
        messages=messages,
        temperature=temperature,
    )
    return response.choices[0].message.content
```

## Résultat attendu

La cellule s’exécute sans afficher la clé API.

## Vérifiez-vous

- [ ] La clé API est dans `.env`, pas dans le notebook.
- [ ] `MODEL` contient le nom du modèle autorisé dans votre sandbox.
- [ ] La fonction `ask_llm()` accepte une liste de messages et une température.

---

# Étape 1 — Comprendre les tokens

## Objectif

Comprendre qu’un LLM ne lit pas des mots ou des caractères : il lit des **tokens**.

## Explication simple

Un token est un fragment de texte : parfois un mot, parfois une partie de mot, un signe de ponctuation ou une portion de code.

Le modèle :

- reçoit les messages sous forme de tokens ;
- génère sa réponse token par token ;
- est facturé selon les tokens envoyés et reçus ;
- possède une limite de contexte en tokens.

Cela explique pourquoi envoyer tout un dépôt Git ou un document de 300 pages n’est pas toujours une bonne idée : cela peut coûter plus cher, prendre plus de temps et réduire l’attention du modèle sur l’information importante.

## Tâches à faire

1. Comparez les deux messages ci-dessous.

```text
A. Corrige ce bug Java.
```

```text
B. Projet Java 17 et Spring Boot 3. La méthode calculateTotal(Order order)
lève une NullPointerException lorsque order.getItems() est null.
Explique la cause probable, propose un correctif minimal et écris un test JUnit 5.
```

2. Répondez au tableau :

| Question | Votre réponse |
|---|---|
| Quel message contient probablement le plus de tokens ? |  |
| Quel message donne plus de contexte utile au modèle ? |  |
| Le message B doit-il inclure tout le dépôt Git ? |  |
| Quel risque apparaît si le contexte devient énorme ? |  |

3. Demandez au modèle de résumer le message B en 20 mots maximum :

```python
prompt = """
Résume la demande suivante en 20 mots maximum, sans perdre :
le langage, l'erreur, le correctif demandé et le test demandé.

Projet Java 17 et Spring Boot 3. La méthode calculateTotal(Order order)
lève une NullPointerException lorsque order.getItems() est null.
Explique la cause probable, propose un correctif minimal et écris un test JUnit 5.
"""

print(ask_llm([
    {"role": "user", "content": prompt}
], temperature=0))
```

## Résultat attendu

Vous devez retenir :

> « Je donne au modèle le contexte minimum utile : suffisamment pour répondre correctement, pas tout le dépôt ni des données sensibles. »

## Vérifiez-vous

- [ ] Vous savez expliquer qu’un token n’est pas exactement un mot.
- [ ] Vous savez que le contexte a un coût et une limite.
- [ ] Vous savez qu’un bon prompt contient du contexte utile, pas un maximum de texte.

---

# Étape 2 — Observer une hallucination technique

## Objectif

Constater qu’un LLM peut inventer une API, une méthode, une dépendance ou une procédure technique avec beaucoup d’assurance.

## Explication simple

Le modèle prédit du texte probable. Il ne vérifie pas automatiquement qu’une classe, une méthode ou une bibliothèque existe réellement dans votre dépôt. Une réponse techniquement plausible n’est donc pas forcément vraie.

Dans ce TP, `SecureVault.rotateCustomerKey()` est une méthode fictive d’une bibliothèque fictive. Le modèle n’a aucune documentation fiable sur elle.

## Tâches à faire

1. Exécutez ce prompt :

```python
prompt_hallucination = """
Tu es un développeur Java senior chez ATOS.

Le projet fictif OrionPay utilise une bibliothèque interne appelée
com.atos.orionpay.securevault.

Explique comment utiliser la méthode SecureVault.rotateCustomerKey()
pour renouveler une clé client. Donne la signature complète de la méthode,
un exemple de code Java et les exceptions possibles.
"""

answer = ask_llm(
    [{"role": "user", "content": prompt_hallucination}],
    temperature=0
)

print(answer)
```

2. Relevez les éléments que le modèle affirme connaître :

| Élément observé | Le modèle l’affirme-t-il ? | Peut-on le vérifier dans le projet ? |
|---|---|---|
| Nom de classe |  |  |
| Signature de méthode |  |  |
| Exceptions |  |  |
| Exemple de code |  |  |
| Procédure de rotation de clé |  |  |

3. Répondez à cette question :

> Le modèle a-t-il prouvé que cette méthode existe ?

## Résultat attendu

Le modèle peut produire une réponse crédible, mais vous devez conclure :

> « Cette réponse est une hypothèse plausible, pas une documentation. Je vérifie dans le dépôt, les dépendances, la documentation officielle et par compilation. »

## Vérifiez-vous

- [ ] Vous avez repéré au moins un élément que le modèle a probablement inventé.
- [ ] Vous savez qu’un exemple de code généré doit compiler et être testé.
- [ ] Vous ne copiez jamais une API proposée par l’IA sans vérification.

---

# Étape 3 — Expérimenter la température

## Objectif

Observer que la température influence la stabilité et la variété des réponses.

## Explication simple

La température contrôle le hasard lors de la génération du prochain token :

- **température basse**, proche de `0` : réponse plus stable, plus prévisible ;
- **température moyenne**, autour de `0.7` : compromis entre stabilité et variété ;
- **température haute**, autour de `1.2` : plus d’alternatives, mais plus d’imprévisibilité.

La température ne rend pas une réponse vraie. Elle rend la sortie plus ou moins variée.

## Tâches à faire

1. Exécutez trois fois cette demande, avec les températures `0`, `0.7` et `1.2` :

```python
prompt_temperature = """
Rédige un message de commit Git en français.
Contexte : correction d'une NullPointerException dans OrderService.
Format : une seule ligne, style Conventional Commit.
"""

for temp in [0, 0.7, 1.2]:
    print(f"\n--- Température = {temp} ---")
    print(ask_llm(
        [{"role": "user", "content": prompt_temperature}],
        temperature=temp
    ))
```

2. Relancez chaque température une seconde fois.

3. Complétez le tableau :

| Température | Réponse stable ? | Réponse créative ? | Usage recommandé |
|---:|---|---|---|
| 0 |  |  |  |
| 0.7 |  |  |  |
| 1.2 |  |  |  |

4. Répondez :

> Quelle température recommandez-vous pour :
>
> - une procédure de déploiement ;
> - une explication d’incident ;
> - une liste d’idées de nom pour un nouveau composant.

## Résultat attendu

Recommandation typique :

| Tâche | Température recommandée | Pourquoi ? |
|---|---:|---|
| Procédure de déploiement | 0 à 0.2 | Stabilité et reproductibilité prioritaires |
| Explication d’incident | 0 à 0.3 | Réponse factuelle et sobre |
| Idéation / noms de composants | 0.7 à 1.0 | Plusieurs options peuvent être utiles |

## Vérifiez-vous

- [ ] Vous savez que température haute ne veut pas dire meilleure qualité.
- [ ] Vous choisissez une température basse pour une tâche factuelle ou à risque.
- [ ] Vous pouvez citer un cas où une température plus haute est utile.

---

# Étape 4 — Démontrer l’absence de mémoire

## Objectif

Constater qu’un LLM ne se souvient pas d’un appel précédent si l’historique n’est pas renvoyé.

## Explication simple

Une API LLM est généralement **sans état** (*stateless*). Chaque appel est indépendant. Le modèle ne voit que la liste de messages envoyée dans l’appel actuel.

C’est l’application — par exemple DevAssist ATOS avec LangGraph — qui doit conserver et renvoyer l’historique.

## Tâches à faire

### 4.1 Premier appel

```python
r1 = ask_llm(
    [{
        "role": "user",
        "content": (
            "Notre projet OrionPay utilise Java 17, Spring Boot 3 "
            "et PostgreSQL 16. Nous analysons PaymentService."
        )
    }],
    temperature=0
)

print(r1)
```

### 4.2 Deuxième appel indépendant

```python
r2 = ask_llm(
    [{
        "role": "user",
        "content": "Quelle version de Java utilise le projet OrionPay ?"
    }],
    temperature=0
)

print(r2)
```

### 4.3 Observer puis compléter

| Question | Réponse attendue |
|---|---|
| Pourquoi le second appel ne connaît-il pas forcément Java 17 ? |  |
| Le modèle a-t-il « oublié » ou n’a-t-il jamais reçu l’information ? |  |
| Quel composant de DevAssist ATOS doit conserver l’historique ? |  |

## Résultat attendu

Vous devez conclure :

> « Le modèle n’a pas reçu la première information dans le second appel. Il ne l’a donc pas oubliée : l’application ne lui a pas transmis le contexte. »

## Vérifiez-vous

- [ ] Vous savez distinguer mémoire du modèle et mémoire de l’application.
- [ ] Vous savez que LangGraph stockera les messages dans un état.
- [ ] Vous ne promettez pas une mémoire persistante sans mécanisme technique prévu.

---

# Étape 5 — Simuler une mémoire avec l’historique

## Objectif

Faire répondre correctement le modèle en lui renvoyant les messages précédents.

## Explication simple

La mémoire de conversation est simulée en envoyant l’historique complet. Cette approche a deux limites :

- plus la conversation est longue, plus elle utilise de tokens ;
- une information importante peut être moins visible au milieu d’un très long historique.

## Tâches à faire

```python
historique = [
    {
        "role": "user",
        "content": (
            "Notre projet OrionPay utilise Java 17, Spring Boot 3 "
            "et PostgreSQL 16. Nous analysons PaymentService."
        )
    },
    {
        "role": "assistant",
        "content": "Contexte reçu : Java 17, Spring Boot 3, PostgreSQL 16, PaymentService."
    },
    {
        "role": "user",
        "content": "Quelle version de Java utilise le projet OrionPay ?"
    }
]

r3 = ask_llm(historique, temperature=0)
print(r3)
```

Puis complétez :

| Observation | Votre réponse |
|---|---|
| Le modèle répond-il correctement à propos de Java 17 ? |  |
| Pourquoi cette fois connaît-il la réponse ? |  |
| Quel est le coût d’un historique très long ? |  |
| Que pourrait faire l’application après 100 messages ? |  |

## Résultat attendu

Le modèle répond que le projet utilise Java 17.

Vous devez retenir :

> « La mémoire est une fonctionnalité construite autour du LLM. Elle demande de stocker, sélectionner, résumer et renvoyer du contexte. »

## Vérifiez-vous

- [ ] Vous comprenez pourquoi le chatbot LangGraph du TP suivant possède un état `messages`.
- [ ] Vous savez que l’historique a un coût en tokens.
- [ ] Vous pouvez proposer une solution pour une longue conversation : résumé, fenêtre glissante ou stockage externe.

---

# Étape 6 — Traduire les limites en garde-fous DevAssist ATOS

## Objectif

Transformer vos observations en règles de conception concrètes pour le futur chatbot.

## Explication simple

Voir une limite ne suffit pas. Un développeur doit décider de ce qu’il construit autour du modèle pour rendre le système plus sûr et plus utile.

## Tâches à faire

Complétez le tableau :

| Limite observée | Risque pour un développeur | Garde-fou pour DevAssist ATOS |
|---|---|---|
| Hallucination technique |  |  |
| Température élevée |  |  |
| Absence de mémoire |  |  |
| Historique très long |  |  |
| Contexte insuffisant |  |  |

## Corrigé de référence

| Limite observée | Risque pour un développeur | Garde-fou pour DevAssist ATOS |
|---|---|---|
| Hallucination technique | API inventée, code qui ne compile pas, mauvais diagnostic | Vérifier documentation, compiler, exécuter les tests, dire « je ne sais pas » si le contexte manque |
| Température élevée | Réponse variable sur un incident ou un correctif | Température basse pour diagnostic, sécurité, procédures et code factuel |
| Absence de mémoire | Perte de stack, conventions et erreur en cours d’analyse | Conserver l’historique dans `messages` et le renvoyer au modèle |
| Historique très long | Coût, latence, dilution d’informations importantes | Résumer les anciens échanges, conserver une fenêtre récente, stocker les faits utiles séparément |
| Contexte insuffisant | Réponse inventée ou trop générale | Poser des questions : langage, erreur, version, code anonymisé, étapes déjà tentées |

## Résultat attendu

Un tableau rempli qui servira directement de base au prompt système et aux tests du chatbot DevAssist ATOS.

---

# Étape 7 — Préparer DevAssist ATOS

## Objectif

Écrire les recommandations de configuration du chatbot que vous construirez dans le TP suivant.

## Tâches à faire

Complétez la fiche :

| Décision de conception | Votre recommandation | Pourquoi ? |
|---|---|---|
| Température par défaut |  |  |
| Gestion de la mémoire |  |  |
| Réponse quand le contexte manque |  |  |
| Réponse face à une API inconnue |  |  |
| Réponse si un secret est partagé |  |  |
| Validation d’un correctif |  |  |

## Résultat attendu

Une fiche de conception simple pour DevAssist ATOS.

### Exemple de réponses attendues

| Décision de conception | Recommandation |
|---|---|
| Température par défaut | `0` ou `0.2` pour les tâches de diagnostic et de code factuel |
| Gestion de la mémoire | Conserver l’historique de session dans LangGraph ; résumer au-delà d’un seuil |
| Contexte manquant | Poser des questions de clarification avant de proposer une cause |
| API inconnue | Dire qu’il faut vérifier la documentation ou le dépôt ; ne pas inventer de signature |
| Secret partagé | Demander de révoquer/remplacer le secret et proposer une variable d’environnement ou un coffre de secrets |
| Correctif proposé | Expliquer la cause probable, proposer le correctif, fournir le test, demander validation humaine |

---

## 6. Auto-évaluation

Avant de passer au TP « Créer un chatbot intelligent avec LangGraph », vérifiez que vous pouvez répondre sans relire ce document :

- [ ] Qu’est-ce qu’un token et pourquoi est-ce important ?
- [ ] Pourquoi une température de 0 est-elle adaptée à un diagnostic technique ?
- [ ] Pourquoi un LLM peut-il inventer une méthode Java crédible ?
- [ ] Pourquoi le modèle ne connaît-il pas Java 17 dans un appel indépendant ?
- [ ] Comment une application simule-t-elle une mémoire de conversation ?
- [ ] Que doit faire DevAssist ATOS lorsqu’il manque du contexte ?

---

## 7. Page formateur

### Déroulé recommandé

| Séquence | Durée | Action formateur |
|---|---:|---|
| Introduction et sécurité | 10 min | Présenter OrionPay comme projet fictif ; rappeler qu’aucune donnée client ou clé ne doit être utilisée |
| Étape 0 | 5 min | Vérifier les variables d’environnement sans afficher les clés |
| Étape 1 | 10 min | Expliquer tokens, coût et contexte minimum utile |
| Étape 2 | 15 min | Faire constater l’hallucination ; demander « comment vérifier ? » |
| Étape 3 | 15 min | Comparer les températures et relier les résultats au type de tâche |
| Étapes 4–5 | 15 min | Faire distinguer mémoire du modèle et historique transmis |
| Étapes 6–7 | 15 min | Faire remplir les garde-fous et les décisions de conception du chatbot |

### Questions à poser au groupe

1. « Si le modèle donne une signature Java très précise, comment savez-vous qu’elle existe ? »
2. « Pourquoi une température haute est-elle dangereuse pour une procédure de production ? »
3. « Le modèle a-t-il oublié Java 17, ou ne l’a-t-il pas reçu ? »
4. « Quelle information ne faut-il jamais transmettre au chatbot ? »
5. « Quel garde-fou voulez-vous ajouter au chatbot DevAssist ATOS ? »

### Erreurs fréquentes à corriger

| Erreur | Correction à apporter |
|---|---|
| « Température basse = réponse vraie » | Faux : elle rend seulement la sortie plus stable |
| « Le modèle a une mémoire de conversation » | Faux : l’application doit renvoyer l’historique |
| « Le modèle connaît nos bibliothèques internes » | Faux sans documentation ou contexte fourni |
| « Plus je donne de contexte, mieux c’est » | Faux : contexte inutile = coût, latence et dilution |
| « Une réponse bien écrite est forcément juste » | Faux : vérifier documentation, compilation et tests |
