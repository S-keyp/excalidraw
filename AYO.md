# TD : mesurer DORA sur un dépôt réel

**Durée : 4 h 25. En binômes. Support de référence : [`../Readme.md`](../Readme.md)**

---

## Objet du TD

Vous allez calculer les métriques DORA d'un projet open source, [excalidraw/excalidraw](https://github.com/excalidraw/excalidraw), à partir de ses données publiques, puis instrumenter votre propre fork pour produire les données manquantes.

Ce projet a été retenu pour trois raisons : il déploie en production, il enregistre ses déploiements dans l'API Deployments de GitHub, et il distingue des environnements `Production` et `Preview`. Vous rencontrerez donc, sur des données réelles, la première erreur du tableau 7.3 du support : compter comme déploiement ce qui n'en est pas un.

À la fin, vous saurez répondre à trois interrogations :

- Que puis-je mesurer avec les données dont je dispose ?
- Qu'est-ce qui manque, et pourquoi un outil ne peut pas le déduire ?
- Que faut-il produire pour combler le manque ?

L'objectif n'est pas d'obtenir cinq chiffres, mais de comprendre pourquoi vous n'en obtiendrez d'abord que deux, et ce que coûte l'obtention des cinq.

## Organisation

Deux documents, deux usages :

| Document | Contenu |
|---|---|
| Ce fichier | le sujet : ce que vous cherchez, et les questions auxquelles répondre |
| [`Guide.md`](Guide.md) | le mode opératoire : les commandes à taper, et ce que vous devez voir |

Gardez les deux ouverts. Le guide liste les erreurs courantes et leur résolution.

Les phases du sujet et les étapes du guide se correspondent ainsi :

| Phase du sujet | Étape du guide |
|---|---|
| Phase 0 : le contrat | aucune étape, phase sans outil |
| Phase 1 : reconnaissance | étapes 0 à 2 |
| Phase 2 : collecte outillée | étape 3 |
| Phase 3 : le proxy | étape 4 |
| Phase 4 : produire la donnée | étape 5 |
| Phase 5 : lecture critique | étape 6 |

Les durées annoncées dans le sujet couvrent la manipulation et la rédaction des réponses. Celles du guide ne couvrent que la manipulation : elles sont donc plus courtes.

**Rendu attendu :** un fichier `reponses.md` dans votre fork, contenant vos réponses numérotées et votre `dora-definitions.yml` rempli.

---

## Prérequis

| Outil | Vérification |
|---|---|
| `git` 2.30 ou plus | `git --version` |
| `python` 3.9 ou plus | `python --version`, ou `py --version` sous Windows |
| Un compte GitHub | pour le fork et le jeton d'accès |

Le collecteur n'utilise que la bibliothèque standard : il n'y a rien à installer, ni `pip`, ni environnement virtuel.

La création du jeton d'accès est la première étape du guide pas à pas. Faites-la avant le reste : sans jeton, la limite de l'API est atteinte au bout de trois exécutions.

---

## Phase 0 : le contrat, avant l'outil (20 min)

Aucun outil pendant cette phase. Discussion et éditeur de texte.

Le chapitre 5.2 du support recommande d'écrire le contrat de définitions avant la première ligne de code. Les décisions prises ici déterminent les chiffres obtenus en phase 2.

Ouvrez [`outils/dora-definitions.yml`](outils/dora-definitions.yml) et remplissez-le pour excalidraw, sans regarder les données. En binôme, décidez et écrivez :

- qu'est-ce qui compte comme un déploiement, et qu'est-ce qui est exclu ?
- quel horodatage retenez-vous : la création du déploiement, ou son passage au statut `success` ?
- où démarre un changement : la committer date, ou l'ouverture de la pull request ?
- qu'est-ce qu'un incident, pour un projet dont vous n'exploitez pas la production ?
- quelle fenêtre, quelle agrégation ?

**Rendu de phase :** votre `dora-definitions.yml` rempli, et l'heure à laquelle vous l'avez figé. Vous n'y toucherez plus avant la phase 3.

Les deux dernières questions n'ont pas de réponse unique. Écrivez la vôtre et documentez-la : c'est la situation d'une équipe qui démarre.

---

## Phase 1 : reconnaissance (40 min)

Vous forkez le dépôt, le clonez, et explorez l'API à la main avant d'automatiser.

**Questions :**

1. Combien de valeurs différentes d'`environment` trouvez-vous dans les déploiements ? Listez-les.

Sur 100 déploiements: 
	91: “Preview - excalidraw”
	1: “Production - excalidraw”
	3: “Production – excalidraw-package-example-with-nextjs”
	3: "Production – excalidraw-package-example"
	2: “Production – docs"

2. Lesquelles correspondent à une mise en production au sens de DORA ? Lesquelles doivent être exclues, et pourquoi ?

Une seule mise en production effective, les autres ne concerne pas le produit lui même.

3. Si vous comptiez tous ces déploiements, de quel facteur votre deployment frequency serait-elle surestimée ?

Si nous comptions tous les déploiements le DF seraient surévalué d’un facteur de 9

