---
title: "Section 11 — Sortie structurée et traitement par lots"
linkTitle: "11. Sortie structurée et lots"
weight: 11
description: "Domaines 4.3–4.5 — JSON Schema avec champs nullables/optionnels, boucles validation-réessai et compromis de la Message Batches API."
---

## Ce que couvre cette section

Trois décisions d'architecture liées : **comment** faire produire à Claude une sortie analysable par machine (`tool_use` + JSON schemas, `strict: true`, la fonctionnalité plus récente Structured Outputs), **quand** déplacer le travail vers la **Message Batches API** pour le coût et le débit (et quand ne pas le faire), et **pourquoi** un relecteur indépendant ou une séparation par fichier + passe d'intégration est préférable à l'auto-relecture.

L'examen teste ces trois sujets avec des scénarios concrets — contrôles pré-merge contre rapports nocturnes, passe unique contre revue par fichier, prompts JSON-only contre `tool_use`. Les bonnes réponses découlent d'un principe : **adapter la technique aux contraintes de latence, de fiabilité et de budget d'attention de la charge de travail**.

## Source (guide officiel)

### 4.3 Sortie structurée via tool_use et JSON schemas
- `tool_use` avec JSON schemas est l'approche la plus fiable pour garantir une sortie conforme au schéma ; elle élimine les erreurs de syntaxe JSON.
- Modes `tool_choice` : `"auto"` (le modèle peut renvoyer du texte), `"any"` (il doit appeler un outil, mais peut choisir lequel), ou forcé `{"type": "tool", "name": "..."}`.
- Les schémas stricts éliminent seulement les erreurs de **syntaxe** — pas les erreurs **sémantiques** (lignes qui ne totalisent pas le montant, valeurs dans les mauvais champs, contenu fabriqué pour des données absentes).
- Conception du schéma : requis vs optionnel/nullable, enums avec `"other"` + chaîne de détail pour l'extensibilité, `"unclear"` pour l'ambiguïté, règles de normalisation de format dans le prompt.

### 4.5 Stratégie de traitement par lots
- Message Batches API : **50 % d'économie**, fenêtre de traitement jusqu'à **24 heures**, **aucun SLA de latence**.
- Adaptée aux charges non bloquantes et tolérantes à la latence (rapports nocturnes, audits hebdomadaires, génération de tests de nuit). Inadaptée aux workflows bloquants (contrôles pré-merge).
- Selon le guide : la Batch API ne prend pas en charge l'exécution d'outils *agentique* multi-tour au sein d'une seule requête — impossible de suspendre une requête pour exécuter un outil puis réinjecter ses résultats.
- `custom_id` corrèle requête et réponse ; les `custom_id` échoués peuvent être soumis à nouveau après correction (par exemple découper les documents qui dépassaient le contexte).

### 4.6 Revue multi-instance et multi-passe
- Un modèle qui conserve son raisonnement de génération remet moins en question ses propres décisions ; l'**auto-relecture** est donc structurellement faible.
- Des instances de revue indépendantes détectent des problèmes que l'auto-relecture et l'extended thinking manquent.
- Les revues multi-fichiers doivent être séparées en **passes locales par fichier** plus une **passe d'intégration inter-fichiers** pour éviter la dilution de l'attention et les constats contradictoires.
- Les passes de vérification peuvent demander au modèle de déclarer une **confiance par constat** afin de permettre un routage calibré.

## Sortie structurée avec tool_use

### Pourquoi tool_use bat "réponds uniquement en JSON"

Demander au modèle de "répondre uniquement en JSON" fonctionne la plupart du temps et échoue juste assez souvent pour devenir un risque de production : prose parasite, guillemets typographiques, virgules finales, blocs Markdown, préambule d'excuse. `tool_use` supprime toute cette classe d'échecs — le modèle n'émet pas du texte libre, il émet un bloc `tool_use` structuré dont l'`input` est garanti comme objet JSON.

Ajouter `"strict": true` (échantillonnage contraint par grammaire) garantit en plus que le JSON respecte les types, enums et champs requis du schéma. Le mode strict est pris en charge sur Opus 4.7/4.6/4.5, Sonnet 4.6/4.5 et Haiku 4.5 ([Strict tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/strict-tool-use)). La nouvelle fonctionnalité **Structured Outputs** (`output_config.format = { "type": "json_schema", "schema": ... }`) apporte la même garantie sans définir un faux outil ([Structured outputs](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs)) ; les deux partagent le même sous-ensemble JSON Schema et la même limite sur les erreurs sémantiques.

