# TP — Construire un assistant RAG pour DevAssist ATOS

> **Parcours IA pour développeurs juniors ATOS**  
> **Durée :** 1 h 45 à 2 h 15  
> **Niveau :** Python débutant, LLM et prompt engineering fondamentaux  
> **Prérequis :** TP Fondamentaux LLM + TP Prompt Engineering  
> **Livrable :** un prototype RAG local qui répond à partir d’une documentation fictive OrionPay, cite ses sources et reconnaît les questions hors périmètre.

---

## 1. Pourquoi ce TP ?

Dans le TP sur les fondamentaux, vous avez vu qu’un LLM peut inventer une réponse plausible lorsqu’il ne connaît pas une bibliothèque, une API ou une procédure.

Dans le TP sur le prompt engineering, vous avez appris à mieux formuler les demandes. Mais un prompt bien écrit ne peut pas donner au modèle une connaissance absente de son entraînement.

Pour aider un développeur sur un projet réel, DevAssist ATOS doit pouvoir s’appuyer sur une documentation contrôlée : README, guides de contribution, conventions de code, procédures d’incident, API internes validées ou documentation technique publique.

C’est le rôle d’un système **RAG** — *Retrieval Augmented Generation* :

1. il recherche les extraits de documentation les plus utiles ;
2. il les donne au LLM comme contexte ;
3. le LLM rédige une réponse ancrée dans ces extraits ;
4. si l’information n’existe pas, il doit le dire au lieu de l’inventer.

> **Règle de sécurité :** le corpus de ce TP est fictif. Dans une mission réelle, indexez uniquement des documents autorisés. Ne placez pas de secrets, clés API, données clients, exports de production ou code propriétaire non validé dans une base RAG.

---

## 2. Objectifs pédagogiques

À la fin de ce TP, vous serez capable de :

- expliquer les deux phases d’un RAG : **recherche** (*retrieval*) puis **génération** (*generation*) ;
- expliquer les notions d’**embedding**, **chunk**, **base vectorielle** et **retriever** ;
- découper une documentation en fragments utiles ;
- créer une base vectorielle locale avec FAISS ;
- interroger cette base en recherche sémantique ;
- construire une chaîne RAG avec un LLM ;
- imposer un comportement de repli lorsque la réponse n’est pas dans le corpus ;
- afficher les sources utilisées par la réponse ;
- comparer une réponse LLM « nue » à une réponse RAG ;
- tester et enrichir un corpus documentaire.

---

## 3. Résultat attendu

À la fin du TP, vous devez pouvoir démontrer :

1. Un fichier de documentation fictive `orionpay_docs.txt`.
2. Une base vectorielle FAISS créée à partir de ce corpus.
3. Une recherche sémantique qui retrouve les bons extraits.
4. Un assistant RAG qui répond à partir des sources trouvées.
5. Une réponse de repli sur une question absente du corpus.
6. Une réponse qui affiche le titre ou l’extrait source utilisé.
7. Deux nouveaux documents ajoutés puis correctement retrouvés après réindexation.

---

## 4. Contexte métier : DevAssist ATOS et OrionPay

ATOS maintient l’application fictive **OrionPay** :

- Java 17 ;
- Spring Boot 3 ;
- PostgreSQL 16 ;
- API de paiement et de gestion de commandes ;
- équipe de développeurs juniors accompagnée par des seniors.

DevAssist ATOS doit aider les juniors à retrouver :

- les conventions de code ;
- les commandes de lancement et de test ;
- la procédure de diagnostic d’un incident ;
- la politique de gestion des secrets ;
- le fonctionnement d’API documentées.

Il ne doit pas inventer une réponse si la documentation ne contient pas l’information.

---

## 5. Concepts à connaître avant de commencer

