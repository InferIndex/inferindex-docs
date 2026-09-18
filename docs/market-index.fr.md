# Indice de marché InferIndex — méthode

*Version 1.0, adoptée le 18/09/2026, figée avant la première publication du lundi 05/10/2026.*
[English version](market-index.md)

L'indice de marché InferIndex suit, semaine après semaine, le prix d'un million de tokens de LLM pour chaque
famille de modèles. Cette page décrit précisément son calcul, pour que chaque valeur publiée puisse être comprise
et vérifiée.

## Ce qui est mesuré

- **Unité** : le prix d'un million de tokens, en dollars américains, **mixte** 3:1, soit trois tokens d'entrée pour
  un token de sortie : `(3 × entrée + sortie) / 4`.
- **Regroupement** : une valeur par **famille de modèles** (voir plus bas).
- **Fréquence** : publié chaque **lundi**, à la valeur de **00:00 UTC** ce jour-là. Première publication :
  **05/10/2026**.
- **Données** : uniquement les relevés de prix d'InferIndex, collectés depuis le début du suivi le **14/09/2026**.
  Aucune donnée antérieure, ni aucune donnée d'un autre indice ou jeu de données, n'y entre. Chaque valeur est
  reproductible à partir de nos relevés.

## Deux séries, jamais mélangées

1. **Série de marché** (`series=market`) : calculée à partir des offres que nous relevons chez les fournisseurs et
   revendeurs, c'est-à-dire ce que l'on paie réellement pour un modèle.
2. **Série officielle** (`series=official`) : calculée à partir des **prix de liste** que nous relevons sur les pages
   de tarifs des labs eux-mêmes.
   - Une remise annoncée n'est pas un prix de liste : c'est le prix avant remise qui compte.
   - Seul le niveau de service standard est retenu.
   - Quand un lab publie des prix pour plusieurs régions, on prend la moins chère.

Les deux séries démarrent le 14/09/2026, avec les mêmes familles et le même minimum de 3 modèles par famille. Elles répondent à deux
questions différentes, « que fait payer le marché ? » et « quels prix affichent les labs ? », et ne sont jamais
combinées.

## Prix de référence d'un modèle (série de marché)

On part des offres **en vigueur à l'instant du calcul** (même règle que `/history?at=`), puis on **exclut** :

- les offres en promotion, annoncée ou probable, y compris une promotion terminée ;
- les prix convertis depuis des points ou des crédits ;
- le prix unique d'une passerelle pour plusieurs fournisseurs non nommés ;
- les offres périmées à cet instant (non revérifiées depuis plus de trois fois la fréquence de collecte de leur
  source, avec un minimum de six heures) ;
- les offres signalées comme revendues sous le prix de liste du lab (`below_official_list`) ;
- les niveaux de service non standard (flex, batch, priority) ;
- les identifiants de modèle que le lab a redirigés vers un autre modèle ;
- les revendeurs non officiels d'accès aux modèles fermés, qu'InferIndex ne référence jamais ;
- les modèles récemment ajoutés, encore en cours de vérification au moment du calcul ;
- les prix dont la devise n'a pas de taux de change.

Pour un modèle fermé, tous les revendeurs comptent, pas seulement le lab.

On garde ensuite **un seul prix par fournisseur** (son plus bas), et le prix de référence du modèle est la **médiane
des trois fournisseurs les moins chers**. **Un modèle qui a moins de 3 fournisseurs n'a pas de prix de référence.**

Les prix en devise étrangère sont convertis en dollars au dernier taux de référence de la BCE publié avant l'instant
mesuré (daté au plus tard la veille ; pour une publication du lundi 00:00 UTC, le taux du vendredi précédent).

Dans la série officielle, la référence d'un modèle est son prix de liste chez le lab (voir *Deux séries*) : hors
remise, niveau standard, région la moins chère. Les exclusions de la série de marché et le minimum de 3 fournisseurs
ne s'appliquent pas. Les modèles en cours d'examen sont exclus dans les deux séries.

