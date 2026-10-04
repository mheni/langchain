# TP — Maîtriser le prompt engineering pour les développeurs juniors ATOS

> **Parcours IA pour développeurs juniors ATOS**  
> **Durée :** 1 h 30 à 2 h  
> **Niveau :** débutant Python / API IA / assistants de code  
> **Prérequis :** TP « Fondamentaux LLM : tokens, température, hallucinations et mémoire »  
> **Livrable :** une fiche de spécification de prompt et une bibliothèque de prompts de développement testés.

---

## 1. Pourquoi ce TP ?

Le TP précédent a montré les limites d’un LLM : il peut halluciner, varier ses réponses et manquer de contexte.

Le levier le plus simple pour améliorer un résultat n’est pas toujours de changer de modèle ou d’ajouter une architecture complexe. C’est souvent de **mieux écrire la demande**.

Pour un développeur, un prompt joue un rôle proche d’une spécification technique courte : il décrit le contexte, le comportement attendu, le format de sortie et les contraintes de sécurité ou de qualité.

Dans ce TP, vous allez aider l’équipe fictive **OrionPay** à automatiser une décision simple : déterminer si un ticket d’incident peut être **clôturé automatiquement** ou doit être **escaladé à un développeur senior**.

> **Règle essentielle :** un LLM assiste une décision. Il ne décide pas seul d’un incident critique, d’un sujet de sécurité ou d’une action irréversible.

---

## 2. Objectifs pédagogiques

À la fin du TP, vous serez capable de :

- écrire un prompt complet avec le cadre **RCIFC** : Rôle, Contexte, Instruction, Format, Contraintes ;
- distinguer les approches **zero-shot**, **few-shot**, raisonnement structuré et **self-consistency** ;
- choisir une approche selon une tâche de développement ;
- demander une sortie structurée et vérifiable, notamment en JSON ;
- détecter une règle métier ambiguë avant de la coder ;
- tester un prompt avec des cas normaux, des erreurs et des cas limites ;
- documenter un prompt afin qu’un autre développeur puisse le réutiliser.

---

## 3. Résultat attendu

À la fin du TP, vous devez remettre :

1. Un prompt `INCIDENT-01` qui décide entre clôture automatique et escalade.
2. Une sortie JSON structurée, testée sur au moins quatre cas.
3. Un cas limite qui met en évidence une ambiguïté de règle métier.
4. Une fiche de spécification de prompt réutilisable.
5. Trois prompts supplémentaires pour le cycle de développement : tests, documentation et revue de code.

---

## 4. Contexte métier : OrionPay

L’équipe ATOS maintient **OrionPay**, une application fictive de paiement basée sur Java 17, Spring Boot 3 et PostgreSQL 16.

Les développeurs reçoivent de nombreux tickets. Certains peuvent être clôturés automatiquement avec une réponse standard. D’autres doivent être envoyés à un développeur senior ou à l’équipe sécurité.

### Règle de gestion utilisée dans le TP

```text
Un ticket peut être clôturé automatiquement si TOUTES les conditions suivantes sont réunies :

1. Une solution documentée existe dans la base de connaissances.
2. Le ticket n'est ni critique ni lié à la sécurité.
3. Le ticket n'a pas été rouvert au cours des 30 derniers jours.
4. Le client n'a pas demandé explicitement une analyse humaine.
```

> Cette règle est volontairement simple. Dans un vrai projet, elle viendrait du support, du PO, de la sécurité et du client. Le développeur doit demander une clarification lorsque la règle est ambiguë.

---

## 5. Prérequis techniques

Réutilisez l’environnement du TP précédent :

```bash
pip install -U groq python-dotenv
```

Votre fichier `.env` :

```text
GROQ_API_KEY=gsk_votre_cle_ici
```

Code commun :

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


def ask_llm(prompt: str, temperature: float = 0) -> str:
    response = client.chat.completions.create(
        model=MODEL,
        messages=[{"role": "user", "content": prompt}],
        temperature=temperature,
    )
    return response.choices[0].message.content