| Concept | Explication simple | Pourquoi c’est utile ? |
|---|---|---|
| **RAG** | Recherche des documents puis génère une réponse avec ces documents | Réduit les réponses inventées sur un projet spécifique |
| **Chunk** | Petit fragment d’un document | On ne donne pas tout le corpus au modèle ; on donne les fragments utiles |
| **Embedding** | Vecteur numérique qui représente le sens d’un texte | Permet de chercher par sens, pas seulement par mot exact |
| **Base vectorielle** | Stockage qui sait retrouver les vecteurs proches | Accélère la recherche d’extraits pertinents |
| **Retriever** | Composant qui prend une question et retourne les chunks pertinents | Fait le lien entre question et documentation |
| **Contexte RAG** | Chunks retrouvés, ajoutés au prompt du LLM | Ancre la réponse dans une source contrôlée |
| **Repli** | Réponse quand aucune source fiable n’est disponible | Évite l’hallucination hors périmètre |

### Schéma à retenir

```text
Question du développeur
        ↓
Embedding de la question
        ↓
Recherche dans la base vectorielle
        ↓
Chunks les plus pertinents
        ↓
Prompt avec contexte + question
        ↓
LLM
        ↓
Réponse + sources ou message de repli
```

---

## 6. Prérequis techniques

Installez les dépendances :

```bash
pip install -U \
  groq \
  python-dotenv \
  langchain \
  langchain-groq \
  langchain-huggingface \
  langchain-community \
  faiss-cpu \
  "sentence-transformers>=5.2.0"
```

> Les modèles d’embeddings sont exécutés localement avec HuggingFace dans ce TP. Le LLM sert à générer la réponse. Dans une architecture entreprise, les choix de modèles, de stockage et de région doivent respecter les règles ATOS et celles du client.

### Configuration des secrets

Fichier `.env` :

```text
GROQ_API_KEY=gsk_votre_cle_ici
```

Fichier `.gitignore` :

```gitignore
.env
__pycache__/
.ipynb_checkpoints/
faiss_index/
```

---

# Étape 0 — Créer le corpus documentaire OrionPay

## Objectif

Créer une documentation fictive, contrôlée et sans donnée sensible, qui servira de source de vérité à DevAssist ATOS.

## Explication simple

Un RAG répond uniquement aussi bien que les documents qui lui sont fournis. Si une information est absente, obsolète ou ambiguë dans le corpus, le chatbot ne peut pas inventer une réponse fiable.

Dans ce TP, le corpus est volontairement petit. Dans un vrai projet, il peut contenir des README, ADR, documentation d’API, runbooks et guides de contribution validés.

## Tâches à faire

Créez un fichier nommé `orionpay_docs.txt` avec ce contenu :

```text
[DOC-01] OrionPay — Préparer l'environnement de développement
OrionPay utilise Java 17, Maven 3.9, Spring Boot 3 et PostgreSQL 16.
Pour lancer l'application localement, exécutez : mvn spring-boot:run.
Pour lancer les tests unitaires, exécutez : mvn test.
La configuration locale utilise le profil Spring "local".
Aucun secret ne doit être placé dans application.yml : utilisez les variables d'environnement.

[DOC-02] OrionPay — Convention de gestion des erreurs
Les API OrionPay retournent un code HTTP 400 pour une requête invalide,
401 pour une authentification absente ou invalide, 403 pour un accès refusé,
404 pour une ressource introuvable et 500 pour une erreur interne inattendue.
Les contrôleurs ne doivent pas exposer la stack trace dans la réponse HTTP.
Les erreurs métier doivent être traitées avec des exceptions dédiées.

[DOC-03] OrionPay — Diagnostic d'une NullPointerException
Avant de modifier le code, reproduisez le problème avec un test.
Lisez la ligne exacte de la stack trace et identifiez la référence nulle.
Vérifiez les entrées de la méthode, les retours de repository et les collections.
Ajoutez un garde-fou explicite ou une validation lorsque le cas null est possible.
Après correction, exécutez mvn test et relisez le diff avant commit.

[DOC-04] OrionPay — Politique de secrets
Les mots de passe, clés API, tokens OAuth et chaînes de connexion sont des secrets.
Ils ne doivent jamais être présents dans Git, dans un ticket, dans un prompt ou dans un log applicatif.
Pour le développement local, chargez les secrets depuis des variables d'environnement.
En cas de secret exposé, révoquez-le ou remplacez-le immédiatement,
puis informez le responsable technique ou sécurité de la mission.

[DOC-05] OrionPay — API de création de commande
La route POST /v1/orders crée une commande.
Le corps JSON doit contenir customerReference et items.
Une commande sans item est invalide et retourne HTTP 400.
Une commande créée retourne HTTP 201 avec son identifiant et son statut.
Les données client réelles ne doivent pas être utilisées dans les tests automatisés.

[DOC-06] OrionPay — Revue de code avant pull request
Avant une pull request, exécutez les tests unitaires et le linter.
Vérifiez qu'aucun secret, fichier .env ou donnée de test réelle n'est inclus.
Ajoutez une description courte : objectif, impact, tests exécutés et risque identifié.
Un correctif généré avec une IA doit être relu et testé par le développeur.
```