## Familles

Chaque modèle est placé dans une famille par une règle écrite, appliquée dans cet ordre :

1. **Usage spécial** (modération, embeddings, reranking, voix, OCR) : hors indice.
2. **Code** (`code`) : le nom désigne un modèle de code (coder, codestral, devstral, codex, « -code »), qu'il soit
   ouvert ou fermé.
3. **Modèles fermés**, sans poids publiés (par exemple Claude, GPT, Gemini, Grok, Qwen Max et Plus) :
   - **Fermés compacts** (`compact_closed`) : le nom porte une gamme compacte du lab (mini, nano, flash, lite,
     haiku, luna, micro) ;
   - **Fermés frontier** (`frontier_closed`) : tous les autres modèles fermés.
4. **Modèles à poids ouverts**, selon le nombre total de paramètres affiché sur la page Hugging Face du modèle (relevé,
   daté et sourcé par nous) :
   - **Grands** (`open_large`) : 200 milliards de paramètres ou plus ;
   - **Moyens** (`open_medium`) : de 40 à 200 milliards ;
   - **Petits** (`open_small`) : moins de 40 milliards.
5. **Sinon, non classé** : hors indice et listé comme tel, sans jamais deviner sa famille. C'est le cas, par
   exemple, d'un modèle ouvert dont la taille n'est pas encore relevée, ou d'un modèle dont le statut des poids
   n'est pas établi.

Le raisonnement n'est pas une famille : la plupart des modèles récents sont hybrides, et une famille à part ne serait
ni stable ni exclusive.

Le nombre de paramètres des nouveaux modèles ouverts est relevé à chaque calcul du lundi.

## Valeur d'une famille

La valeur d'une famille est la **médiane, non pondérée, des prix de référence de ses modèles**. **Une famille qui a
moins de 3 modèles n'a pas de valeur cette semaine-là** (`value: null`).

**Pourquoi pas de pondération** : nous n'observons pas les volumes réellement consommés, toute pondération serait donc
une hypothèse. La médiane reste robuste aux cas extrêmes et facile à vérifier. Chaque publication donne la liste des
modèles comptés dans chaque famille : n'importe qui peut refaire le calcul.

## Limites

- **Ce n'est pas un indice à panier constant.** La composition d'une famille change quand un modèle arrive,
  disparaît ou n'a plus assez de fournisseurs. Chaque publication donne le nombre de modèles comptés, et une
  variation d'une semaine sur l'autre peut venir d'un changement de composition autant que d'un changement de prix.
- L'indice reflète les prix de liste et d'offre tels que publiés, pas les remises négociées ou de volume.
- Les prix viennent d'informations publiques et peuvent contenir des erreurs ; voir les
  [conditions d'utilisation](https://api.inferindex.dev/terms).

## Versions

La méthode est figée avant la première publication. Toute modification ultérieure reçoit un nouveau numéro de version,
avec sa date et sa raison, dans l'historique ci-dessous. Chaque publication indique la `method_version` utilisée.
**Les valeurs déjà publiées ne sont jamais recalculées en silence.**

**Corrections.** Une semaine publiée n'est jamais recalculée en silence. Si une erreur de données est corrigée après
publication, la semaine est recalculée par une correction explicite, avec un motif obligatoire : chaque famille dont la
valeur ou la composition change garde sa valeur précédente, et `GET /index` liste les corrections de cette date (valeur
précédente, nouvelle valeur, motif, date de la correction).

| Version | Date | Changement |
|---|---|---|
| 1.0 | 18/09/2026 | Première version |

## Accès

`GET https://api.inferindex.dev/index?series=market` (ou `official`), avec éventuellement `&at=AAAA-MM-JJ` pour un
lundi passé. Voir [`GET /index`](api.md#get-index) dans la référence de l'API (en anglais).