```

> Utilisez exclusivement des tickets fictifs. Ne copiez jamais un ticket client réel, une clé, un token, un log de production brut ou du code propriétaire non autorisé.

---

# Étape 1 — Comprendre RCIFC

## Objectif

Apprendre une structure simple pour écrire des prompts professionnels et réutilisables.

## Explication simple

Un prompt vague laisse le modèle deviner trop de choses. RCIFC évite ce problème :

| Élément | Question à se poser | Exemple OrionPay |
|---|---|---|
| **R — Rôle** | Qui doit répondre ? | Tu es un ingénieur support senior ATOS. |
| **C — Contexte** | Dans quel projet et quelle situation ? | OrionPay, Java 17, tickets de support applicatif. |
| **I — Instruction** | Que doit faire le modèle ? | Décide entre clôture et escalade. |
| **F — Format** | Sous quelle forme répondre ? | Réponds avec un JSON précis. |
| **C — Contraintes** | Quelles règles respecter ? | Ne jamais décider seul pour un ticket sécurité. |

## Tâches à faire

1. Comparez les deux prompts ci-dessous.
2. Identifiez ce qui manque dans le premier.
3. Complétez votre propre version RCIFC dans le tableau.

### Prompt vague

```text
Est-ce que je peux fermer ce ticket ?
```

### Prompt structuré

```text
Tu es un ingénieur support senior ATOS.

Contexte : OrionPay est une application Java 17 et Spring Boot 3.

Instruction : décide si le ticket ci-dessous peut être clôturé automatiquement
ou doit être escaladé à un développeur senior.

Format : réponds par un JSON avec les champs decision, justification et action.

Contraintes : applique strictement les 4 règles fournies. Si le ticket est critique,
lié à la sécurité ou ambigu, choisis escalade_requise.
```

### Votre fiche RCIFC

| Élément | Votre texte |
|---|---|
| Rôle |  |
| Contexte |  |
| Instruction |  |
| Format |  |
| Contraintes |  |

## Résultat attendu

Un prompt complet qui donne au modèle un rôle, un contexte, une tâche précise, un format et des contraintes.

## Vérifiez-vous

- [ ] Votre prompt contient-il un rôle ?
- [ ] Le contexte indique-t-il la technologie et le sujet ?
- [ ] L’instruction commence-t-elle par un verbe d’action ?
- [ ] Le format de sortie est-il imposé ?
- [ ] Les contraintes indiquent-elles quoi faire en cas de doute ?

---

# Étape 2 — Tester le zero-shot

## Objectif

Observer ce qu’un modèle fait avec une règle et une demande simple, sans exemple.

## Explication simple

**Zero-shot** signifie : donner une consigne au modèle sans lui fournir d’exemple déjà résolu.

Cette technique est rapide et économique. Elle fonctionne pour des tâches simples, mais elle peut produire un format irrégulier ou oublier une condition si le raisonnement devient complexe.

## Tâches à faire

```python
prompt_zero_shot = """
Règle OrionPay : un ticket peut être clôturé automatiquement si :
1. Une solution documentée existe.
2. Il n'est ni critique ni lié à la sécurité.
3. Il n'a pas été rouvert au cours des 30 derniers jours.
4. Le client ne demande pas une analyse humaine.

Cas :
- solution documentée : oui
- priorité : normale
- sujet sécurité : non
- dernière réouverture : il y a 10 jours
- analyse humaine demandée : non

Le ticket peut-il être clôturé automatiquement ?
"""

print(ask_llm(prompt_zero_shot, temperature=0))
```

Puis répondez :

| Question | Votre réponse |
|---|---|
| Quelle décision le modèle a-t-il proposée ? |  |
| Quelle condition bloque la clôture ? |  |
| La réponse indique-t-elle toutes les conditions vérifiées ? |  |
| Le format est-il facile à lire par un programme ? |  |

## Résultat attendu

La décision attendue est : **escalade requise**, car le ticket a été rouvert il y a moins de 30 jours.

Vous devez retenir :

> « Le zero-shot est rapide, mais le format et la justification ne sont pas toujours assez contrôlés pour une automatisation. »

---

# Étape 3 — Utiliser le few-shot pour stabiliser le format

## Objectif

Utiliser des exemples résolus pour guider le modèle sur la réponse attendue.

## Explication simple

**Few-shot** signifie : ajouter quelques exemples de cas déjà résolus dans le prompt.

Les exemples montrent au modèle :

- la logique attendue ;
- le format de sortie ;
- le niveau de détail ;
- le vocabulaire à utiliser.

Les exemples ne remplacent pas la règle. Si la règle change, les exemples doivent être vérifiés et mis à jour.

## Tâches à faire

```python
prompt_few_shot = """
Règle OrionPay : un ticket peut être clôturé automatiquement si :
1. Une solution documentée existe.
2. Il n'est ni critique ni lié à la sécurité.
3. Il n'a pas été rouvert au cours des 30 derniers jours.
4. Le client ne demande pas une analyse humaine.

Exemple 1 :
Solution documentée : oui
Priorité : normale
Sujet sécurité : non
Dernière réouverture : il y a 45 jours
Analyse humaine demandée : non
Réponse : cloture_autorisee (toutes les conditions sont satisfaites)

Exemple 2 :
Solution documentée : non
Priorité : normale
Sujet sécurité : non
Dernière réouverture : jamais
Analyse humaine demandée : non
Réponse : escalade_requise (aucune solution documentée)

Exemple 3 :
Solution documentée : oui
Priorité : critique
Sujet sécurité : non
Dernière réouverture : il y a 90 jours
Analyse humaine demandée : non
Réponse : escalade_requise (ticket critique)

Nouveau cas :
Solution documentée : oui
Priorité : normale
Sujet sécurité : non
Dernière réouverture : il y a 90 jours
Analyse humaine demandée : oui
Réponse :
"""

