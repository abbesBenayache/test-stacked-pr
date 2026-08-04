# test-stacked-pr

Ce repo est un terrain d'expérimentation pour comprendre, en pratique, le fonctionnement des **Pull Requests empilées** (*stacked PR*) sur GitHub.

---

## Cours : Les Pull Requests empilées (Stacked PR)

*Support de cours — niveau junior à confirmé. Rédigé à partir d'une expérimentation réelle menée dans ce repo (PR #1 à #15).*

### Introduction

Bonjour à toutes et à tous. Aujourd'hui, on va parler d'une feature GitHub relativement récente mais qui répond à un problème vieux comme le travail en équipe sur du code : **comment découper un gros changement en plusieurs revues indépendantes, sans se noyer dans les rebases manuels ?**

C'est le genre de sujet qu'on n'enseigne pas dans un cours de git classique, parce que c'est avant tout un problème de *flux de travail d'équipe* (workflow), pas un problème de commandes git. Mais comprendre la mécanique qui se cache derrière est essentiel pour tout ingénieur qui travaille avec plusieurs personnes sur la même base de code.

### 1. Avant cette feature : le problème qu'on cherchait à résoudre

Historiquement, pour reviewer du code, deux approches dominaient — et les deux avaient un vrai coût :

1. **La PR géante.** Un développeur travaille seul pendant deux semaines sur une branche, puis ouvre une PR de 2000 lignes. Le reviewer se retrouve à devoir tout avaler d'un coup : fatigue de revue, bugs qui passent entre les mailles, feedback qui arrive trop tard pour être corrigé facilement.
2. **Les branches empilées à la main.** Des équipes plus matures (notamment chez Meta ou Google, en interne, ou via des outils tiers comme **Phabricator** ou **Graphite**) avaient déjà compris l'intérêt de découper le travail en petites PR séquentielles, où la PR B dépend du contenu de la PR A. Mais sans outillage natif, chaque merge du bas de la pile déclenchait une cascade d'opérations manuelles : changer la base de la PR du dessus, la rebaser, la force-pousser, vérifier qu'il n'y a pas de conflit — et recommencer pour chaque PR au-dessus. Sur une pile de 5 PR, une seule erreur humaine dans cette chaîne cassait tout.

GitHub a introduit le support natif des PR empilées pour éliminer ce travail manuel répétitif : automatiser la partie mécanique, pour que les équipes se concentrent sur la partie qui a de la valeur — la revue de code elle-même.

> **mot-clé : stacked PR** = "PR empilée" — de l'anglais *to stack*, empiler des objets les uns sur les autres. GitHub a choisi ce mot parce que chaque PR repose littéralement sur la précédente, comme une pile d'assiettes : retire l'assiette du bas (merge la PR-1), et toutes celles au-dessus doivent se réajuster.

### 2. La valeur ajoutée, concrètement

- **Petites PR = revues rapides et précises.** Un reviewer évalue 100 lignes cohérentes plutôt que 2000 lignes mélangées.
- **Parallélisme.** Tu peux continuer à écrire la PR-2 pendant que la PR-1 est en cours de revue, sans attendre son merge.
- **Zéro rebase manuel dans le cas nominal.** GitHub retargete et rebase la suite de la pile pour toi — c'est la partie qu'on va observer techniquement plus bas.

### 3. Ce que fait GitHub, techniquement, derrière l'écran