## Résultat attendu

Un fichier `orionpay_docs.txt` contenant six blocs documentaires fictifs.

## Vérifiez-vous

- [ ] Le fichier est enregistré en UTF-8.
- [ ] Il ne contient aucun secret réel.
- [ ] Chaque bloc commence par un identifiant `[DOC-XX]`.
- [ ] Les documents répondent à des questions de développement concrètes.

---

# Étape 1 — Charger et découper les documents

## Objectif

Charger la documentation et la découper en **chunks** utilisables pour la recherche.

## Explication simple

Un LLM ne doit pas recevoir toute la documentation à chaque question. On découpe les documents en fragments. Chaque fragment doit être :

- assez petit pour être précis ;
- assez grand pour garder le contexte ;
- cohérent sur un même sujet.

Dans ce TP, un bloc `[DOC-XX]` est déjà presque un chunk. Le séparateur double saut de ligne préserve donc naturellement les six documents.

## Tâches à faire

```python
from langchain_community.document_loaders import TextLoader
from langchain_text_splitters import CharacterTextSplitter

loader = TextLoader("orionpay_docs.txt", encoding="utf-8")
documents = loader.load()

text_splitter = CharacterTextSplitter(
    separator="\n\n",
    chunk_size=700,
    chunk_overlap=80,
)

chunks = text_splitter.split_documents(documents)

print(f"Nombre de documents chargés : {len(documents)}")
print(f"Nombre de chunks générés : {len(chunks)}")

for i, chunk in enumerate(chunks):
    print(f"\n--- Chunk {i} ---")
    print(chunk.page_content[:250])
```

## Résultat attendu

Vous obtenez environ six chunks, un par thème principal.

## Tâches d’observation

Complétez le tableau :

| Question | Votre réponse |
|---|---|
| Combien de chunks ont été générés ? |  |
| Quel chunk contient `mvn test` ? |  |
| Quel chunk parle des secrets ? |  |
| Pourquoi ne faut-il pas faire un chunk unique avec tout le fichier ? |  |
| Quel risque existe si un chunk est trop petit ? |  |

## Vérifiez-vous

- [ ] Les chunks sont lisibles et cohérents.
- [ ] Le chunk sur les secrets contient bien la consigne de révocation.
- [ ] Le chunk sur les erreurs ne mélange pas une procédure de déploiement sans lien.

---

# Étape 2 — Créer les embeddings et la base vectorielle

## Objectif

Transformer les chunks en vecteurs et les stocker dans une base FAISS locale.

## Explication simple

Un embedding est une liste de nombres qui représente le sens d’un texte. Deux textes proches en sens auront des vecteurs proches.

Exemple :

- « Comment exécuter les tests ? »
- « Quelle commande Maven lance les tests ? »

Les mots sont différents, mais le sens est proche. La recherche vectorielle peut donc retrouver le chunk contenant `mvn test` même si la question ne contient pas exactement ces mots.

## Tâches à faire