print(ask_llm(prompt_few_shot, temperature=0))
```

Complétez :

| Question | Votre réponse |
|---|---|
| Quelle décision est attendue ? |  |
| Quel exemple a aidé le modèle à adopter le bon format ? |  |
| Quel risque existe si les exemples ne suivent plus la règle métier actuelle ? |  |

## Résultat attendu

```text
escalade_requise (analyse humaine demandée)
```

Vous devez retenir :

> « Le few-shot est utile quand le format doit être constant : JSON, commentaire de pull request, rapport d’incident, résultat de classification. »

## Vérifiez-vous

- [ ] Les exemples couvrent-ils plusieurs raisons possibles d’escalade ?
- [ ] Les exemples sont-ils compatibles avec la règle écrite ?
- [ ] La sortie est-elle plus régulière qu’en zero-shot ?

---

# Étape 4 — Demander une sortie JSON structurée

## Objectif

Obtenir une sortie exploitable et vérifiable par une application.

## Explication simple

Une phrase libre est difficile à lire automatiquement. Pour intégrer un LLM dans une application, on impose un format. Ici, nous utilisons JSON.

Le modèle doit répondre avec les conditions observées, une décision et une justification courte. Cette structure ne rend pas la réponse vraie par magie, mais elle facilite :

- la validation ;
- les tests ;
- le journal d’audit ;
- l’intégration dans du code.

## Tâches à faire

```python
prompt_json = """
Tu es un ingénieur support senior ATOS.

Règle OrionPay : un ticket peut être clôturé automatiquement si :
1. Une solution documentée existe.
2. Il n'est ni critique ni lié à la sécurité.
3. Il n'a pas été rouvert au cours des 30 derniers jours.
4. Le client ne demande pas une analyse humaine.

Cas :
- solution documentée : oui
- priorité : critique
- sujet sécurité : non
- dernière réouverture : il y a 120 jours
- analyse humaine demandée : non

Réponds uniquement avec un objet JSON valide au format exact :
{
  "solution_documentee": true,
  "priorite": "normale|critique",
  "sujet_securite": true,
  "reouvert_30_jours": false,
  "analyse_humaine_demandee": false,
  "decision": "cloture_autorisee|escalade_requise",
  "justification": "phrase courte"
}

Contraintes :
- aucune clé supplémentaire ;
- aucune phrase avant ou après le JSON ;
- en cas d'ambiguïté, decision = "escalade_requise".
"""

print(ask_llm(prompt_json, temperature=0))
```

## Tâches complémentaires

1. Copiez la réponse dans un validateur JSON ou chargez-la en Python :

```python
import json

raw_response = ask_llm(prompt_json, temperature=0)
data = json.loads(raw_response)

