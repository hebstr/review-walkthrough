# Refactor plan — review-walkthrough

## Diagnostic

SKILL.md : ~424 lignes, fichier unique, pas de sous-agents ni templates.
Mélange orchestration, batch triage, walkthrough interactif, intégration Ouroboros, wrap-up et persistance dans un seul prompt.

### Risques

- Attention decay sur les instructions en fin de fichier (Ouroboros, persistance)
- Branching combinatoire (batch x adversarial x ouroboros x context) rend le comportement imprévisible
- Débogage difficile quand le skill dévie

## Chantier 1 : Extraire en sous-agents

| Sous-agent | Responsabilité | Source |
|---|---|---|
| `agents/orchestrator.md` | Step 0 : parsing args, context detection, calibration, lancement reviewer | ~55 lignes |
| `agents/batch-triage.md` | Step 1b : pré-verdict, tables, exécution batch, post-fix hooks | ~100 lignes |
| `agents/ouroboros-bridge.md` | Intégration Ouroboros : détection, QA, advocate/devil's advocate, lateral think, evaluate, drift | ~50 lignes |

SKILL.md garde : extraction findings (Step 1), walkthrough interactif (Step 2), wrap-up (Step 3), persistance (Step 4).
Objectif : SKILL.md < 200 lignes.

## Chantier 2 : Externaliser les données statiques

| Fichier | Contenu |
|---|---|
| `templates/calibration-personal.md` | Bloc de calibration contexte personal |
| `templates/calibration-internal.md` | Bloc de calibration contexte internal |
| `docs/autofix-whitelist.md` | Patterns mécaniques éligibles auto-fix |
| `docs/postfix-hooks.md` | Table file → lock command |
| `templates/transparency-glossary.md` | Glossaire des mécanismes Ouroboros |

## Chantier 3 : Réduire la redondance

- Transparency : définie 3 fois (status block Step 1, inline Step 2b, wrap-up Step 3) → définir une fois, référer ailleurs
- Conditions Ouroboros disponible/pas disponible : apparaît 5+ fois → centraliser dans `agents/ouroboros-bridge.md`
- Exemples de format inline (FR/EN) : les mettre dans les templates, pas dans les instructions

## Chantier 4 : Simplifier les arbres de décision

- Liste exhaustive des tiers basse sévérité (14 noms) → simplifier en règle : "tier explicitement marqué non-bloquant ou cosmétique"
- Conditions imbriquées adversarial + ouroboros → le sous-agent ouroboros-bridge gère ça, SKILL.md n'a pas besoin de connaître les détails

## Ordre d'exécution

1. **Sous-agents** (chantier 1) — le plus gros gain, débloque le reste
2. **Templates/docs** (chantier 2) — mécanique, rapide une fois la structure en place
3. **Redondance** (chantier 3) — nettoyage du SKILL.md allégé
4. **Simplification** (chantier 4) — polish final

## Critères de succès

- [ ] SKILL.md < 200 lignes
- [ ] Chaque sous-agent < 100 lignes
- [ ] Aucune instruction dupliquée entre fichiers
- [ ] Comportement identique avant/après (tester avec `/review-walkthrough SKILL.md`)
