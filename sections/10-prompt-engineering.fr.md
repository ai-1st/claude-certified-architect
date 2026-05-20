---
title: "Section 10 — Ingénierie de prompts : critères explicites, few-shot et boucles de validation"
linkTitle: "10. Ingénierie de prompts"
weight: 10
description: "Domaines 4.1, 4.2, 4.4 — critères explicites plutôt que seuils vagues, few-shot pour scénarios ambigus et revue multi-passe pour gros diffs."
---

## Ce que couvre cette section

Trois techniques de prompt que l'examen Foundations considère comme centrales pour faire passer une fonctionnalité Claude de la qualité démo à la qualité production : **critères catégoriels explicites** plutôt qu'instructions vagues, **2–4 exemples few-shot ciblés**, et une **boucle validation-retry** avec signaux d'auto-correction intégrés au schéma. Le Domaine 4.3 (tool use + schémas JSON) est couvert dans la Section 11 ; cette section ferme l'écart *sémantique* que les schémas stricts ne ferment pas.

## Matériel source (guide officiel)

### 4.1 Critères explicites plutôt qu'instructions vagues

- Les critères catégoriels explicites battent les instructions vagues : *"flag comments only when claimed behavior contradicts actual code behavior"* surpasse *"check that comments are accurate"*. Les instructions générales comme *"be conservative"* ou *"only report high-confidence findings"* n'améliorent **pas** la précision ; de forts taux de faux positifs dans une seule catégorie érodent la confiance dans **toutes** les catégories.
- Compétences : écrire des règles report-when / skip-when spécifiques par catégorie ; désactiver temporairement les catégories à fort FP pendant leur amélioration ; définir des critères de sévérité explicites avec exemples de code concrets par niveau.

### 4.2 Few-shot prompting