```python
import anthropic

extract_invoice = {
    "name": "extract_invoice",
    "description": "Extract structured invoice data from the document.",
    "strict": True,
    "input_schema": {
        "type": "object",
        "additionalProperties": False,
        "required": ["vendor", "invoice_number", "line_items",
                     "stated_total", "calculated_total", "currency", "category"],
        "properties": {
            "vendor": {"type": "string"},
            "invoice_number": {"type": "string"},
            "issue_date": {"type": ["string", "null"], "format": "date"},
            "currency": {"type": "string", "enum": ["USD", "EUR", "GBP", "other"]},
            "currency_other": {"type": ["string", "null"]},
            "category": {"type": "string",
                         "enum": ["saas", "hardware", "travel",
                                  "professional_services", "other", "unclear"]},
            "category_other": {"type": ["string", "null"]},
            "line_items": {
                "type": "array",
                "items": {
                    "type": "object",
                    "additionalProperties": False,
                    "required": ["description", "amount"],
                    "properties": {
                        "description": {"type": "string"},
                        "amount": {"type": "number"}
                    }
                }
            },
            "stated_total":     {"type": "number"},
            "calculated_total": {"type": "number"}
        }
    }
}

client = anthropic.Anthropic()
resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    tools=[extract_invoice],
    tool_choice={"type": "tool", "name": "extract_invoice"},
    messages=[{"role": "user", "content": INVOICE_TEXT}],
)
data = next(b.input for b in resp.content if b.type == "tool_use")
```

### Aide-mémoire tool_choice

| `tool_choice` | Comportement du modèle | À utiliser quand |
| -------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------- |
| `{"type": "auto"}` | Peut répondre en texte *ou* appeler n'importe quel outil | Boucles agentiques où les réponses textuelles sont valides |
| `{"type": "any"}` | **Doit** appeler un outil ; choisit lequel | Extraction multi-schémas lorsque le type de document est inconnu |
| `{"type": "tool", "name": "extract"}` | **Doit** appeler cet outil précis | Forcer une étape d'extraction connue avant l'enrichissement |
| `{"type": "none"}` | Ne peut appeler aucun outil | Désactiver les outils pour un tour sans reconstruire la requête |

Associez n'importe lequel de ces modes à `"disable_parallel_tool_use": true` si vous avez besoin d'au plus un appel d'outil par tour — utile quand le code aval attend un seul payload structuré, ou quand des appels concurrents violeraient des invariants d'ordre ([Parallel tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/parallel-tool-use)).

### Modèles de conception de schéma

- **Requis vs optionnel / nullable.** Les champs requis forcent le modèle à les remplir, ce qui provoque des fabrications quand la source ne contient réellement pas la donnée. Marquez les champs parfois absents comme `"type": ["string", "null"]` pour que le modèle puisse renvoyer `null` au lieu d'halluciner.
- **Enum avec `"other"` + détail.** Les enums fermées sont fragiles. Une enum `category` `[..., "other"]` + un champ frère `category_other: string` préserve l'information pour de nouvelles catégories sans casser le code aval.
- **`"unclear"` pour l'ambiguïté.** Une valeur d'enum `"unclear"` permet au modèle de signaler une faible confiance ; routez ces lignes vers une revue humaine.
- **Champs auto-validants.** Émettre à la fois `stated_total` (tiré du document) et `calculated_total` (somme des `line_items`) ; un contrôle de post-traitement détecte les erreurs sémantiques que les schémas stricts ne peuvent pas attraper.

### Sous-ensemble JSON Schema pris en charge

Le compilateur du mode strict / structured outputs accepte un sous-ensemble de JSON Schema. La plupart des erreurs "pourquoi mon schéma renvoie 400 ?" viennent de cette liste :

- **Pris en charge :** object/array/string/integer/number/boolean/null, `enum` (primitives uniquement), `const`, `anyOf`/`allOf` (limité), `$ref`/`$def` internes, `default`, `required`, `additionalProperties: false`, formats (`date-time`, `date`, `email`, `uri`, `uuid`, ...), `minItems` de 0 ou 1.
- **Non pris en charge :** `$ref` externe, schémas récursifs, types complexes dans les enums, contraintes numériques (`minimum`/`maximum`/`multipleOf`), contraintes de longueur de chaîne, `additionalProperties` != `false`.
- **Regex :** quantificateurs, classes de caractères et groupes fonctionnent ; les backreferences, lookarounds et word boundaries ne fonctionnent pas.

