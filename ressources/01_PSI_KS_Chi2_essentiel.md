# Détecter la dérive — PSI · KS · Chi² — Mini-cours

> Brief associé : M6-B1
> Durée de lecture : ~30 min
> Pré-requis : pandas, notion de distribution, p-value

## Pourquoi cette techno ?

Un modèle est entraîné sur une photo des données à un instant T. En production,
les données entrantes **évoluent** : nouveaux profils clients, contexte
économique, bug d'ingestion. Si la distribution des entrées s'éloigne trop de
celle d'entraînement, le modèle se dégrade. Encore faut-il le **mesurer
objectivement**, feature par feature — pas « à l'œil ».

Trois méthodes complémentaires : **PSI** (indice robuste avec seuils
d'interprétation simples), **KS** (test statistique sur numériques, donne une
p-value), **Chi²** (sur les catégorielles). On les **croise** : aucune n'est
parfaite seule. Alternative « tout-en-un » : Evidently AI (bonus) — mais
comprendre le calcul d'abord.

## Concepts clés

- **PSI (Population Stability Index)** : compare la répartition par bins (déciles
  de la référence) entre référence et prod. `PSI = Σ (p_cur - p_ref)·ln(p_cur/p_ref)`.
  Seuils standards : **< 0.1 stable, 0.1–0.25 suspect, > 0.25 dérive**.
- **KS (Kolmogorov-Smirnov)** : test sur 2 échantillons numériques. p < 0.05 ⇒
  distributions différentes. **Très sensible** sur gros échantillons (peut
  flagger des écarts minimes — regarder aussi l'ampleur).
- **Chi²** : sur les **catégorielles**, compare les fréquences de modalités via
  une table de contingence. p < 0.05 ⇒ répartition différente.
- **Croisement** : PSI et KS peuvent **diverger** (KS plus sensible). On
  conclut sur l'ensemble des signaux, pas une seule métrique.
- **Lissage anti-zéro** : un bin vide donne `ln(0)` / division par 0 → ajouter
  un ε (1e-6) aux proportions.

## Exemple minimal qui tourne

```python
# versions : pandas 2.2, scipy 1.13
import numpy as np
from scipy.stats import ks_2samp

def psi(ref, cur, n_bins=10):
    edges = np.unique(np.quantile(ref, np.linspace(0, 1, n_bins + 1)))
    edges[0], edges[-1] = -np.inf, np.inf
    p_ref = np.histogram(ref, edges)[0] / len(ref) + 1e-6
    p_cur = np.histogram(cur, edges)[0] / len(cur) + 1e-6
    return float(np.sum((p_cur - p_ref) * np.log(p_cur / p_ref)))

import pandas as pd
ref = pd.Series(np.random.normal(0, 1, 5000))
cur = pd.Series(np.random.normal(0.8, 1, 5000))   # décalé
print("PSI :", round(psi(ref, cur), 3))            # > 0.25 → dérive
print("KS p:", ks_2samp(ref, cur).pvalue)          # ~0 → différent
```

## Exercice guidé

Sur `reference_set.csv` vs `prod_3months.csv` :
1. Calculez le PSI **et** la p-value KS pour `int_rate` et `loan_amnt`.
2. Calculez le Chi² pour `grade`.
3. Que concluez-vous ? (Indice attendu : `int_rate` dérive fortement, `grade`
   aussi ; `loan_amnt` est stable.)

## Pièges fréquents

| Piège | Conséquence |
|---|---|
| PSI sans lissage anti-zéro | `inf` / `NaN` dès qu'un bin est vide |
| Conclure « dérive » sur la seule p-value KS | Faux positifs (KS ultra-sensible sur gros n) |
| Bins recalculés sur la prod | PSI non comparable (bins doivent venir de la référence) |
| Chi² sans aligner les modalités | Erreur de dimension de la table |
| Regarder une feature à la fois sans synthèse | On rate la vue d'ensemble |

| Symptôme | Cause probable |
|---|---|
| PSI = inf | bin vide, pas de ε |
| « tout dérive » au KS | gros échantillon, sensibilité excessive — croiser avec PSI |
| Chi² lève une erreur | modalités non alignées entre ref et prod |
| PSI faible mais KS significatif | normal : sensibilités différentes, c'est un signal à discuter |

## Pour aller plus loin

- scipy KS : https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.ks_2samp.html
- scipy Chi² : https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.chi2_contingency.html
- PSI (overview) : https://en.wikipedia.org/wiki/Population_stability_index

## Vérification (checklist apprenant)

- [ ] Je calcule PSI + KS sur les numériques, Chi² sur les catégorielles.
- [ ] Mon PSI gère le lissage anti-zéro (pas de `inf`).
- [ ] Les bins du PSI viennent de la **référence**.
- [ ] Je croise les méthodes au lieu de conclure sur une seule.
- [ ] J'ai un tableau de synthèse avec un verdict par feature.
