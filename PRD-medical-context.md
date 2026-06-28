# Posology — PRD : Contexte médical / Mémoire du Nurse

## Objectif

Donner au Nurse suffisamment de contexte sur l'utilisateur pour que ses réponses soient pertinentes dès la première question, sans que l'utilisateur ait à se ré-expliquer à chaque conversation.

Ce n'est pas un dossier médical. C'est une mémoire légère et stable que l'utilisateur contrôle.

---

## Périmètre

Trois couches complémentaires, toutes déjà partiellement en place ou décidées :

| Couche | Statut |
|---|---|
| Champ `reason` sur chaque médicament du tracker | En dev (`feat/pill-reason`), mergé sur main dev |
| Section "Contexte médical" dans Settings | À implémenter |
| Bandeau de contexte replié en haut de l'onglet Nurse | À implémenter |
| Injection dans le system prompt de `nurse-chat` | À implémenter |

---

## 1. Champ `reason` sur les médicaments

Déjà documenté dans le tracker. Chaque entrée du tracker peut porter un champ `reason` libre (ex. "Pour les migraines", "Anticoagulant post-op"). Affiché comme chip gris neutre en bas de la carte médicament.

Injecté dans le system prompt sous forme de liste : `- Doliprane 1g (matin) — Pour les douleurs chroniques`.

---

## 2. Section "Contexte médical" dans Settings

Surface d'édition unique pour les données stables. L'utilisateur remplit une fois, met à jour si ça change.

### Champs

| Champ | Type | Limite | Placeholder |
|---|---|---|---|
| Conditions chroniques | Texte libre | 300 car. | "Ex. : diabète de type 2, hypothyroïdie…" |
| Allergies et intolérances connues | Texte libre | 200 car. | "Ex. : pénicilline, aspirine, lactose…" |
| Antécédents importants | Texte libre | 300 car. | "Ex. : opération cardiaque 2019, grossesse…" |
| Notes libres | Texte libre | 500 car. | "Tout ce qui peut aider le Nurse à mieux répondre" |

### Règles

- Tous les champs sont optionnels
- Sauvegarde automatique (pas de bouton "Enregistrer" séparé) ou bouton unique en bas de section
- Stocké dans Supabase sous la clé `pr:medical-context` (même namespace utilisateur)
- Aucun champ n'est affiché ailleurs dans l'app sauf dans le bandeau Nurse

---

## 3. Bandeau de contexte dans l'onglet Nurse

Affiché en haut de l'onglet Nurse, en lecture seule. Collapsé par défaut.

### Comportement

- État par défaut : replié, une ligne de résumé visible ("Contexte médical · 3 éléments renseignés")
- Dépliable par tap pour voir le détail complet
- Si aucun champ n'est renseigné : bandeau absent (pas de placeholder vide)
- Lien "Modifier" ouvre directement la section Settings correspondante
- Lecture seule — l'édition se fait exclusivement dans Settings

### Contenu affiché déplié

- Conditions chroniques (si renseigné)
- Allergies (si renseigné)
- Antécédents (si renseigné)
- Notes libres (si renseignées)
- Liste des médicaments actifs avec leur `reason` (si renseigné)

---

## 4. System prompt `nurse-chat` enrichi

Le system prompt est construit côté client et envoyé à `nurse-chat` via le champ `system` existant. La Edge Function le passe directement à l'API Anthropic sans modification.

### Structure du prompt

```
Tu es le Nurse de Posology, un assistant de santé bienveillant et informé.
Tu aides les utilisateurs à comprendre leurs médicaments, leurs interactions possibles,
et les précautions à prendre — sans jamais remplacer l'avis d'un médecin ou pharmacien.

RÈGLES DE COMPORTEMENT :
- Tu ne prescris pas, ne diagnostiques pas, ne modifies pas les traitements.
- Pour toute décision médicale, tu renvoies vers le médecin ou le pharmacien.
- Tu restes factuel, calme et rassurant.
- Si une question dépasse ton rôle, tu le dis clairement et sans détour.

CONFIDENTIALITÉ :
- Les données ci-dessous sont fournies par l'utilisateur pour améliorer la pertinence de tes réponses.
- Tu ne répètes pas ces données à l'utilisateur sauf si c'est utile à la réponse.
- Tu ne les transmets à personne et ne les mentionnes pas inutilement.

PROFIL UTILISATEUR :
[Si renseigné] Prénom : {{profile.name}}
[Si renseigné] Âge : {{profile.age}} ans
[Si renseigné] Poids : {{profile.weight}} kg

CONTEXTE MÉDICAL :
[Si renseigné] Conditions chroniques : {{medicalContext.conditions}}
[Si renseigné] Allergies et intolérances : {{medicalContext.allergies}}
[Si renseigné] Antécédents importants : {{medicalContext.history}}
[Si renseigné] Notes : {{medicalContext.notes}}

TRAITEMENT ACTIF :
[Liste générée dynamiquement]
- {{pill.name}} {{pill.dose}} ({{session}})[ — {{pill.reason}}]
```

### Règles d'injection

- Les blocs `[Si renseigné]` sont omis si le champ est vide ou null — jamais de ligne vide dans le prompt
- Le champ `reason` d'un médicament est omis si absent — pas de tiret orphelin
- Le profil est omis en entier si tous les champs profil sont vides
- Le bloc "Contexte médical" est omis en entier si tous les champs sont vides
- Construction côté client dans la fonction qui appelle `nurse-chat`

---

## Stockage

| Clé Supabase | Contenu |
|---|---|
| `pr:medical-context` | `{ conditions, allergies, history, notes }` — strings ou null |
| `pr:profile` | Existant — `{ name, age, weight, height }` |
| `pr:list` | Existant — inclut `reason` sur chaque entrée |

---

## Ce qui est hors périmètre

- Validation ou structuration des entrées (pas de liste déroulante de conditions ICD, texte libre uniquement)
- Partage du contexte médical entre utilisateurs
- Export du contexte médical
- Historique des modifications du contexte

---

## Dépendances

- `feat/pill-reason` mergé sur main dev : requis (champ `reason` doit exister sur les entrées tracker)
- `pr:profile` existant : utilisé tel quel
- Edge Function `nurse-chat` : aucune modification requise (accepte déjà `system` en paramètre)
