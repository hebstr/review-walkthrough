# Glowup Code Review — Synthese multi-agents

> Synthese de 9 analyses croisees (3 modeles x 3 niveaux de profondeur) sur l'amelioration du workflow de code review.
> Date : 2026-03-25

---

## Verdict unanime sur le workflow actuel

**Ton workflow est solide et au-dessus de la moyenne** — les 9 agents convergent sur ce point.
Le combo `critical-code-reviewer` (adversarial) + `review-walkthrough` (contre-review interactive) est un pattern rare, discipline, et structurellement superieur a une simple passe unique.

### Forces identifiees (consensus 9/9)

- **Friction productive** : le reviewer attaque, le walkthrough force a digerer chaque point — anti-rubber-stamping
- **Systematicite** : appliquer le process a chaque PR elimine le biais de selection ("ce code est trivial, pas besoin de review")
- **Separation des roles** : detection de defauts (passe 1) vs comprehension/validation (passe 2)
- **Human-in-the-loop** : tu gardes le jugement final, tu ne delegues pas aveuglement

---

## Faiblesses et angles morts (par consensus — 9 agents + revue independante)

### 0. Workflow purement reactif — pas de design review (revue independante)

Tout le process se declenche *apres* que le code est ecrit. Pour un nouveau module ou un refactor majeur, c'est trop tard — les problemes d'architecture auraient du etre identifies en amont. Un bug fix de 3 lignes ne passe pas par cette etape ; un nouveau module, si.

**Remede** : ajouter une phase -1 de design review leger (intention, approche, alternatives considerees) pour les changements non triviaux.

### 1. Biais de confirmation entre les deux passes (9/9 agents)

Les deux skills tournent sur le meme modele (Claude). Le walkthrough travaille *sur le rapport*, pas sur le code en partant de zero. Si le critical-reviewer rate un bug, le walkthrough ne le rattrapera pas — il t'amenera a conclure que tout ce qui n'est pas dans le rapport est OK.

**Risque concret** : faux sentiment de couverture. Deux passes, mais un seul "cerveau".

**Remedes proposes** :
- Varier occasionnellement le modele pour la contre-review (diversite des biais, pas qualite intrinseque)
- Reformuler le walkthrough pour qu'il reparte du code source independamment, puis compare avec le rapport
- Ajouter une review humaine (pair) sur les pieces critiques, meme occasionnellement
- **Diversifier avec un skill a angle unique** (revue independante) : ajouter un skill specialise (secu, perf, ou `ouroboros:qa`) pour une vraie contre-perspective plutot qu'un second LLM generaliste qui raisonne dans le meme espace

### 2. Absence de couche pre-review automatisee (9/9 agents)

Un LLM ne remplace pas les verifications deterministes. Sans linting/formatting en amont, la review IA perd du temps sur du bruit syntaxique.

**Action immediate — setup `pre-commit`** :

| Langage | Outils | Role |
|---------|--------|------|
| R | `air`, `jarl` | Lint + format |
| Python | `ruff` (lint + format), `pyright` | Lint + format + type checking |
| Quarto | `quarto check` / `quarto render` | Validation structure |
| Transversal | `codespell`, secrets scanning | Typos, fuites de secrets |

### 3. Reviewer generaliste vs profil data/biostat (9/9 agents)

Le `critical-code-reviewer` est calibre pour du code generaliste. Il ne couvre pas les risques specifiques a ton domaine :

- **Reproductibilite** : seeds non fixees, dependances non verrouillees, chemins absolus
- **Data leakage** : preprocessing qui fuit du test set dans le train set
- **Coercitions implicites** : `character` -> `numeric` avec `NA` silencieux en R
- **Tidyeval/NSE** : `.data[[]]` pronoms, `across()`, `pick()` — angles morts frequents des reviewers LLM

### 4. Pas de tests dans la boucle de review (8/9 agents)

La review detecte des problemes mais rien ne garantit qu'ils sont corriges *et restent corriges*. Un code sans tests est intrinsequement non-reviewable.

**Outils recommandes** :

| Langage | Tests unitaires | Validation donnees |
|---------|----------------|-------------------|
| R | `testthat3`, `covr` | `pointblank`, assertions inline (`stopifnot`) |
| Python | `pytest`, `hypothesis` | `pandera`, `great_expectations` |

### 5. Pas d'adaptation au type de code (7/9 agents)

Le workflow est "one-size-fits-all". Un script d'exploration jetable ne merite pas la meme rigueur qu'un pipeline de production clinique.

**Stratification proposee** :

