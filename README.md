# test-stacked-pr

Reverse-engineering du mécanisme de **stacked PR** de GitHub : comportement observé, mécanique interne, et limites — via une série d'expérimentations contrôlées directement dans ce repo (PR #1 à #15).

---

## Stacked PRs sur GitHub : ce qui se passe réellement au merge

### Contexte

Découper un gros changement en plusieurs revues indépendantes est un problème ancien. Deux approches dominaient avant que GitHub n'outille nativement les PR empilées :

1. **La PR monolithique.** Une branche qui vit plusieurs semaines, puis une revue de plusieurs milliers de lignes d'un coup — fatigue de revue, régressions qui passent entre les mailles, feedback trop tardif pour être actionnable.
2. **L'empilement manuel.** Des équipes outillées (Meta, Google, ou via des produits tiers comme Phabricator ou Graphite) pratiquaient déjà le découpage séquentiel — PR B basée sur PR A — mais sans support natif : chaque merge du bas de la pile imposait de retargeter, rebaser et force-pusher manuellement tout ce qui se trouvait au-dessus. Sur une pile de cinq PR, une seule étape oubliée cassait toute la chaîne.

   Exemple sur une pile à 3 niveaux (`auth-api` → `auth-ui` → `auth-tests`), après merge de `auth-api` dans `main` :

   ```bash
   gh pr edit auth-ui --base main          # 1. retarget manuel
   git checkout auth-ui && git rebase main && git push --force-with-lease   # 2. rebase manuel (nouveaux SHA)
   git checkout auth-tests && git rebase auth-ui && git push --force-with-lease   # 3. répercuter en cascade
   ```

   Oublier l'étape 3 laisse `auth-tests` pointer sur les anciens SHA de `auth-ui` : son diff affiche alors des changements déjà mergés ailleurs, sans lien évident avec la cause racine. C'est précisément ce que le stacking natif automatise (`automatic_base_change_succeeded` + `head_ref_force_pushed`, propagés en cascade).

GitHub a intégré cette mécanique nativement pour automatiser la partie purement opérationnelle — retargeting et rebase — et laisser l'attention humaine sur ce qui a réellement de la valeur : la revue.

### Mécanique interne

Une pull request n'est qu'une comparaison entre deux pointeurs : `head` (ce qui est proposé) et `base` (la cible d'intégration). Rien n'impose que `base` soit `main` — c'est exactement ce que fait un stacking : la PR du haut prend pour `base` la branche de la PR du bas, au lieu de `main`.

Séquence exacte capturée via `gh api .../timeline` sur ce repo, quand la PR du bas (base=`main`) est mergée alors que la PR du haut (base=`feature/feature-1`) est encore ouverte :

| T | Événement | Effet |
|---|---|---|
| T+0s | Merge de la PR du bas dans `main` | La branche qui servait de base disparaît en tant que branche active |
| T+1s | `automatic_base_change_succeeded` | GitHub détecte que la base de la PR du haut vient d'être absorbée dans `main`, et repointe son `base_ref` vers `main` |
| T+2s | `head_ref_force_pushed` (nouveau SHA) | Rebase serveur : les commits de la PR du haut sont rejoués sur le nouveau `main`, avec de nouveaux hash — pas un simple changement d'affichage |

**Point critique** : si ce rebase automatique rencontre un chevauchement textuel réel (même zone modifiée dans la pile et en dehors), il échoue silencieusement et la PR bascule en `CONFLICTING`. GitHub n'arbitre jamais qui a raison — c'est une fusion à trois points standard, et sans résolution automatique possible, la main revient à l'humain, exactement comme un `git rebase` interrompu.

### Résultats de l'expérimentation

Trois configurations testées, mêmes contenus, ordres de merge différents :

- **Aucun stacking** — branche du haut créée depuis `main` plutôt que depuis la branche du bas, malgré une dépendance logique entre les deux. Après merge du bas : conflit (`CONFLICTING`), résolution manuelle obligatoire (`git merge` + résolution + push).
- **Stacking isolé** — branche du haut réellement créée depuis la branche du bas. Après merge : `automatic_base_change_succeeded` + rebase serveur automatique, zéro intervention. État final : `mergeable: MERGEABLE`, `mergeStateStatus: CLEAN`.
- **Stacking + modification concurrente hors pile** — une troisième branche, indépendante, modifie la même zone et merge avant le reste de la pile. Résultat : `automatic_base_change_succeeded` se déclenche bien, mais le rebase automatique échoue — conflit réel, résolution manuelle requise.

| | Sans stacking | Stacking isolé | Stacking + concurrence externe |
|---|---|---|---|
| Historique partagé dès le départ | Non | Oui | Oui |
| Action déclenchée au merge du bas | Aucune | `automatic_base_change_succeeded` + rebase auto | `automatic_base_change_succeeded` puis échec |
| Résolution manuelle | Oui | Non | Oui |
| Origine du conflit | Historiques jamais réconciliés | — | Collision avec un changement externe à la pile |

### Limites et bonnes pratiques

- Le stacking automatise uniquement la synchronisation **interne à la pile**. Il n'offre aucune protection contre une collision venant de l'extérieur — la troisième expérimentation le démontre.
- Chaque rebase automatique génère de nouveaux SHA : tout lien externe pointant vers un ancien commit (cache CI, référence partagée) devient obsolète après un `automatic_base_change_succeeded`.
- Sur une pile longue, une modification instable en bas de pile propage un rebase en cascade sur tout ce qui est au-dessus.
- Pertinent pour un travail qui se découpe naturellement en étapes séquentielles (migration de schéma → code consommateur → nettoyage) et pour des équipes pratiquant le trunk-based development.