```python
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_community.vectorstores import FAISS

embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2"
)

vector_store = FAISS.from_documents(chunks, embeddings)

print("Base vectorielle FAISS créée.")
```

> Le premier lancement peut télécharger le modèle d’embeddings. Attendez la fin du téléchargement avant de conclure à une erreur.

## Résultat attendu

```text
Base vectorielle FAISS créée.
```

## Vérifiez-vous

- [ ] Vous savez qu’un embedding n’est pas une réponse du LLM.
- [ ] Vous savez que FAISS stocke des vecteurs et effectue la recherche de similarité.
- [ ] Vous savez que le modèle d’embedding est différent du modèle génératif.

---

# Étape 3 — Tester la recherche sémantique seule

## Objectif

Vérifier que la recherche retrouve les bons documents **avant** d’ajouter le LLM.

## Explication simple

Un RAG comporte deux parties : recherche puis génération. Si la recherche retourne le mauvais document, un bon LLM ne peut pas fabriquer une réponse fiable. Il faut donc tester la recherche seule.

## Tâches à faire

```python
questions = [
    "Quelle commande permet de lancer les tests unitaires ?",
    "Que faut-il faire si une clé API apparaît dans un ticket ?",
    "Quel code HTTP doit être retourné pour une ressource introuvable ?",
    "Comment diagnostiquer une NullPointerException ?"
]

for question in questions:
    results = vector_store.similarity_search(question, k=2)

    print("\nQUESTION :", question)
    for i, doc in enumerate(results, start=1):
        print(f"\n--- Résultat {i} ---")
        print(doc.page_content[:350])
```

## Résultat attendu

| Question | Source attendue |
|---|---|
| Commande des tests | `DOC-01` |
| Clé API exposée | `DOC-04` |
| Ressource introuvable | `DOC-02` |
| NullPointerException | `DOC-03` |

## Tâches d’observation

| Question testée | Meilleur document retourné | Pertinent ? Oui / Non | Pourquoi ? |
|---|---|---|---|
| Tests Maven |  |  |  |
| Clé API |  |  |  |
| Code 404 |  |  |  |
| NullPointerException |  |  |  |

## Vérifiez-vous

- [ ] Le meilleur résultat est pertinent pour chacune des quatre questions.
- [ ] Vous testez le retriever avant le LLM.
- [ ] Vous savez expliquer qu’un mauvais chunk récupéré peut dégrader la réponse finale.

---

# Étape 4 — Construire la chaîne RAG

## Objectif

Créer un assistant qui répond uniquement à partir des documents retrouvés.

## Explication simple

Le retriever cherche les documents utiles. Ensuite, leurs contenus sont ajoutés au prompt du LLM.

La consigne la plus importante est le comportement de repli :

> « Si l’information n’est pas dans les sources, dis que tu ne la possèdes pas. N’invente rien. »

Sans cette instruction, le modèle peut recommencer à compléter avec ses connaissances générales, même si vous avez ajouté un RAG.

## Tâches à faire

```python
import os
from langchain_groq import ChatGroq
from langchain_core.prompts import ChatPromptTemplate
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain.chains import create_retrieval_chain

llm = ChatGroq(
    groq_api_key=os.environ.get("GROQ_API_KEY"),
    model="llama-3.3-70b-versatile",
    temperature=0,
)

prompt = ChatPromptTemplate.from_template(
    """
Tu es DevAssist ATOS, un assistant pédagogique pour développeurs juniors.

Réponds uniquement à partir du contexte OrionPay fourni ci-dessous.

Règles obligatoires :
- si l'information n'est pas présente dans le contexte, réponds exactement :
  "Je ne dispose pas de cette information dans la documentation OrionPay fournie."
- n'invente jamais une commande, une API ou une procédure interne ;
- réponds en français ;
- termine par "Sources :" suivi des identifiants [DOC-XX] utilisés ;
- rappelle une validation humaine lorsqu'un correctif est proposé.

Contexte :
{context}

Question du développeur :
{input}
"""
)

document_chain = create_stuff_documents_chain(llm, prompt)
retriever = vector_store.as_retriever(search_kwargs={"k": 2})
retrieval_chain = create_retrieval_chain(retriever, document_chain)
```