Une pull request, ce n'est fondamentalement qu'une comparaison entre deux pointeurs : une branche `head` (ce que tu proposes) et une branche `base` (où tu veux l'intégrer). Rien n'empêche que `base` soit une autre branche de feature plutôt que `main` — c'est tout le principe du stacked PR : PR-B a pour `base` la branche de PR-A, au lieu de `main`.

> **mot-clé : base branch** = "branche de base" — la branche sur laquelle une PR calcule son diff. C'est le point de repère, comme le niveau zéro d'un plan de construction : tout ce qui est mesuré au-dessus est relatif à ce niveau.

Voici la séquence exacte observée dans ce repo (PR #1 → PR #2, extraite via `gh api .../timeline`) quand la PR du bas (PR #1, base=`main`) est mergée alors que PR #2 (base=`feature/feature-1`) est encore ouverte :

| Heure | Événement | Explication |
|---|---|---|
| T+0s | PR #1 mergée dans `main` | Le "socle" de la pile disparaît en tant que branche active |
| T+1s | `automatic_base_change_succeeded` | GitHub détecte que la base de PR #2 vient d'être absorbée dans `main`, et **change automatiquement** le `base_ref` de PR #2 vers `main` |
| T+2s | `head_ref_force_pushed` (nouveau commit, nouveau SHA) | GitHub **rejoue** le commit de PR #2 par-dessus le nouveau `main` — un vrai rebase serveur, pas un simple changement d'affichage |

> **mot-clé : rebase** = littéralement "changer de base" (*re-* = à nouveau, *base* = fondation). On prend des commits et on les rejoue par-dessus un nouveau point de départ, comme si on les avait écrits directement là. Contrairement à un `merge`, qui crée un nouveau commit reliant deux historiques, le `rebase` réécrit l'historique en donnant de nouveaux parents (et donc de nouveaux SHA) aux commits.

> **mot-clé : force-push** = "pousser en force" (*to force*). Normalement, `git push` refuse d'écraser un historique distant qui a divergé, par sécurité. Le "force" dit à git "écrase quand même, je sais ce que je fais" — ici, c'est GitHub lui-même qui force-push le commit rejoué à la place de l'ancien, en coulisses, sans intervention humaine.

**Point crucial** : si ce rebase automatique tombe sur un vrai chevauchement textuel (la même ligne modifiée à la fois dans la pile et ailleurs), il échoue silencieusement côté automatisation et la PR passe en état `CONFLICTING`. GitHub ne devine jamais qui doit "gagner" — il applique la règle standard de fusion à trois points (*three-way merge*), et s'il ne peut pas réconcilier automatiquement deux versions d'une même zone de texte, il te rend la main, exactement comme un `git rebase` qui s'arrête sur un conflit.

### 4. Ce que nos exercices ont prouvé, en pratique

On a mené trois expériences dans ce repo pour comparer les comportements :

**Exercice 1 — sans stacked PR.** `exo1-feature-B` a été créée depuis `main`, pas depuis `exo1-feature-A`, alors que B dépendait logiquement du contenu de A. Résultat : après le merge de A, PR-B s'est retrouvée en conflit (`CONFLICTING`), et il a fallu un `git merge origin/main` manuel, une résolution à la main, et un nouveau push. Aucune automatisation.

**Exercice 2 — avec stacked PR, cas isolé.** Même dépendance logique, mais B créée depuis A. Après le merge de A, GitHub a déclenché `automatic_base_change_succeeded` puis un rebase automatique (`head_ref_force_pushed`) sans aucune commande git tapée par un humain. PR-B est passée directement à `mergeable: MERGEABLE`, `mergeStateStatus: CLEAN`.

**Exercice 3 — la limite du stacked PR.** On a rejoué le scénario de l'exercice 2, mais avec une troisième branche indépendante (`exo3-feature-C`), simulant un collègue qui modifie la même zone du fichier en parallèle, hors de la pile. Merge de C, puis merge de A : `automatic_base_change_succeeded` s'est bien déclenché sur PR-B, mais le rebase automatique a cette fois échoué — conflit réel à résoudre manuellement, comme en exercice 1.

### 5. Tableau comparatif

| | Sans stacking (Exo 1) | Stacking isolé (Exo 2) | Stacking + tiers concurrent (Exo 3) |
|---|---|---|---|
| B connaît le commit de A dès le départ | Non | Oui | Oui |
| Action après merge de A | Aucune automatique | `automatic_base_change_succeeded` + rebase auto | `automatic_base_change_succeeded` puis échec |
| Résolution manuelle nécessaire | Oui | Non | Oui |
| Cause du conflit éventuel | Historique divergent (A jamais intégré dans B) | — | Modification tierce sur la même zone, hors de la pile |

### 6. Quand utiliser cette feature (et quand s'en méfier)

**Pertinent** quand :
- une fonctionnalité se découpe naturellement en étapes séquentielles (ex : migration de schéma, puis code qui l'utilise, puis suppression de l'ancien code) ;
- l'équipe pratique le *trunk-based development* — retourner vite et souvent vers `main` plutôt que laisser vivre de longues branches parallèles — et veut un historique linéaire et des revues courtes.

**À surveiller** :
- **le stacked PR ne protège jamais contre les collisions venant de l'extérieur de la pile** (exercice 3) — il automatise uniquement la synchronisation interne à ta propre chaîne de branches ;
- le rebase automatique change les SHA des commits : toute référence externe à un ancien commit (lien partagé, cache CI) devient obsolète après un `automatic_base_change_succeeded` ;
- sur une pile longue, une modification instable en bas déclenche un rebase en cascade de tout ce qui est au-dessus.

### Conclusion

Le stacked PR n'est pas une nouvelle façon de faire du git : c'est une automatisation d'un pattern (les *stacked diffs*) que les équipes matures pratiquaient déjà manuellement. La feature élimine la partie mécanique — retargeting et rebase — mais ne t'exempte jamais de la responsabilité fondamentale de tout travail collaboratif sur du code : savoir que ta branche peut entrer en collision avec le travail de quelqu'un d'autre, et savoir résoudre ce conflit quand l'automatisation atteint ses limites.
