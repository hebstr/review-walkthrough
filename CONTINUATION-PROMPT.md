# Prompt de continuation — refactor review-walkthrough

Colle ce prompt dans une nouvelle conversation Claude Code, depuis le répertoire `~/.claude/skills/review-walkthrough/`.

---

Lis `REFACTOR-PLAN.md` et `SKILL.md`. Le plan décrit un refactor en 4 chantiers pour découper ce skill monolithique (~424 lignes) en sous-agents, templates et docs.

**Approche incrémentale** — on ne fait pas tout d'un coup. Exécute uniquement le chantier 1a (premier sous-agent), teste, puis on itère.

## Étape 1 : Extraire l'orchestrateur seul

1. Crée `templates/calibration-personal.md` et `templates/calibration-internal.md` — extrais les blocs de calibration verbatim du Step 0c de SKILL.md (les blocs "Context: this is a personal..." et "Context: this is internal...").

2. Crée `agents/orchestrator.md` — extrais le Step 0 complet (0a→0d) de SKILL.md. Le sous-agent :
   - Reçoit : target, flags (--reviewer, --adversarial, --batch/--no-batch)
   - Fait : parsing args, context detection (path heuristics → memory check → fallback ask), calibration injection (lit les templates), lancement du reviewer via Skill tool
   - Retourne : le rapport du reviewer est dans la conversation, prêt pour Step 1

   Le sous-agent doit avoir un frontmatter avec `name` et `description`. Limite : < 100 lignes.

3. Réécris le Step 0 de `SKILL.md` — remplace les ~55 lignes du Step 0 par une invocation Agent vers `agents/orchestrator.md` (~5 lignes). Le reste de SKILL.md (Steps 1→4) ne bouge pas. Le contrat est : "si un target est fourni, lance l'orchestrateur ; quand il a fini, le rapport est dans la conversation, continue à Step 1."

## Étape 2 : Vérification

Après les modifications :
- Lis chaque fichier créé/modifié et vérifie : pas de sections manquantes vs l'original, pas de références cassées entre fichiers
- Vérifie que SKILL.md fait toujours référence aux concepts du Step 0 dont les autres Steps dépendent (notamment : le contexte détecté pour le transparency status block du Step 1, les flags parsés pour le batch mode)
- Compte les lignes de SKILL.md — note la réduction

## Contraintes

- Le comportement observable doit être identique avant/après
- Texte en anglais dans les fichiers (c'est du prompt, pas du user-facing)
- Ne duplique aucune instruction entre SKILL.md et le sous-agent
- Si une info du Step 0 est nécessaire dans un Step ultérieur (ex: contexte détecté, reviewer utilisé), le sous-agent doit la reporter dans sa réponse pour que SKILL.md puisse l'utiliser

## Étape 3 : Refactor du mécanisme anti-bias (section Ouroboros integration)

**Contexte :** session du 2026-03-31 sur sync-files. On a découvert que `ouroboros_qa` ne supporte PAS `trigger_consensus` — le "cross-model judge" tournait en same-model sans le signaler. Le design spec complet est dans `~/.claude/projects/-home-julien--claude-skills-review-walkthrough/memory/project_review_walkthrough_bias_refactor.md`.

**Déjà fait (appliqué directement dans SKILL.md) :**
- Cross-model judge corrigé : utilise `ouroboros_evaluate` avec `trigger_consensus: true` au lieu de `ouroboros_qa` (section Ouroboros integration, ~line 395)
- Model transparency ajoutée : le nom du modèle externe doit être reporté dans les lignes de transparence (~line 397)
- Warning explicite : `ouroboros_qa` ne supporte PAS `trigger_consensus` (~line 399)
- Transparency status, glossaire, mechanism examples et wrap-up mis à jour avec les détails d'outils (`ouroboros_qa` vs `ouroboros_evaluate`) et le model name

**Reste à faire — design deux niveaux :**

| Niveau | Mécanisme | Quand | Coût | Vraie diversité ? |
|--------|-----------|-------|------|-------------------|
| 1. Intra-famille | Agent tool avec `model: "sonnet"` si on est sur Opus (ou inversement) | Tous les findings Important+ | Nul | Partielle (même famille, poids différents) |
| 2. Cross-provider | `ouroboros_evaluate` avec `trigger_consensus: true` + OpenRouter | Blocking/Required, ou quand niveau 1 diverge | Tokens OpenRouter | Oui |

**Changements restants :**
1. Supprimer l'Advocate/Devil's Advocate (deux appels `ouroboros_qa` avec quality_bars opposés) — c'est du same-model bias déguisé en débat
2. Remplacer par : le modèle principal fait sa réévaluation (Step 2b tel quel), puis un Agent spawn avec le modèle alterné réévalue indépendamment en isolation
3. Si les deux convergent → verdict clair. Si divergence → escalade vers niveau 2
4. Supprimer le fallback "multi-perspective same-model" de `ouroboros_evaluate` sans OpenRouter — ça n'apporte pas de diversité réelle
5. Redéfinir `--adversarial` : niveau 1 toujours actif sur Important+, `--adversarial` force le niveau 2 sur tous les Blocking/Required (pas seulement en cas de divergence)

**Sections restant à modifier dans SKILL.md :**
- Glossaire (~line 105-113) : remplacer Advocate/DA et cross-model par les deux niveaux
- Mechanism transparency examples (~line 243-247) : nouveaux exemples avec Agent intra-famille + evaluate cross-provider
- Section Ouroboros integration "During re-evaluation" (~line 386-401) : réécriture complète pour remplacer Advocate/DA par Agent intra-famille
- Wrap-up mechanisms example (~line 327) : mettre à jour avec les niveaux

## Ce qu'on ne fait PAS encore

- agents/batch-triage.md (chantier 1, étape suivante)
- agents/ouroboros-bridge.md (chantier 1, étape suivante)
- docs/autofix-whitelist.md, docs/postfix-hooks.md (chantier 2)
- templates/transparency-glossary.md (chantier 2)
- Réduction de redondance (chantier 3)
- Simplification des arbres de décision (chantier 4)

Quand les étapes 1-2 sont faites, mets à jour `REFACTOR-PLAN.md` pour noter que le chantier 1a (orchestrateur) est terminé. L'étape 3 (bias refactor) peut être faite indépendamment — elle touche la section Ouroboros, pas le Step 0.