## Résultat attendu

La chaîne RAG est construite sans erreur.

## Vérifiez-vous

- [ ] La température est basse (`0`) car le sujet est factuel.
- [ ] Le prompt impose une réponse de repli.
- [ ] Le prompt impose l’affichage des sources.
- [ ] Le retriever retourne au maximum deux chunks (`k=2`).

---

# Étape 5 — Interroger le chatbot RAG et afficher les sources

## Objectif

Obtenir une réponse ancrée dans le corpus, avec les documents récupérés.

## Explication simple

La réponse du LLM est utile, mais les sources permettent au développeur de vérifier. Dans une application réelle, on afficherait des liens vers les documents, leurs identifiants ou les passages concernés.

## Tâches à faire

```python
questions = [
    "Quelle commande Maven lance les tests unitaires ?",
    "Quelle est la procédure après exposition d'une clé API ?",
    "Quel code HTTP utiliser si une commande n'existe pas ?",
    "Comment traiter une NullPointerException dans OrionPay ?"
]

for question in questions:
    response = retrieval_chain.invoke({"input": question})

    print("\nQUESTION :", question)
    print("RÉPONSE :", response["answer"])

    print("SOURCES RÉCUPÉRÉES :")
    for doc in response["context"]:
        print(doc.page_content.split("\n")[0])

    print("=" * 70)
```

## Résultat attendu

| Question | Élément attendu dans la réponse | Source attendue |
|---|---|---|
| Tests Maven | `mvn test` | `DOC-01` |
| Clé API exposée | Révoquer/remplacer puis informer le responsable | `DOC-04` |
| Commande inexistante | `404` | `DOC-02` |
| NullPointerException | Reproduire, lire la stack trace, vérifier les références nulles, tester | `DOC-03` |

## Vérifiez-vous

- [ ] Les réponses reprennent l’information du corpus.
- [ ] Les sources affichées correspondent à la question.
- [ ] Les réponses ne donnent pas une commande absente de la documentation.
- [ ] Une recommandation de correction rappelle d’exécuter les tests.

---

# Étape 6 — Tester une question hors périmètre

## Objectif

Vérifier que le chatbot reconnaît ce qu’il ne sait pas.

## Explication simple

Un bon RAG ne répond pas à tout. Il doit reconnaître que la documentation ne contient pas une information et orienter le développeur vers une autre source ou un responsable.

C’est une fonctionnalité de qualité, pas une faiblesse.

## Tâches à faire

```python
question_hors_perimetre = ""
Quelle version de Kubernetes est utilisée pour déployer OrionPay en production ?
"""

response = retrieval_chain.invoke({"input": question_hors_perimetre})

print(response["answer"])

print("\nChunks récupérés :")
for doc in response["context"]:
    print(doc.page_content.split("\n")[0])
```

## Résultat attendu

Le chatbot doit répondre exactement ou très près de :

```text
Je ne dispose pas de cette information dans la documentation OrionPay fournie.
```

Il ne doit pas inventer une version Kubernetes, une plateforme de déploiement ou une procédure ATOS.

## Vérifiez-vous

- [ ] Le chatbot n’invente pas de version Kubernetes.
- [ ] La réponse de repli est claire.
- [ ] Vous comprenez qu’un document hors sujet peut quand même être récupéré : le prompt de repli reste donc indispensable.

---

# Étape 7 — Comparer LLM nu et RAG

## Objectif

Voir concrètement la valeur ajoutée du RAG sur une question spécifique au projet.

## Explication simple

Le LLM nu peut répondre avec une procédure plausible. Le RAG fournit une réponse à partir d’une documentation contrôlée. Cela ne garantit pas que le corpus est à jour, mais cela permet de vérifier la source et de réduire les inventions.

## Tâches à faire

### 7.1 Question au LLM nu