4. Retrouvez la ligne correspondante dans le tableau des erreurs d'implémentation (7.3 du support).

5. Que contient le tableau des statuts d'un déploiement ? Pourquoi l'existence d'un déploiement ne suffit-elle pas à établir qu'un changement est arrivé en production ?

Le tableau des status contient tel que l’état, le créateur, la description, l’environnement, l’id, le node, le created_ad, et l’url de déploiement. La simple existence d’un déploiement ne suffit pas à établir qu’un changement est arrivé en production car le déploiement peut ne pas fonctionner pour de nombreuses raisons (Server indisponibles, test non passant etc)

6. Quel horodatage faut-il retenir : le `created_at` du déploiement, ou celui du statut `success` ? Votre contrat de phase 0 avait-il tranché ?

Il faut retenir le created_at du déploiement et pas le success car DORA mesure l’intégralité du temps de déploiement et pas seulement la fin.

---

## Pause (15 min)

---

## Phase 2 : collecte outillée (50 min)

Vous lancez le collecteur et obtenez vos premiers chiffres.

**Reportez vos résultats :**

| Métrique | Valeur obtenue |
|---|---|
| Deployment frequency | 0.167 /jour |
| Délai médian entre deux déploiements | 4.5 j |
| Change lead time P50 | 43.8 h |
| Change lead time P90 | 8.9 j |
| Commits analysés / nombre de lots | 79 commits / 14 lots |

**Questions :**

7. Rapportez le nombre de commits au nombre de lots. Que vaut la taille moyenne d'un lot ? Que dit le chapitre 6.1 du support de ce chiffre ?

79 / 14 ≈ 5,6 commits/lot. Un lot qui regroupe plusieurs commits mélange plusieurs changements dans un seul déploiement, cela augmente le risque associé à chaque déploiement car plus de choses peuvent casser en même temps.

8. Comparez P50 et P90. Quel est le rapport entre les deux ? D'après la section "médiane et percentiles" de l'annexe B, que signale un tel écart, et que ne signale-t-il pas ?

P50 = 43,8 h (environ 1,8 j), P90 = 8,9 j (environ 213 h) soit environ un rapport de ×4,9.
Indique une forte dispersion, certains déploiements traînent beaucoup plus longtemps que le cas "typique". Il ne dit pas pourquoi ni combien de fois ce cas extrême se produit

9. La deployment frequency vous place dans quel ordre de grandeur au regard de la distribution 2024 (4.2) ? Tenez compte du chapitre 4.1 avant de conclure.

0,167/jour ≈ 1 déploiement tous les 6 jours, soit un rythme hebdomadaire — plutôt dans la tranche "High", proche de "l'elite". Mais avec seulement 15 déploiements observés l'échantillon est petit et donc pas nécessairement représentatif

10. Le collecteur affiche aussi le délai médian entre deux déploiements. Pourquoi cette formulation est-elle préférable à la fréquence brute pour une équipe qui déploie peu (2.3) ?

Une fréquence moyenne peut induire en errreur. Le délai médian entre deux déploiements est moins sensible aux irrégularités ou aux regroupements ponctuels.

11. Trois métriques sur cinq s'affichent `n/a`. Lesquelles ? Qu'ont-elles en commun ?

Failed deployment recovery time, Change fail rate, Deployment rework rate. Ils nécessitent de savoir si un déploiement a échoué ou provoqué un incident, on ne peut pas le savoir sur base seule des commits et horodatages de déploiement.

12. Le support désigne un maillon faible (5.1). Lequel, et pourquoi ne peut-il pas être reconstitué à partir des données publiques du dépôt ?

Le lien entre déploiement et échec/incident en production. On ne peut pas le reconstituer à partir des données car rien n'indique qu'un déploiement a été rollback, a causé un incident, ou a nécessité un correctif

13. Un collègue propose la règle suivante : un déploiement suivi d'un autre moins de 24 h après est un échec. Donnez deux situations où cette règle se trompe, une dans chaque sens.

Faux positif : deux déploiements planifiés rapprochés -> cadence normale d'une équipe qui déploie souvent, pas un échec.
Faux négatif : un déploiement défaillant corrigé seulement après plusieurs jours à cause d'une détection tardive

---

## Phase 3 : le proxy et ses limites (40 min)

Les incidents étant absents, vous les approximez par le label `bug` du dépôt, puis vous vérifiez le résultat dans l'interface GitHub.

**Questions :**

14. Combien d'issues `bug` le collecteur trouve-t-il sur la fenêtre ? Combien sont rattachées à un déploiement ?

Total de 23 issues "bug" dont 0 sont rattachés à un déploiement

15. Dans l'interface GitHub : combien d'issues toutes catégories ont été ouvertes sur ces 90 jours ? Combien portent le label `bug` depuis la création du dépôt ?

124 issues dans les 3 derniers mois dont 0 ayant le label "bug" et 554 issues (ouvertes et fermées) avec un label "bug" depuis la création du dépôt

16. Confrontez les trois nombres. Que s'est-il passé dans ce projet ?

