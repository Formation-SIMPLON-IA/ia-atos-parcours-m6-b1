# Étendre un dashboard Grafana existant — Mini-cours

> Brief associé : M6-B1
> Durée de lecture : ~20 min
> Pré-requis : Grafana provisionné en M5 (mini-cours M5 `04`)

## Pourquoi cette techno ?

Vous avez déjà un dashboard de prod en M5. En M6, on ne crée **pas** un nouveau
dashboard from scratch : on **étend l'existant** avec des panels de suivi de
dérive. C'est le réflexe pro — capitaliser sur l'outillage en place plutôt que
multiplier les tableaux de bord orphelins. Le dashboard reste **provisionné**
(versionné en JSON, chargé au démarrage), pas bricolé à la main.

## Concepts clés

- **Provisioning** : le dashboard vit en JSON dans le repo (`grafana/dashboards/`)
  et est chargé automatiquement — un panel ajouté à la main dans l'UI est perdu
  au prochain `compose down` s'il n'est pas exporté.
- **Étendre = ajouter des panels** au JSON (ou un nouveau JSON provisionné par le
  même provider), pas refaire l'UI.
- **Métriques offline vs live** : le PSI se calcule en **batch** (notebook/script)
  ; pour l'afficher en continu, on **pousse** une gauge (`pyrenex_feature_psi`)
  vers Prometheus. Le live (distribution des probas) utilise les métriques déjà
  exposées par le service `model` (M5).
- **Seuils colorés** : configurez les `thresholds` du panel PSI (vert < 0.1,
  orange < 0.25, rouge ≥ 0.25) pour une lecture immédiate.
- **uid de datasource** : référencez la datasource Prometheus par son `uid`
  (comme en M5) pour que le provisioning soit reproductible.

## Exemple minimal qui tourne

```json
// panel ajouté au dashboard, lecture d'une gauge PSI poussée par un batch
{
  "title": "PSI par feature", "type": "timeseries",
  "datasource": { "type": "prometheus", "uid": "prometheus" },
  "fieldConfig": { "defaults": { "thresholds": { "steps": [
    { "color": "green", "value": null }, { "color": "orange", "value": 0.1 },
    { "color": "red", "value": 0.25 } ] } } },
  "targets": [ { "expr": "pyrenex_feature_psi", "legendFormat": "{{feature}}" } ]
}
```

## Exercice guidé

À partir du dashboard M5 :
1. Ajoutez **3 panels** : PSI par feature, F1 macro sur 12 semaines, distribution
   des probabilités (quantiles de `pyrenex_prediction_proba` — métrique M5).
2. Versionnez le JSON dans `grafana/dashboards/pyrenex_drift.json`.
3. `docker compose up` doit le charger **sans import manuel**.

## Pièges fréquents

| Piège | Conséquence |
|---|---|
| Créer un nouveau dashboard from scratch | Hors-sujet : on **étend** l'existant |
| Bricoler dans l'UI sans exporter le JSON | Perdu au redémarrage |
| Afficher le PSI sans le pousser à Prometheus | Panel « No data » (métrique offline) |
| `uid` de datasource non référencé | « Datasource not found » au provisioning |

| Symptôme | Cause probable |
|---|---|
| Panels disparaissent au restart | JSON non versionné/provisionné |
| PSI « No data » | gauge non poussée vers Prometheus |
| « Datasource not found » | mauvais `uid` dans le panel |

## Pour aller plus loin

- Grafana — Provisioning : https://grafana.com/docs/grafana/latest/administration/provisioning/
- Prometheus — histogram_quantile : https://prometheus.io/docs/practices/histograms/

## Vérification (checklist apprenant)

- [ ] J'ai **étendu** le dashboard M5 (pas créé un nouveau).
- [ ] Mes 3 panels sont versionnés en JSON et provisionnés.
- [ ] Les seuils PSI sont colorés (vert/orange/rouge).
- [ ] `docker compose up` charge le dashboard sans clic.
- [ ] Je distingue métrique offline (PSI poussé) et live (proba M5).