```python
question = "Comment doit-on traiter une clé API exposée dans OrionPay ?"

raw_answer = llm.invoke(question)
print("RÉPONSE LLM NU :")
print(raw_answer.content)
```

### 7.2 Même question au RAG

```python
rag_answer = retrieval_chain.invoke({"input": question})
print("RÉPONSE RAG :")
print(rag_answer["answer"])
```

### 7.3 Comparaison

| Critère | LLM nu | RAG |
|---|---|---|
| Réponse basée sur un document OrionPay |  |  |
| Source identifiable |  |  |
| Risque d’inventer une procédure |  |  |
| Réponse vérifiable par un junior |  |  |

## Résultat attendu

Le RAG doit reprendre la procédure du `DOC-04` : révoquer ou remplacer le secret, puis informer le responsable technique ou sécurité.

Vous devez retenir :

> « Le RAG ne rend pas le LLM vrai. Il donne à la réponse une source contrôlée que je peux vérifier. »

---

# Étape 8 — Enrichir le corpus et réindexer

## Objectif

Comprendre qu’un RAG doit être maintenu quand la documentation évolue.

## Explication simple

Un RAG n’est jamais « terminé ». Si l’équipe ajoute une API, modifie une convention ou change une procédure, le corpus doit être mis à jour et réindexé.

## Tâches à faire

Ajoutez ces deux documents à la fin de `orionpay_docs.txt` :

```text
[DOC-07] OrionPay — Annulation de commande
La route DELETE /v1/orders/{id} annule une commande uniquement si son statut est PENDING.
Une commande déjà PAID ou SHIPPED ne peut pas être annulée par cette route.
Une annulation réussie retourne HTTP 204 sans corps de réponse.
Une commande introuvable retourne HTTP 404.

[DOC-08] OrionPay — Logs applicatifs
Les logs techniques doivent inclure un identifiant de corrélation généré par l'application.
Ils ne doivent jamais contenir de mot de passe, token, clé API, numéro de carte ou donnée client complète.
Pour diagnostiquer un incident, partagez uniquement un extrait de log anonymisé.
```

Puis :

1. Relancez les étapes de chargement, découpage et indexation.
2. Testez les questions suivantes :

```python
new_questions = [
    "Dans quel état une commande peut-elle être annulée ?",
    "Quel code HTTP retourne une annulation réussie ?",
    "Quelles données sont interdites dans les logs OrionPay ?"
]
```

3. Vérifiez que les réponses utilisent `DOC-07` et `DOC-08`.

## Résultat attendu

Les nouveaux documents sont retrouvés et utilisés dans les réponses.

## Vérifiez-vous

- [ ] Vous avez réindexé le corpus après modification.
- [ ] La question sur l’annulation retourne `PENDING`.
- [ ] La question sur l’annulation réussie retourne `204`.
- [ ] La question sur les logs mentionne secrets et données sensibles.

---

# Étape 9 — Écrire la fiche de conception RAG

## Objectif

Documenter le comportement de DevAssist ATOS pour qu’un autre développeur puisse le maintenir.

## Tâches à faire

Complétez ce tableau :

| Élément | Votre décision |
|---|---|
| Nom de l’assistant |  |
| Utilisateurs cibles |  |
| Documents autorisés dans le corpus |  |
| Documents interdits dans le corpus |  |
| Taille de chunk initiale |  |
| Nombre de chunks retournés (`k`) |  |
| Température du modèle |  |
| Réponse hors périmètre |  |
| Sources affichées |  |
| Contrôle humain requis |  |
| Politique de mise à jour du corpus |  |

## Corrigé de référence