- Les exemples few-shot sont la technique **la plus efficace** pour obtenir une sortie formatée, actionnable et cohérente lorsque des instructions détaillées seules produisent des résultats incohérents. Ils guident les cas ambigus (sélection d'outils, escalade), permettent la généralisation à de nouveaux motifs et réduisent les hallucinations dans l'extraction sur structures documentaires variées.
- Compétences : 2–4 exemples ciblés montrant le *raisonnement* pour l'action choisie plutôt que des alternatives plausibles ; exemples démontrant le format de sortie souhaité (localisation, problème, sévérité, correction suggérée) ; exemples distinguant les motifs acceptables des vrais problèmes ; exemples couvrant chaque variante structurelle de l'entrée.

### 4.4 Validation, retry et boucles de feedback

- Retry-with-error-feedback : ajouter l'erreur de validation **spécifique** au prompt suivant pour guider l'auto-correction. Le retry est inefficace lorsque l'information requise est **absente de la source**. Distinguer les erreurs de validation **sémantiques** (valeurs qui ne somment pas, mauvais placement de champ) des erreurs de **syntaxe de schéma** (déjà éliminées par tool use).
- Compétences : follow-up contenant document original + extraction échouée + erreurs spécifiques ; ajouter `detected_pattern` pour l'analyse des faux positifs ; concevoir l'auto-correction en extrayant `calculated_total` à côté de `stated_total` ; ajouter des booléens `conflict_detected` pour les données source incohérentes.

## Écrire des critères explicites

### Vague vs spécifique — réécritures avant/après

Le geste le plus important du prototype à la production consiste à remplacer les instructions floues ("be careful", "only report high-confidence findings", "be conservative") par des **critères catégoriels qui définissent le comportement**. La confiance auto-déclarée est mal calibrée : un modèle confiant à tort sur les cas difficiles ne sera pas sauvé par *"only report high-confidence findings"* — il continuera à rapporter les mêmes choses fausses.

| Vague (ce que les équipes écrivent d'abord) | Spécifique (à quoi ressemblent les prompts de production) |
| --- | --- |
| "Check that comments are accurate." | "Flag a comment **only** when its claimed behaviour contradicts the actual code behaviour. Do **not** flag comments that are merely vague, outdated stylistically, or under-detailed." |
| "Find security issues." | "Report only: SQL string concatenation reaching a query call, `eval`/`exec` on attacker-controlled input, secrets in source, missing auth checks on routes under `/api/admin/*`. Skip: defensive `assert` patterns, hardcoded test fixtures, anything in `tests/`, `fixtures/`, `examples/`." |
| "Be conservative when escalating." | "Escalate when (a) the customer asks for a policy exception, (b) the claim value > $X, or (c) more than 2 prior contacts on the same case. Resolve autonomously when the photo evidence matches a standard damage SKU and value < $X." |
| "Only report high-confidence findings." | "Report a finding only if you can quote the exact line from the source and name the specific rule it violates. If you cannot quote a line, do not report." |

Remarquez le motif : chaque "bonne" version répond à deux questions — *qu'est-ce qui compte comme positif ?* et *qu'est-ce qui est un non-problème à ignorer ?* Sample Question 3 du guide fait le même point côté agent : 55% de résolution au premier contact s'améliore avec des **critères d'escalade explicites et exemples few-shot**, pas avec un score de confiance auto-déclaré.

### Définitions de sévérité avec exemples de code

Pour tout classifieur qui émet un champ `severity`, définissez chaque niveau avec un exemple de code concret *dans le prompt*. Sinon, les runs différents brouillent la distinction entre `low` et `medium`.

```xml
<severity_definitions>
  <critical>Data loss, auth bypass, or RCE in prod.
    Example: db.query("SELECT * FROM users WHERE id=" + req.params.id)</critical>
  <high>Bug producing incorrect output for at least one realistic input.
    Example: off-by-one in pagination dropping the last page.</high>
  <medium>Resource leak or correctness issue only under load.
    Example: file handle opened in a loop without `with` / `defer close`.</medium>
  <low>Style or readability only — do NOT include unless explicitly asked.</low>
</severity_definitions>
```

Ancrer chaque niveau dans un exemple force une classification cohérente entre runs et entre reviewers.

### Désactiver puis reconstruire les catégories à fort FP

Les faux positifs d'une seule catégorie érodent la confiance dans toutes les catégories. Le guide recommande un motif délibéré disable-then-rebuild : quand "comment accuracy" tourne à 40% FP, mieux vaut retirer complètement cette catégorie que la laisser active et dégrader la confiance dans les findings sécurité que les développeurs traitent réellement. Itérez la mauvaise catégorie dans un prompt latéral jusqu'à dépasser votre seuil de précision, puis réactivez.

### Pourquoi "be conservative" ne marche pas

Calibration. Les adverbes comme "carefully" ou "only when sure" ne donnent rien au modèle qu'il puisse vérifier mécaniquement. À l'inverse, *"do not report a finding unless you can quote the exact source line and name the rule"* est une règle vérifiable — le modèle peut la contrôler lui-même avant d'émettre la sortie.

## Deep-dive few-shot prompting

### Combien d'exemples ?

La recommandation publiée d'Anthropic et le guide de certification convergent sur le même nombre : **2–4 exemples ciblés**. Moins de 2 n'établit pas de motif ; plus de ~4 brûle des tokens pour des rendements décroissants et risque de surajuster le modèle à des traits de surface des exemples. Pour les prompts extended-thinking, Anthropic note spécifiquement que multishot aide encore mais doit montrer des *motifs de raisonnement*, pas seulement des paires entrée→sortie.

### Anatomie d'un exemple

Les exemples efficaces portent trois champs, pas deux :

1. **Input** — un extrait qui ressemble à l'entrée de production.
2. **Reasoning** — *pourquoi* l'action choisie est correcte plutôt que les alternatives plausibles.
3. **Output** — exactement la forme (XML ou JSON) voulue à l'inférence.

```xml
<example>
  <input>
    User: "My order from last Tuesday hasn't arrived and I want my money back."
  </input>
  <reasoning>
    This is a delivery exception within standard policy (lost package, &lt; 30 days,
    refund value within auto-approve threshold). No policy exception is being
    requested. Resolve autonomously with refund + reship.
  </reasoning>
  <output>
    {"action": "resolve", "tool": "issue_refund", "escalate": false}
  </output>
</example>
```

### Exemples pour cas ambigus, format et réduction de FP

- **Cas ambigus.** Choisissez 2–4 exemples de votre eval set où le modèle s'est mal comporté, et incluez une **paire contrastive** — un exemple qui ressemble mais doit être traité de façon **opposée**. Les paires contrastives enseignent la généralisation plutôt que la mémorisation.
- **Cohérence de format.** Si vous voulez chaque finding sous forme `{location, issue, severity, suggested_fix}`, montrez trois findings exactement sous cette forme. Les instructions seules laissent fuiter des capitalisations incohérentes, champs manquants et prose finale.
- **Réduction des faux positifs.** Associez un exemple "issue" à un exemple "acceptable pattern" qui partage des traits de surface. Pour un reviewer SQL injection, montrez une concat vulnérable *et* un appel paramétré étiqueté `not_a_finding` — c'est ainsi que vous empêchez le modèle de signaler chaque `db.query(...)`.
- **Structures documentaires variées.** Pour des factures à lignes désordonnées, des papiers avec citations inline vs bibliographie, ou une méthodologie dans la narration, donnez un exemple *par variante structurelle*. Sans couverture de chaque forme, l'extraction renvoie null sur les formats non vus.

### Où placer les exemples — tags XML

Les docs de prompt engineering d'Anthropic sont explicites : enveloppez les exemples dans des tags XML (`<example>`, `<examples>`, `<input>`, `<output>`) et placez-les **après** les instructions et critères de tâche mais **avant** l'entrée live.

```xml
<task>...explicit criteria here...</task>
<severity_definitions>...</severity_definitions>
<examples>
  <example>...</example>
  <example>...</example>
  <example>...</example>
</examples>
<input>{{actual_document_to_process}}</input>
```

## Validation, retry et boucles de feedback

### Retry-with-error-feedback

Le motif est : valider programmatiquement la sortie du modèle, puis en cas d'échec envoyer un *nouveau* tour contenant l'entrée originale, la sortie échouée et l'erreur de validation **spécifique**. Un "try again" générique n'aide jamais — le texte d'erreur concret oui.

```python
from anthropic import Anthropic

client = Anthropic()
MODEL = "claude-opus-4-7"

def extract_with_retry(document: str, max_attempts: int = 2) -> dict:
    messages = [{"role": "user", "content": build_extraction_prompt(document)}]
    for attempt in range(max_attempts + 1):
        resp = client.messages.create(
            model=MODEL, max_tokens=2048,
            tools=[INVOICE_TOOL],
            tool_choice={"type": "tool", "name": "extract_invoice"},
            messages=messages,
        )
        extraction = next(b.input for b in resp.content if b.type == "tool_use")

        errors = validate(extraction)
        if not errors:
            return extraction
        if not is_recoverable(errors):
            raise ExtractionError(errors, extraction)

        messages.append({"role": "assistant", "content": resp.content})
        messages.append({"role": "user", "content":
            f"Your previous extraction failed validation:\n"
            f"{format_errors(errors)}\n\n"
            f"Re-examine the original document and call extract_invoice again, "
            f"correcting only the listed errors."
        })
    raise ExtractionError(errors, extraction)
```

Trois choses rendent cette boucle efficace : (1) le tour assistant est ajouté *verbatim* (même règle que la boucle agentique en Section 2), (2) le follow-up nomme les champs échoués **spécifiques**, et (3) le retry utilise tool use avec un outil forcé, donc la forme de réponse reste stable entre tentatives.

### Quand le retry n'aidera PAS

Les retries corrigent les **erreurs de format et structurelles** (mauvais noms de champs, valeurs mal placées, line items qui ne somment pas). Ils ne corrigent **pas** une information réellement absente de la source. Si la facture PDF ne contient pas d'ID fiscal, aucun retry ne le produira — et une boucle de retry agressive peut pousser le modèle à en halluciner un pour passer la validation. Détectez "absent in source" dans votre validateur et arrêtez la boucle avec un null structuré + raison au lieu de réessayer.

### Signaux d'auto-correction dans le schéma

Intégrez la validation *dans l'extraction elle-même* afin que le modèle traite les contradictions pendant la génération :

- **`calculated_total` à côté de `stated_total`** — le modèle émet les deux ; votre validateur signale les écarts. Le modèle s'auto-corrige souvent dans la même réponse parce que calculer la valeur force à regarder chaque ligne.
- **Booléens `conflict_detected`** — pour des documents où deux sections divergent (date d'en-tête vs date de pied de page, compteur résumé vs longueur de liste), faire émettre un booléen par famille de conflit. Cela transforme l'ambiguïté en signal structuré routable.

```json
{
  "line_items": [{"desc": "Widget A", "amount": 150.00},
                 {"desc": "Widget B", "amount": 300.00}],
  "calculated_total": 450.00,
  "stated_total": 500.00,
  "total_discrepancy": true,
  "conflict_detected": false,
  "detected_pattern": "missing_tax_line"
}
```

### `detected_pattern` pour analyser les dismissals

Ajoutez un champ `detected_pattern` (ou `rule_id`) à chaque finding. Lorsque les développeurs rejettent des findings dans votre UI, groupez les rejets par `detected_pattern` pour voir quelles catégories sont bruyantes. Sans ce champ, votre seul signal est le taux FP agrégé — qui dit *qu'il* y a un problème, pas *quoi* corriger.

### Erreurs sémantiques vs syntaxiques

Tool use avec un schéma JSON strict (Domaine 4.3) élimine les erreurs de **syntaxe** — JSON mal formé, champs requis manquants, mauvais types. Il n'élimine **pas** les erreurs **sémantiques** — valeurs qui ne somment pas, dates hors plage du document, contenu plausible mais faux. La boucle validation-retry est la couche qui gère les erreurs sémantiques. Piège fréquent d'examen : "nous avons ajouté un schéma JSON, pourquoi les line items sont-ils encore faux ?" — parce que les schémas ne font pas d'arithmétique.

## Tout assembler : workflow de réglage de précision

1. **Baseline FP/FN par catégorie.** Lancer un eval set ; grouper les findings par `detected_pattern` ; enregistrer precision/recall par catégorie, pas seulement globalement.
2. **Réécrire les instructions vagues en critères explicites** pour les principales catégories FP — règles "report when… / skip when…" avec exemples de code pour chaque côté.
3. **Ajouter 2–4 exemples few-shot par scénario ambigu,** incluant au moins une **paire contrastive** (ressemble à un problème, n'en est pas un) par catégorie bruyante. Envelopper dans des tags XML `<examples>` placés après les instructions, avant l'entrée live.
4. **Encapsuler dans une boucle validation-retry.** Utiliser tool use pour forcer le schéma ; valider les contraintes sémantiques (`calculated_total == stated_total`, ligne source citée requise) ; en cas d'échec ajouter l'erreur spécifique et réessayer une fois.
5. **Suivre `detected_pattern` et dismissals.** Trier chaque semaine par taux FP ; désactiver temporairement toute catégorie au-dessus du seuil pendant l'itération.
6. **Évaluer avant et après chaque changement** avec l'[Evaluation tool](https://console.anthropic.com/docs/en/test-and-evaluate/eval-tool) de la Console Anthropic ou [Promptfoo](https://www.promptfoo.dev/docs/getting-started/).

## Points d'attention style examen

- Face à un prompt qui dit "be conservative" ou "only report high-confidence findings", reconnaître que ce n'est **pas** la bonne correction pour un taux élevé de faux positifs — critères catégoriels + few-shot le sont.
- Choisir le bon nombre d'exemples few-shot : **2–4 ciblés** (pas 10, pas 1).
- Reconnaître que les exemples few-shot doivent inclure du **raisonnement** (pourquoi cette action plutôt que les alternatives), pas seulement entrée/sortie, surtout pour les cas ambigus comme escalade ou sélection d'outil.
- Identifier quand un retry aide (erreur de format/structure, line-items-don't-sum) vs quand il n'aide pas (information réellement absente de la source). Ne pas retry ce dernier cas.
- Distinguer erreurs de validation **sémantiques** et erreurs de **syntaxe de schéma**. Tool use corrige les secondes ; une boucle validation-retry corrige les premières.
- Connaître le rôle des champs d'auto-correction : `calculated_total` vs `stated_total`, `conflict_detected`, `detected_pattern`.
- Pour Sample Question 3 (escalade d'agent assurance) : la bonne réponse est **critères d'escalade explicites + exemples few-shot**, pas scores de confiance auto-déclarés, pas modèle classifieur séparé, pas analyse de sentiment.

## Références

- [Prompt engineering overview — Claude API docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — page canonique ; liste la pile de techniques (be clear and direct → multishot → chain-of-thought → XML tags → system prompts → prefill → chain prompts) par ordre de priorité.
- [Be clear, contextual, and specific](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/be-clear-and-direct) — guidance fondationnelle Anthropic derrière le Domaine 4.1.
- [Use examples (multishot prompting)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/multishot-prompting) — source officielle de la recommandation 2–4 exemples, anatomie des exemples et wrapping XML.
- [Let Claude think (chain of thought)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/chain-of-thought) and [Use XML tags](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/use-xml-tags) — séparer raisonnement et sortie, empêcher exemples, instructions et entrée live de se mélanger.
- [Extended thinking tips](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/extended-thinking-tips) — multishot reste utile avec extended thinking ; préférer les instructions haut niveau aux étapes prescriptives.
- [Increase output consistency (prefill)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/increase-consistency) and [Reduce hallucinations](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/reduce-hallucinations) — techniques derrière "forcer une accolade JSON de départ" et "exiger une ligne source citée".
- [Evaluate prompts in the developer console](https://www.anthropic.com/news/evaluate-prompts) and [Using the Evaluation tool](https://console.anthropic.com/docs/en/test-and-evaluate/eval-tool) — Anthropic Workbench pour cas de test, comparaison côte à côte et notation humaine en 5 points.
- [Promptfoo getting started](https://www.promptfoo.dev/docs/getting-started/) — framework d'évaluation tiers avec evals code-graded, model-graded et classification sur 60+ providers.
- [Building effective agents — Anthropic Engineering](https://www.anthropic.com/engineering/building-effective-agents) — cadre le prompt engineering comme l'étape plus simple avant l'infrastructure ML (la même logique derrière la bonne réponse de Sample Question 3).