Un grand nombre de bug ont étés découvert mais il semble qu'aucun rapprochement entre bug et déploiement n'ait été fait. Plus, il semblerai que les utilisateurs aient arrétés de rapporter les bugs.

17. Le chapitre 7.3 énumère trois explications à un taux d'échec de 0 %. Aucune ne décrit ce cas : formulez la quatrième.

Il semblerait qu'il y ait une absence de liens entre bug et déploiement, nous ne pouvons donc pas comptabiliser les échec.

18. Quelle métrique le support décrit-il comme la plus sensible à la discipline de saisie (2.8) ? Ce rapprochement vous paraît-il fortuit ?

C'est le change fail rate qui est extrèmement sensible à l'observabilité, ce qui ne semble pas être un hasard car c'est le problème que nous rencontrons ici et qui fait que nous avons un taux 
d'échec de 0% 

Un outil rend toujours un chiffre. Ce chiffre peut être le symptôme d'un marqueur qui a cessé d'être appliqué, et non la mesure d'une réalité.

---

## Phase 4 : produire la donnée manquante (50 min)

Cette phase se déroule sur votre fork. Vous ajoutez un workflow qui émet des évènements de déploiement, produisez cinq déploiements dont un correctif d'urgence, simulez un incident et le rattachez à son déploiement, puis relancez le collecteur.

**Questions :**

19. Que pouvez-vous calculer maintenant que vous ne pouviez pas calculer avant ?
20. Combien de temps a demandé la production de cette donnée, comparé au temps passé à tenter de la déduire en phase 3 ?
21. Votre change fail rate est-il représentatif ? Que faudrait-il pour qu'il le devienne ?

---

## Phase 5 : lecture critique des outils (25 min)

Le chapitre 5.5 du support recense les outils recommandés. Vous n'avez pas à les installer.

| Outil | État vérifié au 6 septembre 2026 |
|---|---|
| Apache DevLake | actif, dernière version `v1.0.3-beta16` du 27/08/2026 |
| Middleware | dépôt actif, dernière version `0.3.1` de mai 2025 |
| Four Keys (Google) | archivé depuis janvier 2024, en lecture seule |

**Questions :**

22. Ces outils calculeraient-ils le change fail rate d'excalidraw ? Sur quelle donnée s'appuieraient-ils ? Votre conclusion de la phase 2 change-t-elle parce que l'outil est professionnel plutôt qu'un script de 200 lignes ?
23. Le chapitre 5.5 propose l'ordre suivant : Quick Check, puis conversation d'équipe, puis instrumentation. Au vu de votre demi-journée, pourquoi l'outillage arrive-t-il en dernier ?
24. Four Keys était la référence citée dans la plupart des tutoriels jusqu'en 2024. Quelle habitude de travail cela suggère-t-il avant d'adopter un outil trouvé en ligne ?

---

## Restitution collective (25 min)

Chaque binôme présente en 3 minutes :

- ses chiffres de la phase 2, et l'écart avec ceux du binôme voisin ;
- la ligne de son `dora-definitions.yml` qui explique cet écart ;
- ce qu'il aurait écrit différemment en phase 0.

**Questions de la discussion finale :**

- Deux binômes ont mesuré le même dépôt, la même semaine, avec le même outil. Pourquoi leurs chiffres diffèrent-ils ? Que dit le chapitre 5.2 de deux équipes qui n'ont pas le même contrat ?
- Excalidraw affiche un lead time P90 plusieurs fois supérieur à sa médiane. Si vous étiez dans l'équipe, quelle serait votre première action, et quelle question poseriez-vous avant d'agir ?
- Vous disposez des chiffres de livraison d'un projet open source. Que pouvez-vous en conclure sur la performance de l'équipe qui le maintient ? Justifiez avec le chapitre 7.2.

---

## Auto-évaluation

| Vous savez | Chapitre |
|---|---|
| distinguer un déploiement de production d'un déploiement de préproduction, sur données réelles | 2.3, 7.3 |
| dire ce que mesurent une médiane et un P90, et pourquoi les deux sont nécessaires | 2.2, annexe B |
| nommer le maillon faible d'une instrumentation et expliquer pourquoi un outil ne le comble pas | 5.1 |
| reconnaître un proxy et énoncer ce qu'il ne mesure pas | 2.5, 2.6 |
| lire un taux de 0 % comme un signal d'alerte plutôt que comme un résultat | 7.3 |
| émettre un évènement de déploiement depuis un pipeline | 5.3 |
| refuser de classer deux équipes, et dire pourquoi | 7.1, 7.2 |

---

## Pour aller plus loin

- Refaites la phase 2 avec `--window 28`, puis `--window 365`. Vos conclusions changent-elles ? Quelle fenêtre retiendriez-vous, et pourquoi pas 7 jours ?
- Comparez excalidraw à un projet qui déploie plus souvent. `PostHog/posthog` expose des environnements `prod-us` et `prod-eu`, ce qui pose une question absente du support : deux productions, une ou deux séries de métriques ?
- Faites le [DORA Quick Check](https://dora.dev/quickcheck/) sur votre propre équipe. Comparez le ressenti déclaré aux chiffres mesurés.