| Élément | Recommandation |
|---|---|
| Nom de l’assistant | DevAssist ATOS — OrionPay |
| Utilisateurs cibles | Développeurs juniors de l’équipe OrionPay |
| Documents autorisés | README, guides de contribution, ADR validés, docs API, runbooks validés |
| Documents interdits | Secrets, `.env`, exports de production, données clients, logs bruts, code non autorisé |
| Taille de chunk initiale | 500 à 700 caractères, à tester selon le corpus |
| Nombre de chunks retournés | 2 au départ, puis évaluer |
| Température | 0 à 0.2 pour du support technique factuel |
| Réponse hors périmètre | Dire que l’information est absente du corpus ; orienter vers la documentation ou un senior |
| Sources affichées | Identifiant de document + extrait ou lien interne autorisé |
| Contrôle humain | Validation des correctifs, des incidents critiques, des sujets sécurité et des changements de production |
| Mise à jour | Réindexation à chaque version validée de documentation |

---

## 7. Grille de validation

| Critère | Réussi lorsque… |
|---|---|
| Corpus | Le fichier contient des documents fictifs, structurés et sans secrets |
| Chunks | Les fragments sont cohérents et suffisamment petits |
| Embeddings | La base FAISS est créée sans erreur |
| Recherche | Les quatre questions initiales retrouvent le bon document |
| RAG | Les réponses sont ancrées dans les sources retournées |
| Repli | Une question hors périmètre ne produit pas de réponse inventée |
| Sources | Les documents utilisés sont affichés ou identifiables |
| Maintenance | Deux documents ajoutés sont retrouvés après réindexation |
| Sécurité | Le corpus ne contient ni secrets ni données client |

---

## 8. Auto-évaluation

Avant de terminer, vérifiez que vous pouvez répondre sans relire le TP :

- [ ] Quelle est la différence entre recherche (*retrieval*) et génération (*generation*) ?
- [ ] Pourquoi utilise-t-on des embeddings plutôt qu’une recherche par mots exacts ?
- [ ] Pourquoi découper un document en chunks ?
- [ ] Pourquoi tester le retriever avant le LLM ?
- [ ] Pourquoi un prompt de repli reste-t-il nécessaire même avec un RAG ?
- [ ] Que faut-il faire lorsque la documentation change ?
- [ ] Quels documents ne faut-il jamais indexer dans un RAG ?

---

## 9. Page formateur

### Déroulé recommandé

| Séquence | Durée | Action formateur |
|---|---:|---|
| Introduction | 10 min | Relier l’hallucination du TP LLM à la nécessité d’une documentation contrôlée |
| Étape 0 | 10 min | Vérifier que le corpus est fictif et sans secrets |
| Étapes 1–2 | 20 min | Expliquer chunks, embeddings et FAISS ; laisser le téléchargement du modèle se terminer |
| Étape 3 | 15 min | Insister : tester la recherche seule avant la chaîne RAG |
| Étapes 4–5 | 20 min | Construire la chaîne et faire afficher les sources |
| Étapes 6–7 | 15 min | Tester hors périmètre et comparer LLM nu / RAG |
| Étape 8 | 15 min | Ajouter les docs, réindexer et vérifier les nouvelles réponses |
| Étape 9 / restitution | 10 min | Relier les décisions de conception au futur chatbot LangGraph |

### Questions à poser au groupe

1. « Si le retriever retourne le mauvais chunk, le LLM peut-il répondre de façon fiable ? »
2. « Pourquoi un RAG ne dispense-t-il pas de vérifier la réponse ? »
3. « Quel est le risque si le corpus contient une procédure obsolète ? »
4. « Pourquoi ne faut-il jamais indexer un fichier `.env` ? »
5. « Que faites-vous lorsque le chatbot ne trouve pas l’information ? »

### Erreurs fréquentes à corriger

| Erreur | Correction |
|---|---|
| « Le RAG élimine toutes les hallucinations » | Faux : il réduit le risque si le retriever et le prompt sont bien conçus |
| « Plus de chunks retournés est toujours mieux » | Faux : trop de contexte augmente coût, latence et dilution |
| « On peut indexer tout le dépôt Git » | Faux : il faut sélectionner les documents autorisés et utiles |
| « Le LLM nu et le RAG donnent la même valeur » | Le RAG apporte une source contrôlée et vérifiable |
| « Quand un document change, le RAG se met à jour seul » | Faux : il faut mettre à jour puis réindexer le corpus |