| Niveau | Type de code | Intensite review |
|--------|-------------|-----------------|
| **Exploratoire** | Notebook EDA, script one-shot | Lint auto + review legere (style + logique evidente) |
| **Production** | Pipeline data, fonctions reutilisees | Review adversariale complete |
| **Critique** | Analyse reglementaire, pipeline clinique | Review + validation stat + pair humain |

### 6. Pas de memoire longitudinale (6/9 agents)

Chaque session repart de zero. Si tu fais la meme erreur 5 fois, personne ne te le dit. Pas de feedback loop pour ameliorer tes habitudes.

### 7. Walkthrough potentiellement passif (5/9 agents)

Sans decision explicite par finding, le walkthrough produit de la comprehension mais pas necessairement du changement. Risque de "decision fatigue" et de validation par lassitude.

### 8. Pas de triage humain entre les deux passes (revue independante)

Le critical-code-reviewer est maximaliste ("zero tolerance"). Sans tri intermediaire, le walkthrough traite tous les findings avec la meme intensite. Risque de **bikeshedding assiste par IA** : debattre du nommage de variables pendant que la logique de jointure est fausse.

**Remede** : 2 minutes de tri manuel entre les deux passes :
- **Bloquant** : bug, securite, corruption de donnees -> traitement immediat
- **Important** : performance, maintenabilite -> walkthrough approfondi
- **Mineur** : style, naming -> note rapide ou rejet
- **Rejete** : hors scope, choix delibere documente -> ignore

### 9. Findings non actionnables (revue independante + Sonnet)

"Potentiel probleme de jointure" = bruit. "Ligne 42, jointure potentiellement dupliquante, verifier avec `distinct()` avant" = signal. Un finding doit proposer un fix actionnable, pas juste signaler.

---

## Recommandations concretes — Plan d'action

### Immediat (cette semaine)

- [ ] **Pre-commit hooks** : `ruff` (Python), `air`/`jarl` (R), `codespell` — eliminer le bruit avant la review IA
- [ ] **`REVIEW_CONTEXT.md` par projet** : contexte metier, invariants connus, decisions architecturales, contraintes reglementaires — a injecter dans chaque invocation du reviewer
- [ ] **Structurer la sortie du walkthrough** : chaque finding -> statut explicite (`ACCEPTED` / `REJECTED` / `DEFERRED` / `NOTED`) avec justification
- [ ] **Triage humain rapide** (2 min) entre critical-code-reviewer et walkthrough : classer bloquant / important / mineur / rejete — le walkthrough ne traite en profondeur que les bloquants et importants *(revue independante)*
- [ ] **Enrichir le `CLAUDE.md`** avec les anti-patterns R/Python specifiques a tes projets *(revue independante)*
- [ ] **Traiter differemment selon le type de changement** : bug fix 3 lignes ≠ nouveau module *(revue independante)*

### Court terme (ce mois)

- [ ] **Skill ou prompt "data-reviewer"** specialise : checklist NA, jointures, reproductibilite, hypotheses statistiques, data leakage
- [ ] **Variantes de prompt par type de code** : R data transform ≠ Python API ≠ Quarto — adapter le focus du reviewer *(revue independante)*
- [ ] **Passe "defense de l'auteur"** dans le walkthrough : pour chaque finding retenu, generer l'argument le plus solide que l'auteur pourrait opposer, puis evaluer sa validite *(revue independante)*
- [ ] **Tests minimaux** : `testthat3`/`pytest` pour les fonctions critiques, `pandera`/`pointblank` pour les contrats de donnees
- [ ] **`TECH_DEBT.md` par repo** : logger les findings `DEFERRED` avec justification — revisiter periodiquement
- [ ] **`review_patterns.md`** : log des findings recurrents, regle "3+ occurrences = convention a standardiser dans le projet" (ajout au `CLAUDE.md` ou au linter) *(revue independante)*
- [ ] **Design review** pour les features non triviales — avant code, pas apres *(revue independante)*
- [ ] **Diversifier les perspectives** : ajouter un skill a angle unique (`ouroboros:qa`, secu, perf) en complement *(revue independante)*

### Moyen terme (ce trimestre)

- [ ] **CI/CD basique** : GitHub Actions avec lint + tests + `quarto render` — checks deterministes automatiques, review IA asynchrone et non-bloquante
- [ ] **Validation donnees a l'execution** : `pandera` (Python) / `pointblank` (R) dans les pipelines eux-memes, pas juste en review
- [ ] **Metriques de review** : tracker taux d'implementation des findings, recurrence des patterns, bugs post-merge
- [ ] **Review humaine occasionnelle** : communautes R (rOpenSci), pairs biostat — trimestrielle sur le code critique
- [ ] **Demander au LLM de generer des tests** plutot que de reviewer le code — un test qui echoue est un finding objectif, pas une opinion *(revue independante, Opus)*
- [ ] **Review architecturale trimestrielle** : patterns emergents, silos, dette technique *(revue independante)*

