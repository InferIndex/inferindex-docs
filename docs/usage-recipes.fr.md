# Exemples

Chaque commande est un simple `curl` sur l'API publique, sans clé. Les réponses sont abrégées et datées ; les prix changent.

## 1. Combien coûte un modèle en ce moment, et qui le vend ?

```bash
curl "https://api.inferindex.dev/cheapest?model=meta-llama/llama-3.3-70b-instruct&limit=3"
```

_Réponse au 16/10/2026, abrégée._

```json
{
  "model": { "id": "meta-llama/llama-3.3-70b-instruct", "name": "Meta: Llama 3.3 70B Instruct" },
  "offers_total": 41,
  "cheapest": {
    "provider": "NanoGPT",
    "via": "nanogpt",
    "via_name": "NanoGPT",
    "input_per_1M": 0.05,
    "output_per_1M": 0.23,
    "blended_per_1M": 0.095,
    "promo": false,
    "checked_at": "2026-10-16T07:40:37.737Z"
  }
}
```

À lire : `blended_per_1M` est le prix pour un mélange input:output de 3:1, `via_name` le vendeur quand l'offre passe par un agrégateur. `promo` signale une baisse ou une promotion (`false` ici ; `probable` : baisse vue par nous, non annoncée par le vendeur) et `checked_at` date la dernière confirmation du prix.

## 2. Combien coûterait ma charge par mois ?

```bash
curl "https://api.inferindex.dev/cheapest?model=openai/gpt-oss-120b&prompt_tokens=2000&output_tokens=500&requests_per_day=1000&limit=3"
```

_Réponse au 16/10/2026, abrégée._

```json
{
  "model": { "id": "openai/gpt-oss-120b", "name": "OpenAI: gpt-oss-120b" },
  "usage": { "prompt_tokens": 2000, "output_tokens": 500, "cached_ratio": 0, "requests_per_day": 1000 },
  "cheapest": {
    "provider": "Runware",
    "via_name": "LLM Gateway",
    "input_per_1M": 0.032,
    "output_per_1M": 0.14,
    "estimated_cost_per_request": 0.000134,
    "estimated_cost_per_month": 4.02,
    "estimate_approx": false,
    "estimate_notes": ["context_unknown", "max_output_unknown"]
  }
}
```

À lire : `estimated_cost_per_month` couvre 30 jours à la charge indiquée, pour l'offre la moins chère à cette charge ; `estimate_approx` vaut `true` quand la source ne publie pas tous ses paliers ou son prix de cache ; `estimate_notes` liste des réserves comme `context_unknown`.

## 3. Lequel de deux modèles coûte le moins pour la même charge ?

Aucune route ne compare deux modèles. Appelez `/cheapest` une fois par modèle avec la même charge et comparez `estimated_cost_per_month` :

```bash
curl "https://api.inferindex.dev/cheapest?model=meta-llama/llama-3.3-70b-instruct&prompt_tokens=2000&output_tokens=500&requests_per_day=1000&limit=1"
curl "https://api.inferindex.dev/cheapest?model=openai/gpt-oss-120b&prompt_tokens=2000&output_tokens=500&requests_per_day=1000&limit=1"
```

_Réponses au 16/10/2026, abrégées._

```json
{ "model": { "id": "meta-llama/llama-3.3-70b-instruct" }, "cheapest": { "provider": "NanoGPT", "via_name": "NanoGPT", "estimated_cost_per_month": 6.45 } }
{ "model": { "id": "openai/gpt-oss-120b" },    "cheapest": { "provider": "Runware",  "via_name": "LLM Gateway", "estimated_cost_per_month": 4.02 } }
```

À lire : les deux chiffres se comparent, la charge étant la même ; la page de comparaison du site les montre côte à côte. Aucune route ne liste non plus toutes les offres d'un fournisseur : sa fiche sur le site le fait.

## 4. Le prix a-t-il changé ?

```bash
curl "https://api.inferindex.dev/history?model=deepseek/deepseek-v3.2&days=7&granularity=day&provider=DeepInfra"
```

_Réponse au 16/10/2026, deux premiers points sur seize._

```json
{
  "model": { "id": "deepseek/deepseek-v3.2" },
  "days": 7,
  "granularity": "day",
  "history": [
    { "period": "2026-10-09", "provider": "DeepInfra", "quantization": "fp4", "variant": "", "input_min": 0.26, "input_max": 0.26, "input_last": 0.26, "output_min": 0.38, "output_max": 0.38, "output_last": 0.38 },
    { "period": "2026-10-09", "provider": "DeepInfra", "quantization": "fp4", "variant": "flex", "input_min": 0.208, "input_max": 0.208, "input_last": 0.208, "output_min": 0.304, "output_max": 0.304, "output_last": 0.304 }
  ]
}
```

À lire : un point par jour et par offre ; `min`, `max` et `last` égaux : le prix n'a pas bougé ce jour-là. `variant: "flex"` est un niveau de priorité basse, pas le prix standard.

## 5. Qui le propose dans l'UE, sans entraînement sur mes prompts ?

```bash
curl "https://api.inferindex.dev/resellers?model=meta-llama/llama-3.3-70b-instruct&region=eu&no_training=true&strict=true&limit=3"
```

_Réponse au 16/10/2026, abrégée._

