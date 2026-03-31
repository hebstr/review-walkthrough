# Prompt de continuation — refactor review-walkthrough

Lis `REFACTOR-PLAN.md` et `SKILL.md`. Le plan décrit un refactor en 4 chantiers pour découper ce skill monolithique en sous-agents, templates et docs.

**Approche incrémentale** — on ne fait pas tout d'un coup. On itère chantier par chantier.

## Déjà fait

### Chantier 1a : Orchestrateur (2026-03-31)
- `templates/calibration-personal.md` et `templates/calibration-internal.md` créés (blocs de calibration extraits verbatim)
- `agents/orchestrator.md` créé (59 lignes) — Step 0 complet (parsing args, context detection, calibration injection, lancement reviewer)
- Step 0 de SKILL.md réduit de ~55 lignes à 7 (délégation vers orchestrateur)
- SKILL.md : 424 → 376 lignes

### Bias refactor (2026-03-31)
- Advocate/Devil's Advocate supprimé (same-model bias déguisé en débat)
- Remplacé par cross-model validation à deux niveaux :
  - L1 (intra-famille) : Agent tool avec modèle alterné (Opus↔Sonnet), sur tous les findings Important+
  - L2 (cross-provider) : `ouroboros_evaluate` avec `trigger_consensus: true` via OpenRouter, sur Blocking/Required (`--adversarial`) ou divergence L1
- Fallback single-model multi-perspective supprimé — L2 indisponible sans `OPENROUTER_API_KEY`
- Sections modifiées : glossaire, mechanism transparency examples, Ouroboros integration "During re-evaluation", wrap-up mechanisms, evaluate fallback
- Design spec (memory) mise à jour : `project_review_walkthrough_bias_refactor.md`

## Prochaine étape : Chantier 1b — Batch triage

Extraire `agents/batch-triage.md` depuis le Step 1b de SKILL.md (~100 lignes). Le sous-agent :
- Reçoit : la liste des findings extraits par Step 1, le contexte détecté
- Fait : rapid pre-verdict, classification auto-fix/auto-reject/manual, présentation du triage, batch execution, post-fix hooks
- Retourne : le bucket manual pour le walkthrough interactif (Step 2), les résultats batch pour le wrap-up (Step 3)

Contraintes identiques : < 100 lignes, pas de duplication, comportement identique avant/après.

## Ce qu'on ne fait PAS encore

- agents/ouroboros-bridge.md (chantier 1c)
- docs/autofix-whitelist.md, docs/postfix-hooks.md (chantier 2)
- templates/transparency-glossary.md (chantier 2)
- Réduction de redondance (chantier 3)
- Simplification des arbres de décision (chantier 4)

## Validation en attente

Un test live avec `/review-walkthrough --adversarial` sur un vrai projet pour vérifier le comportement du nouveau mécanisme cross-model L1/L2. À faire avant de continuer les chantiers.