---

## Workflow cible

```
[Design review]             <- si changement non trivial : intention, approche, alternatives
        |
[Pre-commit hooks]          <- lint, format, type check, secrets (deterministe)
        |
[CI/CD]                     <- tests, validation donnees, quarto render
        |
[Auto-review du diff]       <- 30 sec, lecture humaine rapide
        |
[Injection de contexte]     <- REVIEW_CONTEXT.md + type de changement + scope
        |
[Pre-mortem personnel]      <- 3-5 lignes sur les zones de doute
        |
[critical-code-reviewer]    <- adversarial, calibre domaine data/biostat
  + skill angle unique       <- optionnel : secu, perf, ouroboros:qa
        |
[Triage humain rapide]      <- 2 min : bloquant / important / mineur / rejete
        |
[review-walkthrough]        <- bloquants + importants uniquement
  + defense de l'auteur      <- contre-argument le plus fort, puis evaluation
        |
[Decision humaine]          <- ACCEPTED / REJECTED / DEFERRED / NOTED
        |
[TECH_DEBT.md]              <- log des DEFERRED
[review_patterns.md]        <- log si finding recurrent (3+ = convention)
        |
[Merge]
```

Pour le code **critique** (biostat reglementaire) : ajouter une passe "statistical-validity-reviewer" et/ou une review humaine.

---

## Outils complementaires par stack

### R

| Outil | Usage |
|-------|-------|
| `air` + `jarl` | Lint + format automatique |
| `testthat3` + `covr` | Tests + couverture |
| `pointblank` | Validation de donnees en pipeline |
| `renv` | Reproductibilite des dependances |
| `targets` | Pipelines reproductibles avec cache |
| `goodpractice` | Analyse qualite globale (complexite, dependances) |

### Python

| Outil | Usage |
|-------|-------|
| `ruff` | Lint + format (remplace flake8/black/isort) |
| `mypy` / `pyright` | Type checking statique |
| `pytest` + `hypothesis` | Tests + property-based testing |
| `pandera` / `great_expectations` | Validation schema donnees |
| `uv lock` | Reproductibilite des dependances |
| `bandit` | Detection failles de securite |

### Quarto

| Outil | Usage |
|-------|-------|
| `quarto check` / `quarto render` | Validation structure et rendu |
| Review des outputs rendus | Graphiques lisibles ? Tableaux corrects ? |

---

## Checklist data/biostat (a integrer au reviewer)

### Statistiques

- [ ] Seed fixee pour toute operation stochastique
- [ ] Pas de data leakage (preprocessing apres split)

### Reproductibilite

- [ ] Dependances verrouillees (`renv.lock`, `uv.lock`)
- [ ] Pas de chemins absolus hardcodes
- [ ] Donnees brutes non modifiees in place
- [ ] Code relancable de zero sur machine vierge

---

## Pieges a eviter (revue independante)

| Piege | Description |
|---|---|
| Faux sentiment de rigueur | 2 passes IA ≠ review rigoureuse. Si tu ne lis plus le code toi-meme, tu as degrade ton process |
| Inflation de findings | 20 points -> 12 valides -> 8 appliques dont 5 cosmetiques. Beaucoup de temps, peu de valeur |
| Bikeshedding assiste | Le LLM adore le style. Il te fait debattre du naming pendant que la logique de jointure est fausse |
| "L'IA a dit donc c'est vrai" | Meme deux IA peuvent se tromper en concert. `critical-code-reviewer` est *confiant*, pas *juste* |
| Biais d'autorite | Apres un rapport long et confiant, tu tends a le croire malgre toi |
| Review exhaustive sur tout | Reserver le workflow complet aux PRs substantielles. Code exploratoire = gaspillage |
| Sur-automatiser | La review est aussi un moment de reflexion sur le design. Automatiser toute la boucle = perdre cet espace de recul |
| Limites sur la logique metier | Le LLM ne sait pas que `date_inclusion` peut contenir des dates futures dans un essai clinique |

---

## Convergence 9 agents + revue independante