### Ce que tool_use ne résout PAS

Les schémas stricts garantissent la *parseabilité*, pas la *correction*. Ils peuvent parfaitement émettre une facture valide au schéma où les lignes totalisent 812 $ alors que `stated_total` indique 1 200 $, ou où le nom du fournisseur apparaît dans `invoice_number`. Pour détecter cela, il faut une validation applicative (par exemple l'astuce `calculated_total` ci-dessus), des boucles retry-with-error-feedback (Domaine 4.4), ou une passe de revue indépendante (Domaine 4.6).

## Message Batches API

### Ce que vous gagnez / ce que vous abandonnez

| Dimension | Messages API synchrone | Message Batches API |
| -------------------------- | ---------------------------------- | ------------------------------------------------------------ |
| Tarification | Standard | **50 % de réduction** entrée + sortie (p. ex. Sonnet 4.6 à 1,50 $/7,50 $ par MTok) |
| Latence | Secondes | Jusqu'à **24 heures** ; la plupart des lots finissent en moins d'1 heure ; pas de SLA |
| Requêtes max / lot | n/a | **100 000** requêtes **ou 256 MB**, au premier seuil atteint |
| Utilisation d'outils | Boucle agentique complète | Des outils peuvent être définis ; **l'exécution d'outils multi-tour au milieu d'une requête n'est pas prise en charge** |
| Streaming | Oui | **Non pris en charge** (les résultats sont récupérés quand le lot se termine) |
| Prompt caching | Oui | Oui (best-effort ; à associer au cache 1 heure pour le contexte partagé) |
| Conservation des résultats | n/a | 29 jours |
| Modèles | Tous les modèles actifs | Tous les modèles actifs |

Sources : [Batch processing](https://docs.anthropic.com/en/docs/build-with-claude/message-batches).

### Quand l'utiliser / quand l'éviter

**Utilisez la Batches API quand :**

- La charge n'est pas bloquante — rapports nocturnes de dette technique, audits hebdomadaires, génération de tests de régression, arriérés de modération de contenu, évaluations massives.
- Les volumes sont assez grands pour que la réduction de 50 % compte réellement.
- Chaque requête est **autonome** (pas besoin d'injecter des résultats d'outils entre les tours).

**Ne l'utilisez pas quand :**

- Un humain attend le résultat (contrôles pré-merge, chat, UI interactive).
- Le workflow exige que le modèle appelle des outils, voie les résultats, puis continue à raisonner dans la même requête.
- Vous avez besoin de streaming ou d'une latence inférieure à la minute.

C'est exactement la structure de la **Sample Question 11** : déplacer le rapport nocturne de dette technique vers batch (A), garder le contrôle pré-merge bloquant sur l'API synchrone. Les mauvaises réponses reposent toutes sur l'espoir que les lots "finissent généralement assez vite" ou ajoutent un fallback par timeout — ni l'un ni l'autre n'est acceptable quand le SLA est "un développeur attend devant l'écran".

### custom_id et gestion des échecs

Chaque requête dans un lot porte un `custom_id` (1–64 caractères, `[a-zA-Z0-9_-]`). C'est le **seul** mécanisme de corrélation entre résultats et entrées, car l'ordre de sortie n'est pas garanti. Mettez assez de métadonnées dans le `custom_id` pour retrouver l'enregistrement original — `doc_42891-v3-2026q1` convient ; `req_001` vous hantera.

Quand vous récupérez les résultats, chaque entrée a un `result` de type `succeeded`, `errored`, `canceled` ou `expired`. Modèle d'échec standard :

1. Récupérer le flux de résultats et partitionner par `result.type`.
2. Pour les entrées `errored`, inspecter le code d'erreur : découper les documents trop volumineux, corriger les paramètres invalides, puis resoumettre **uniquement les `custom_id` échoués** dans un nouveau lot plus petit.
3. Pour les entrées `expired` (lot non terminé en 24 h), resoumettre avec une taille de lot inférieure ou hors pic.

### Exemple travaillé : extraction nocturne de 100 documents

```python
from anthropic import Anthropic
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

client = Anthropic()

requests = [
    Request(
        custom_id=f"invoice-{doc.id}",
        params=MessageCreateParamsNonStreaming(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            tools=[extract_invoice],
            tool_choice={"type": "tool", "name": "extract_invoice"},
            messages=[{"role": "user", "content": doc.text}],
        ),
    )
    for doc in documents
]

batch = client.messages.batches.create(requests=requests)

while True:
    batch = client.messages.batches.retrieve(batch.id)
    if batch.processing_status == "ended":
        break
    time.sleep(60)

for entry in client.messages.batches.results(batch.id):
    if entry.result.type == "succeeded":
        msg = entry.result.message
        payload = next(b.input for b in msg.content if b.type == "tool_use")
        store(entry.custom_id, payload)
    else:
        log_failure(entry.custom_id, entry.result)
```

Notez la combinaison : `tool_use` pour une sortie sûre côté schéma **dans** une requête batch. L'extraction en un seul appel n'a pas d'appels d'outils au milieu de la requête, donc la contrainte de la Batch API ne bloque pas.

### Math de SLA : à quelle fréquence soumettre

Si le SLA métier est "résultats sous N heures" et que la Batch API peut prendre jusqu'à 24 heures, vous devez soumettre à une cadence telle que **délai avant soumission + 24 h de traitement ≤ N**. Pour un SLA N = 30 h, soumettre toutes les **4 heures** (attente pire cas de 4 h avant que le prochain lot prenne l'enregistrement, plus 24 h de traitement = 28 h, confortablement sous 30 h). Pour un SLA de 26 h, toutes les heures. Pour un SLA de 24 h, vous ne pouvez pas le tenir avec la seule Batches API — revenez aux appels synchrones ou acceptez des SLA manqués.

## Architecture de revue multi-instance et multi-passe

### Le piège de l'auto-relecture

Quand la même instance Claude qui a écrit le code le relit aussi, le raisonnement de génération est encore dans le contexte — le modèle traite ses choix précédents comme des prémisses plutôt que comme des hypothèses à challenger. Même une instruction explicite "critique ta réponse précédente" est plus faible qu'une **instance fraîche** sans engagement préalable à défendre.

Le système Code Review d'Anthropic reflète cela : il lance **plusieurs agents spécialisés en parallèle** (prompts distincts pour logique, sécurité, cas limites) et exécute une **étape de vérification** contre le comportement réel du code pour filtrer les faux positifs avant de publier des constats ([Claude Code Code Review](https://claude.com/blog/code-review)). Séparez le travail, utilisez un contexte indépendant, réconciliez à la fin.

### Modèle par fichier + intégration inter-fichiers

C'est la bonne réponse à la **Sample Question 12** (PR de 14 fichiers avec profondeur incohérente et constats contradictoires) :

```
                    ┌────────────────────────────┐
                    │  Per-file local passes     │
PR (14 files) ──►  │  - one Claude call per file │  ──► findings_local[]
                    │  - focused prompt           │
                    │  - no other files in ctx    │
                    └────────────────────────────┘
                                  │
                                  ▼
                    ┌────────────────────────────┐
                    │  Integration pass           │
                    │  - all diffs + module map   │  ──► findings_integration[]
                    │  - cross-file data flow     │
                    │  - API contracts, types     │
                    └────────────────────────────┘
                                  │
                                  ▼
                          dedupe + rank + post
```

Les passes par fichier donnent à chaque fichier le **même budget d'attention**, ce qui élimine le mode d'échec "très profond sur le fichier 1, superficiel sur le fichier 14". La passe d'intégration est explicitement promptée pour les préoccupations inter-fichiers (dérive de signature appelant/appelé, changements de schéma partagés, invariants transactionnels), afin de ne pas refaire ce que les passes locales ont déjà fait.

### Modèle de relecteur indépendant

Pour les artefacts à enjeu élevé (migration générée, rapport client), exécutez **deux appels Claude** :

1. **Générateur** — produit l'artefact avec tout son raisonnement.
2. **Relecteur** — une *nouvelle* requête, un *nouveau* system prompt, *sans* transcript du générateur, recevant seulement l'artefact et la spécification. Sa seule mission est de trouver des défauts.

L'absence de contexte du relecteur est une qualité, pas un bug — il ne peut pas rationaliser des décisions qu'il n'a jamais prises.

### Passes de vérification avec confiance annotée

Demandez au relecteur de renvoyer les constats avec une enum `confidence` explicite (`"high"`, `"medium"`, `"low"`) et une `rationale` d'une ligne :

```json
{
  "findings": [
    {"severity": "high", "confidence": "high",
     "file": "billing.py", "line": 142,
     "issue": "Off-by-one in proration when subscription starts on month boundary",
     "rationale": "Integration test billing_test.py:88 covers mid-month only."}
  ]
}
```

Puis routez par confiance : `high` confidence + `high` severity part directement dans la PR comme commentaire bloquant ; les constats `low` confidence vont vers une file de triage ou déclenchent une passe d'arbitrage. C'est le **routage de revue calibré** évoqué dans la compétence officielle.

## Matrice de décision : quelle technique pour quel travail

| Charge de travail | Besoin de latence | Profondeur de revue | Pile recommandée |
| --------------------------------------- | -------------- | ---------------------- | ----------------------------------------------------------------------- |
| Contrôle pré-merge bloquant | Secondes | Par fichier + intégration | Messages API synchrone + `tool_use(strict)` + revue multi-passe |
| Rapport nocturne de dette technique | Heures | Par fichier + intégration | **Batches API** + `tool_use(strict)` + revue multi-passe |
| Extraction de champs sur 100k documents | Nocturne | QC par échantillon seulement | **Batches API** + `tool_choice` forcé + champs auto-validants |
| Chat interactif avec étape d'extraction | Secondes | Aucune | Messages API synchrone + `tool_choice` forcé + champs nullables |
| QC de document réglementaire | Minutes-heures | Relecteur indépendant | Synchrone (ou batch) + séparation générateur/relecteur + routage par confiance |
| Audit hebdomadaire multi-repos | Jours | Par repo seulement | **Batches API** + `tool_use` + passer l'intégration |

## Points d'attention pour l'examen

- **`tool_use` vs "réponds en JSON" :** la bonne réponse pousse presque toujours vers `tool_use` / schémas stricts pour garantir la parseabilité. Les prompts JSON brut sont un distracteur.
- **Choix de `tool_choice` :** `"any"` pour un type de document inconnu parmi plusieurs outils d'extraction ; forcé `{"type":"tool","name":"..."}` pour garantir qu'un outil précis s'exécute avant l'enrichissement ; `"auto"` seulement quand les réponses textuelles sont légitimes.
- **Conception du schéma :** nullable quand la source peut manquer de données (évite la fabrication) ; `"other"` + détail pour l'extensibilité ; `"unclear"` pour l'ambiguïté ; champs auto-validants pour les contrôles sémantiques.
- **Adéquation de la Batches API :** charges **non bloquantes, tolérantes à la latence, ≤24 h** uniquement. Les contrôles pré-merge sont le mauvais cas canonique. 50 % de réduction, limite 100k req / 256 MB, résultats valides 29 jours, pas de streaming, pas d'exécution d'outil au milieu d'une requête.
- **`custom_id` :** obligatoire pour la corrélation ; resoumettre seulement les IDs échoués après correction de la cause.
- **Revue multi-passe :** local par fichier + intégration inter-fichiers bat une passe unique sur les PR multi-fichiers ; les instances indépendantes battent l'auto-relecture ; les constats annotés par confiance permettent le routage.
- **Idées fausses à éviter :** "une fenêtre de contexte plus grande corrige la dilution d'attention" (non), "trois passes complètes + vote majoritaire" (supprime des bugs réels détectés par intermittence), "fallback timeout de batch vers sync" (trop complexe ; choisissez la bonne API par charge).

## Références

- [Tool use overview](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
- [Define tools](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/define-tools)
- [Strict tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/strict-tool-use)
- [Parallel tool use & `disable_parallel_tool_use`](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/parallel-tool-use)
- [Structured outputs (`output_config.format` JSON Schema)](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs)
- [Message Batches API — batch processing](https://docs.anthropic.com/en/docs/build-with-claude/message-batches)
- [Create / list / retrieve Message Batches (API reference)](https://docs.anthropic.com/en/api/creating-message-batches)
- [Prompt caching with batches (1-hour cache duration)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Claude Code: multi-agent Code Review system](https://claude.com/blog/code-review)
- [Claude Code Code Review docs](https://docs.anthropic.com/en/docs/claude-code/code-review)
