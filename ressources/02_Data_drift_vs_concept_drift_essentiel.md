# Data drift vs concept drift — Mini-cours

> Brief associé : M6-B1
> Durée de lecture : ~25 min
> Pré-requis : détection de dérive (mini-cours 01), notion d'AUC

## Pourquoi cette techno ?

Détecter qu'« il y a dérive » ne suffit pas : la **remédiation dépend du type**.
Réentraîner coûte cher ; on ne le propose que si le diagnostic le justifie. La
distinction clé : les **entrées** ont-elles changé (*data drift*), ou la
**relation entrée → cible** a-t-elle changé (*concept drift*) ? Les deux se
soignent différemment.

C'est le cœur du raisonnement attendu par Sophie Léger : un diagnostic
**tranché et prouvé**, pas « on pense que… ».

## Concepts clés

- **Data drift** : la distribution des features change, mais la **logique de
  risque tient**. Ex. les taux montent, la clientèle glisse vers des grades plus
  risqués — mais « taux élevé ⇒ plus de risque » reste vrai.
- **Concept drift** : les features peuvent être stables, mais la **relation
  features → cible** a changé (nouveau comportement, choc réglementaire).
- **Le révélateur : l'AUC.** L'AUC mesure le **pouvoir de tri** indépendamment
  du seuil. **AUC stable ⇒ le modèle ordonne toujours bien ⇒ relation préservée
  ⇒ data drift.** AUC qui chute ⇒ la relation se casse ⇒ concept drift.
- **Triangulation** : croiser (1) dérive des features (PSI/KS/Chi²), (2)
  stabilité de l'AUC, (3) calibration, (4) temporalité (tendance vs saut).
- **Conséquence** : data drift ⇒ réentraîner sur données récentes (recale la
  calibration). Concept drift ⇒ réentraîner **en urgence** + investiguer la cause.

## Exemple minimal qui tourne

```python
from sklearn.metrics import roc_auc_score
# proba & y sur deux périodes
auc_debut = roc_auc_score(y_debut, proba_debut)
auc_fin   = roc_auc_score(y_fin,   proba_fin)
if abs(auc_debut - auc_fin) < 0.03:
    print("AUC stable → relation préservée → piste data drift")
else:
    print("AUC en baisse → piste concept drift")
```

## Exercice guidé

Avec `predictions_log.csv` (colonnes `proba_default`, `true_label`, `timestamp`) :
1. Calculez l'AUC sur les semaines 1-4 puis 9-12.
2. Comparez à la dérive des features (mini-cours 01).
3. Tranchez : data ou concept drift ? (Attendu : features dérivent + AUC stable
   → **data drift**.)

## Pièges fréquents

| Piège | Conséquence |
|---|---|
| Conclure concept drift parce que le F1 baisse | Faux : le F1 dépend du seuil, regarder l'AUC |
| Oublier la temporalité | On confond une tendance avec un saut (bug ETL) |
| Diagnostic non chiffré | Note rejetée par le client |
| Recommander un réentraînement sans type de drift | Remédiation non proportionnée |

| Symptôme | Cause probable |
|---|---|
| F1 ↓ mais AUC stable | data drift + calibration dégradée (pas concept drift) |
| AUC ↓ franchement | concept drift probable |
| Saut brutal d'une feature | suspecter un **bug ETL**, pas une dérive « naturelle » |

## Pour aller plus loin

- Evidently — types de drift : https://www.evidentlyai.com/ml-in-production/data-drift
- scikit-learn — ROC-AUC : https://scikit-learn.org/stable/modules/model_evaluation.html#roc-metrics

## Vérification (checklist apprenant)

- [ ] Je sais définir data drift vs concept drift.
- [ ] J'utilise l'**AUC** comme révélateur (stable ⇒ data drift).
- [ ] Je triangule (features + AUC + calibration + temporalité).
- [ ] Mon diagnostic est **tranché et chiffré**.
- [ ] Je relie le type de drift à la remédiation proposée.