```json
{
  "model": { "id": "meta-llama/llama-3.3-70b-instruct" },
  "total_offers": 4,
  "filters_unknown": { "region": 53, "no_training": 26, "quantization": 9 },
  "excluded": { "region_unknown_strict": 54, "region_mismatch": 8, "training_unknown_strict": 26, "training_not_excluded": 6 },
  "offers": [
    { "provider": "CloudFerro", "blended_per_1M": 0.4763, "conditions": { "provider": { "regions": { "value": "EU (Poland)" }, "training_on_prompts": { "value": "no" } } } },
    { "provider": "IONOS",      "blended_per_1M": 0.7285, "conditions": { "provider": { "regions": { "value": "Germany (EU)" }, "training_on_prompts": { "value": "no" } } } }
  ]
}
```

À lire : sans `strict=true`, les offres dont la région ou la politique n'est pas publiée restent, comptées dans `filters_unknown` ; « non publié » n'est pas « non ». Avec `strict=true` elles sortent, d'où la liste courte ; `excluded` dit combien et pourquoi.

## 6. Louer un GPU ou payer l'API ?

```bash
curl "https://api.inferindex.dev/self-host?model=qwen/qwen3-32b&tokens_per_day=50000000"
```

_Réponse au 16/10/2026, abrégée._

```json
{
  "status": "ok",
  "verdict": "api_cheaper",
  "headline": "At 50 million tokens a day, the API is cheaper whatever the throughput ($0.123/M at a 3:1 input:output mix; self-hosting breaks even at 68.6 million tokens a day).",
  "threshold_m_tokens_per_day": 68.57,
  "required_tokens_per_second": { "at_utilization": 1587.4, "utilization_percent": 50 },
  "api": { "provider": "Nscale", "usd_per_m_tokens": 0.1225, "offers_compared": 21 },
  "assumptions": ["Hardware rental only: the hourly price the provider publishes for the machine. It does not count the engineering to deploy and run the model, monitoring, …", "…"],
  "configuration": { "gpu_name": "RTX A6000 48GB", "gpus_needed": 1, "provider": "Thunder Compute", "usd_per_day": 8.4, "throughput_evidence": "none" }
}
```

À lire : `threshold_m_tokens_per_day` est le volume où une configuration louée coûte autant que l'API la moins chère ; au-dessus, louer ne gagne que si elle tient `required_tokens_per_second`, une condition, pas une promesse. Les chiffres ne couvrent que la location du matériel ; `assumptions` liste ce qui manque.

## 7. Où en est le marché, par famille de modèles ?

```bash
curl "https://api.inferindex.dev/index?series=market"
```

_Réponse pour la publication du 12/10/2026, abrégée._

```json
{
  "index": "InferIndex market index",
  "series": "market",
  "published_at": "2026-10-12T00:00:00.000Z",
  "method_version": "1.2",
  "unit": "USD per 1M tokens, blended 3:1 (input:output)",
  "compared_to": "2026-10-05T00:00:00.000Z",
  "families": [
    { "family": "open_small", "value": 0.125, "models": 54, "change_on_same_models_pct": 0, "same_models": 54 },
    { "family": "open_medium", "value": 0.2953, "models": 30, "change_on_same_models_pct": 2.16, "same_models": 30 }
  ],
  "corrections": []
}
```

À lire : la `value` d'une famille est le prix mixte médian des modèles comptés ; `change_on_same_models_pct` ne compare que les modèles comptés dans les deux publications : un modèle qui entre ou sort ne la fait pas bouger. `corrections` liste toute valeur corrigée après publication, avec le motif (vide ici).

## 8. Où se classe un modèle dans les classements publics ?

```bash
curl "https://api.inferindex.dev/rankings?model=deepseek/deepseek-v3.2"
```

_Réponse au 16/10/2026, abrégée._

```json
{
  "model": { "id": "deepseek/deepseek-v3.2" },
  "rankings": [
    { "source": "Arena", "bench": "text_style_control", "listed_as": "deepseek-v3.2", "rank": 123, "out_of": 415, "score": 1424.67, "votes": 48134, "published_on": "2026-10-13",
      "scale": { "score_name": "Arena Score", "count_name": "votes" } }
  ],
  "attribution": { "license": "CC BY 4.0", "license_url": "https://creativecommons.org/licenses/by/4.0/" }
}
```

À lire : le rang et le score sont ceux de la source, jamais calculés par nous ; `scale` nomme le score et dit ce qui est compté. Gardez l'objet `attribution` si vous réutilisez les données : elles sont sous licence CC BY 4.0.

## Avec un assistant (MCP)

Une fois le serveur connecté (`https://mcp.inferindex.dev/mcp`), demandez en mots simples ; l'assistant choisit l'outil.

- _« Quel est le moyen le moins cher d'appeler DeepSeek V3.2 maintenant ? »_ → `cheapest` : prix, vendeur, date de vérification, et un signal si le prix est une promotion.
- _« J'envoie environ 2 000 tokens en input et j'en reçois 500 en output, 1 000 fois par jour, sur gpt-oss-120b. Combien par mois ? »_ → `estimate_cost` : coût par requête et par mois, hypothèses rappelées.
- _« Auto-héberger Qwen3 32B, est-ce moins cher qu'une API ? »_ → `self_host_or_api` : le seuil de rentabilité et le débit nécessaire, en condition ; location du matériel seulement.