print(data["decision"])
print(data["justification"])
```

2. Vérifiez chaque champ.

| Champ | Valeur attendue |
|---|---|
| `solution_documentee` | `true` |
| `priorite` | `critique` |
| `sujet_securite` | `false` |
| `reouvert_30_jours` | `false` |
| `analyse_humaine_demandee` | `false` |
| `decision` | `escalade_requise` |
| `justification` | Indique que la priorité critique bloque la clôture |

## Résultat attendu

Le modèle retourne un JSON valide avec :

```json
{
  "decision": "escalade_requise"
}
```

## Vérifiez-vous

- [ ] La réponse est-elle du JSON valide ?
- [ ] La décision est-elle compatible avec la règle ?
- [ ] Votre code peut-il lire `data["decision"]` ?
- [ ] Le JSON contient-il une justification courte et vérifiable ?

---

# Étape 5 — Tester un cas limite et détecter une règle ambiguë

## Objectif

Comprendre qu’une réponse instable peut révéler une règle métier imprécise, pas seulement un problème de modèle.

## Explication simple

La règle dit :

> « Le ticket n’a pas été rouvert au cours des 30 derniers jours. »

Mais que signifie exactement **il y a 30 jours** ?

- La clôture est-elle autorisée dès le trentième jour ?
- Faut-il attendre 31 jours complets ?
- Quelle heure, quel fuseau et quelle date servent de référence ?

Le LLM ne peut pas inventer la bonne réponse métier. Le développeur doit détecter l’ambiguïté et demander une clarification au PO ou au support.

## Tâches à faire

```python
prompt_cas_limite = """
Règle OrionPay : un ticket peut être clôturé automatiquement si :
1. Une solution documentée existe.
2. Il n'est ni critique ni lié à la sécurité.
3. Il n'a pas été rouvert au cours des 30 derniers jours.
4. Le client ne demande pas une analyse humaine.

Cas :
- solution documentée : oui
- priorité : normale
- sujet sécurité : non
- dernière réouverture : exactement il y a 30 jours
- analyse humaine demandée : non

Analyse les conditions et donne une décision avec une justification courte.
Si la règle est ambiguë, indique clairement "REGLE_AMBIGUE".
"""

for i in range(3):
    print(f"--- Essai {i + 1} ---")
    print(ask_llm(prompt_cas_limite, temperature=0.7))
    print()