La revue independante confirme les problemes identifies par les 9 agents (chambre d'echo, manque de contexte, pas de priorisation) et y ajoute des points que les agents n'avaient pas (ou insuffisamment) couverts :

| Point | Source | Apport distinct |
|---|---|---|
| Design review pre-implementation | Revue independante | Aucun des 9 agents n'a propose cette etape |
| Triage humain rapide entre les 2 passes | Revue independante | Les agents parlent de categorisation, mais pas d'un tri *entre* les passes |
| Regle "3+ occurrences = convention" | Revue independante | Formalise la capitalisation sur les findings recurrents |
| Diversification multi-skill (angle unique) | Revue independante | Les agents parlent de calibrage ; la revue propose des skills specialises complementaires |
| Passe "defense de l'auteur" | Revue independante (Sonnet) | Generer le contre-argument le plus fort avant de valider un finding |
| Auto-review du diff (30 sec) | Revue independante | Lecture humaine rapide avant d'envoyer a l'IA |
| Findings actionnables obligatoires | Revue independante + Sonnet | "Signaler" ≠ "proposer un fix" |

---

## Idees divergentes notables (points non-consensuels)

- **Inverser l'ordre conditionnel** (Haiku detaillee) : pour du code critique, faire le walkthrough *avant* le critical-reviewer — tu comprends l'intention d'abord, le reviewer fouille ensuite de facon plus ciblee
- **Attendre 24h avant d'implementer** (Sonnet tres approfondie) : revenir avec un regard frais pour identifier les corrections du reviewer qui auraient introduit de nouveaux bugs
- **Pre-mortem personnel** (Sonnet tres approfondie) : ecrire 3-5 lignes sur les zones de doute avant de soumettre — tu sais souvent ou ca cloche
- **Versionner les skills eux-memes** (Sonnet detaillee) : pour attribuer une regression de qualite a un changement de prompt vs un changement de code — pertinent en contexte d'audit biostat
- **ADR (Architecture Decision Records)** (Sonnet tres approfondie) : 5-10 lignes par decision non evidente pour eviter de re-debattre les memes questions a chaque review

---

## Matrice des contributions par agent

| Agent | Apport distinctif |
|-------|------------------|
| **Opus approfondie** | Analyse structurelle du biais "juge et partie", review comme apprentissage pas gatekeeping, litterature (SmartBear <400 lignes), property-based testing |
| **Opus detaillee** | Assertions dans le code (`relationship =`, `validate=`), diversifier les modeles pour casser les biais, exemples de code concrets |
| **Opus concise** | Post-mortem leger sur les bugs echappes, rotation des reviewers, documenter le "pourquoi" pas le "quoi" |
| **Sonnet approfondie** | Contexte metier injectable (`REVIEW_CONTEXT.md`), memoire longitudinale, biais de confirmation du walkthrough, separation temporelle review/implementation, pre-mortem |
| **Sonnet detaillee** | Structuration findings en decisions (ACCEPTED/REJECTED/DEFERRED/NOTED), `TECH_DEBT.md`, versionner les prompts de skills, metriques taux d'implementation |
| **Sonnet concise** | Sur-process sur code exploratoire, NSE/tidyeval comme angle mort LLM, seuil de declenchement (tout ne merite pas 2 passes) |
| **Haiku approfondie** | Exemples concrets de bugs biostat (p-hacking, leakage scaler, t-test naif), template walkthrough structure (tableau), CI/CD specifique data, plan d'action en 3 sprints |
| **Haiku detaillee** | Boucle de feedback fermee, inverser l'ordre conditionnel (walkthrough avant critical pour code critique), categorisation par severite des findings |
| **Haiku concise** | Review post-merge pour Quarto, pair programming asymetrique, ne pas faire porter au reviewer des attentes metier qu'il ne peut pas valider |

---

## Investigation outillage — Chainlink vs Ouroboros (2026-03-25)

### Chainlink (`dollspace-gay/chainlink`) — ecarte

Issue tracker local (SQLite + Rust) concu pour donner une memoire persistante aux assistants IA entre sessions. Handoff notes, breadcrumbs, hooks Claude Code.

**Verdict** : resout la continuite de session (reprendre ou on en etait) mais pas l'apprentissage longitudinal (detection de patterns recurrents). Le systeme de memoire natif de Claude Code (`~/.claude/projects/*/memory/`) + un `review_patterns.md` versionne couvre 80% du besoin sans dependance externe. Cout d'integration disproportionne (binaire Rust + hooks Python) pour le gain. **Passe.**

### Ouroboros (`Q00/ouroboros`) — retenu, integration profonde

Workflow engine specification-first avec Socratic questioning + ontological analysis. Boucle Interview -> Seed -> Execute -> Evaluate -> Evolve. 9 agents specialises ("The Nine Minds"), evaluation en 3 stages (mecanique, semantique, consensus multi-modele), detection de stagnation, lateral thinking.

**Deja installe** comme plugin Claude Code — les outils MCP sont disponibles dans l'environnement.

#### Mapping faiblesses GLOWUP -> features Ouroboros

| Faiblesse GLOWUP | Feature Ouroboros | Integration |
|---|---|---|
| §1 Biais de confirmation | Multi-Model Consensus (Advocate / Devil's Advocate / Judge) | En step 2b, sur findings Blocking/Required uniquement |
| §7 Walkthrough passif | `ooo qa` — verdict structure avec score 0-1 et breakdown dimensionnel | Propose sur findings ambigus (pas les evidences) |
| §3 Angles morts generalistes | Lateral thinking personas (Contrarian, Simplifier, Hacker, Architect, Researcher) | Via `ooo unstuck` quand le walkthrough tourne en rond |
| §9 Findings non actionnables | `ooo evaluate` stage 1 (mecanique) — lint/build/test | En wrap-up pour code critique |
| §6 Memoire longitudinale | Event sourcing — audit trail de chaque decision | Analyse retrospective periodique |

#### Strategie d'integration — profonde mais conditionnelle

L'integration est profonde (toutes les features ci-dessus sont dans le SKILL.md) mais chaque feature a un **seuil de declenchement explicite** pour eviter l'inflation :

| Feature | Seuil de declenchement | Justification |
|---|---|---|
| `ooo qa` (second avis) | Finding ambigu ou evaluation incertaine | Pas sur les faux positifs evidents ni les bugs confirmes |
| Advocate/Devil's Advocate | Findings Blocking ou Required uniquement | Les mineurs ne meritent pas le cout multi-modele |
| `ooo unstuck` (lateral thinking) | Walkthrough bloque sur un point de design (2+ echanges sans resolution) | Detection explicite de stagnation |
| `ooo evaluate` (validation finale) | Wrap-up sur code critique (production/reglementaire) | Pas pour un bug fix de 3 lignes |
| Drift measurement | Plus de 5 fixes appliques dans un meme walkthrough | Verification de coherence globale post-fixes |

**Principe directeur** : Ouroboros est propose a l'utilisateur comme option, jamais impose. Le skill reste fonctionnel sans Ouroboros installe.

---

## Plan d'attaque — ameliorations du SKILL.md (2026-03-25)

Scope : uniquement ce qui est dans le perimetre du skill `review-walkthrough`.

### Phase 1 — Structure et priorisation
1. [x] **Triage par severite** : quand la review a des niveaux (Blocking/Required/Suggestion), traiter dans cet ordre plutot que l'ordre d'apparition
2. [x] **Statut explicite par finding** : chaque point se conclut par `ACCEPTED` / `REJECTED` / `DEFERRED` / `NOTED` avec justification one-liner
3. [x] **Edge cases** : review vide (0 findings) -> proposer une passe independante ; review a 1 finding -> traiter sans la mecanique "X points trouves"

### Phase 2 — Qualite de la re-evaluation + integration Ouroboros
4. [x] **Passe "defense de l'auteur"** : en 2b, avant de valider un finding, generer le meilleur contre-argument que l'auteur pourrait opposer, puis juger sa solidite
5. [x] **Findings actionnables** : en 2b, si le finding original ne propose pas de fix concret, en formuler un — ne jamais valider un "potentiel probleme" sans action claire
6. [x] **Re-evaluation depuis le code** : en 2b, repartir du code source plutot que du seul rapport pour attenuer le biais de confirmation
7. [x] **Hook `ooo qa`** : proposer un second avis Ouroboros sur les findings ambigus (Blocking/Required)
8. [x] **Hook `ooo unstuck`** : proposer le lateral thinking quand le walkthrough tourne en rond
9. [x] **Hook Advocate/Devil's Advocate** : integrer dans la defense de l'auteur sur les findings critiques

### Phase 3 — Robustesse
10. [x] **Rollback explicite** : en 2d, si regression detectee -> revert, explication, choix utilisateur (approche differente / skip / accepter le trade-off)
11. [x] **Wrap-up enrichi** : resume avec tableau des statuts, liste des `DEFERRED` a suivre *(fait en phase 1)*
12. [x] **Hook `ooo evaluate`** : proposer en wrap-up comme validation finale pour le code critique *(fait en phase 2)*
13. [x] **Hook drift measurement** : si plus de 5 fixes, verification de coherence globale *(fait en phase 2)*

### Phase 4 — Evals
14. [x] **Nouveaux cas d'eval** couvrant : triage severite (eval 4), edge case 0 findings (eval 5), edge case 1 finding (eval 6), defense de l'auteur (eval 7), findings non actionnables (eval 8). Evals 1-3 mis a jour pour verifier les statuts explicites. Total : 8 evals (avant : 3)