```

Complétez :

| Observation | Votre réponse |
|---|---|
| Les trois réponses sont-elles identiques ? |  |
| Quelle interprétation du « 30e jour » le modèle fait-il ? |  |
| Quelle question devez-vous poser au PO ou au support ? |  |
| Quelle décision sûre prenez-vous tant que la règle n’est pas clarifiée ? |  |

## Résultat attendu

Vous devez conclure :

> « La règle doit préciser si exactement 30 jours est autorisé ou non. Tant que ce point n’est pas validé, le chatbot doit escalader le ticket. »

## Vérifiez-vous

- [ ] Vous ne cherchez pas à forcer le modèle à inventer une règle métier.
- [ ] Vous avez écrit une question claire au PO ou au support.
- [ ] Vous avez choisi un comportement sûr par défaut : escalade.

---

# Étape 6 — Comparer les techniques de prompting

## Objectif

Choisir une technique adaptée à la tâche, et non appliquer la même recette partout.

## Explication simple

Chaque technique a une utilité différente. Il ne faut pas utiliser des exemples ou une sortie longue si une demande simple suffit.

## Tâches à faire

Complétez ce tableau :

| Technique | Quand l’utiliser dans un projet ATOS ? | Avantage | Limite |
|---|---|---|---|
| Zero-shot |  |  |  |
| Few-shot |  |  |  |
| Sortie JSON structurée |  |  |  |
| Self-consistency / répétition |  |  |  |

## Corrigé de référence

| Technique | Quand l’utiliser dans un projet ATOS ? | Avantage | Limite |
|---|---|---|---|
| Zero-shot | Explication simple d’une erreur ou résumé court | Rapide, prompt court | Format moins prévisible |
| Few-shot | Commentaires de pull request, classification avec libellés fixes | Format et vocabulaire plus réguliers | Exemples à maintenir |
| Sortie JSON structurée | Intégration dans une API ou un workflow automatique | Lisible par du code et testable | Format à valider ; ne garantit pas la vérité |
| Self-consistency / répétition | Cas limite, règle métier ambiguë, test de robustesse | Révèle les incohérences | Plus coûteux ; ne remplace pas une clarification métier |

---

# Étape 7 — Écrire votre fiche de spécification de prompt

## Objectif

Documenter un prompt pour qu’un autre développeur puisse le réutiliser, le tester et le maintenir.

## Explication simple

Un prompt de production est un artefact de développement : il doit avoir un nom, un but, des entrées, une sortie, des tests et des limites connues.

## Tâches à faire

Complétez la fiche suivante pour le prompt `INCIDENT-01`.

| Champ | Votre contenu |
|---|---|
| Nom du prompt | `INCIDENT-01 — Décider clôture ou escalade` |
| But |  |
| Technique choisie |  |
| Entrées nécessaires |  |
| Format de sortie |  |
| Contraintes de sécurité |  |
| Cas normal |  |
| Cas d’erreur |  |
| Cas limite |  |
| Contrôle humain |  |
| Décision en cas d’ambiguïté |  |

## Résultat attendu

Une fiche qui permet à un autre développeur de comprendre :

- quand utiliser le prompt ;
- quelles données il peut recevoir ;
- quelle sortie il produit ;
- comment le tester ;
- quand l’humain doit prendre la main.

---

# Étape 8 — Construire trois prompts complémentaires

## Objectif

Commencer une bibliothèque réutilisable pour le cycle de développement.

## Tâches à faire

Créez au moins trois prompts supplémentaires avec RCIFC.

| Nom | Usage | Résultat attendu |
|---|---|---|
| `TEST-01` | Générer des tests unitaires | Code de tests + tableau cas normal / limite / erreur |
| `DOC-01` | Documenter une méthode legacy | Docstring + hypothèses et zones incertaines |
| `REVIEW-01` | Réaliser une revue de code | Tableau : risque, gravité, explication, correction proposée |

Pour chaque prompt, ajoutez une contrainte :

```text
Aucune donnée client, clé, token, mot de passe ou code propriétaire non autorisé.
```

## Résultat attendu

Une première bibliothèque de quatre prompts :

```text
prompts/
├── INCIDENT-01.md
├── TEST-01.md
├── DOC-01.md
└── REVIEW-01.md
```

---

## 6. Grille de validation

| Critère | Réussi lorsque… |
|---|---|
| RCIFC | Le prompt contient rôle, contexte, instruction, format et contraintes |
| Zero-shot | Le participant explique sa limite principale |
| Few-shot | Les exemples guident un format constant |
| JSON | La sortie est valide et lisible par Python |
| Cas limite | L’ambiguïté « exactement 30 jours » est détectée |
| Sécurité | Aucun prompt ne contient de donnée réelle ou de secret |
| Documentation | `INCIDENT-01` possède une fiche de spécification |
| Réutilisabilité | Trois prompts complémentaires sont créés |

---

## 7. Auto-évaluation

Avant de terminer, vérifiez que vous pouvez répondre sans relire ce document :

- [ ] Que signifie RCIFC ?
- [ ] Quand le zero-shot est-il suffisant ?
- [ ] Pourquoi utiliser des exemples few-shot ?
- [ ] Pourquoi demander une sortie JSON ?
- [ ] Que faites-vous si une règle métier est ambiguë ?
- [ ] Pourquoi trois réponses différentes ne prouvent-elles pas forcément que le modèle est défaillant ?
- [ ] Pourquoi un prompt doit-il être versionné et testé comme du code ?

---

## 8. Page formateur

### Déroulé recommandé

| Séquence | Durée | Action formateur |
|---|---:|---|
| Introduction et rappel du TP précédent | 10 min | Relier hallucination et besoin de prompts précis |
| Étape 1 — RCIFC | 15 min | Construire un prompt collectif à partir du prompt vague |
| Étape 2 — Zero-shot | 10 min | Faire constater l’absence de format fiable |
| Étape 3 — Few-shot | 15 min | Insister sur les exemples à maintenir quand la règle change |
| Étape 4 — JSON | 15 min | Montrer `json.loads()` et la différence entre texte libre et sortie exploitable |
| Étape 5 — Cas limite | 15 min | Ne pas accepter une fausse précision ; faire formuler la question métier |
| Étapes 6–8 | 20 min | Faire produire les fiches de prompt et lancer une courte restitution |

### Questions à poser au groupe

1. « Pourquoi une réponse correcte mais non structurée est-elle difficile à intégrer dans une application ? »
2. « Que se passe-t-il si les exemples few-shot suivent une ancienne règle métier ? »
3. « Pourquoi “exactement 30 jours” est-il un cas dangereux ? »
4. « Que doit faire le chatbot tant que la règle métier n’est pas clarifiée ? »
5. « Quel prompt de votre quotidien allez-vous créer en premier ? »

### Erreurs fréquentes à corriger

| Erreur | Correction |
|---|---|
| « Few-shot rend le modèle intelligent » | Few-shot guide surtout le format et les exemples attendus |
| « JSON garantit une décision juste » | JSON rend la sortie testable, pas vraie par magie |
| « On peut laisser le modèle décider un cas ambigu » | Non : demander une clarification et escalader par défaut |
| « Un prompt est du texte non maintenu » | Un prompt de production doit être versionné, testé et documenté |
| « On colle le ticket complet dans le prompt » | Minimiser le contexte et anonymiser les données |
