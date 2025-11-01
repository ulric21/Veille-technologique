# Veille — Network Security

**But du repo**: centraliser les sources, outils, scripts et workflows pour une veille technologique opérationnelle en sécurité réseau (network security). Ce README est conçu pour être directement utilisé comme `README.md` dans ton dépôt GitHub.

---

## 🚀 Présentation

Ce dépôt regroupe:

* une liste organisée de sources (blogs, alertes, newsletters) à suivre;
* des outils et APIs recommandés (Shodan, VirusTotal, Censys, etc.);
* des scripts d'automatisation (ex : recherches Shodan, récupération RSS);
* des workflows GitHub Actions pour intégrer la veille au repo (mise à jour automatique de fichiers Markdown à partir de RSS, commits automatiques des résultats);
* une structure de dossier pour centraliser les données, rapports et IOC.

L'objectif : disposer d'un système reproductible et traçable de veille pour surveiller les actifs, vulnérabilités et incidents.

---

## 📁 Structure du dépôt (suggestion)

```
/ (root)
├─ README.md                 # ce fichier
├─ VEILLE.md                 # résumé quotidien / hebdo (généré automatiquement)
├─ scripts/
│  ├─ shodan_search.py       # script d'exemple Shodan -> JSON
│  ├─ vt_lookup.py           # script VirusTotal minimal
│  └─ rss_to_md.py           # convertit un RSS en Markdown
├─ .github/workflows/
│  ├─ rss-to-md.yml         # workflow: génère VEILLE.md à partir d'un RSS
│  └─ shodan-scan.yml       # workflow: lance shodan_search.py et commit
├─ data/
│  ├─ raw/                  # résultats bruts (json, pcap, etc.)
│  └─ reports/              # rapports formatés
├─ docs/                    # références, playbooks, runbooks
└─ LICENSE
```

---

## 📚 Sources recommandées (à suivre)

* Krebs on Security — [https://krebsonsecurity.com/](https://krebsonsecurity.com/)
* SANS NewsBites — [https://www.sans.org/newsletters/newsbites](https://www.sans.org/newsletters/newsbites)
* The Hacker News — [https://thehackernews.com/](https://thehackernews.com/)
* DarkReading — [https://www.darkreading.com/](https://www.darkreading.com/)
* BleepingComputer — [https://www.bleepingcomputer.com/](https://www.bleepingcomputer.com/)
* CISA — [https://www.cisa.gov/](https://www.cisa.gov/)
* CERT / CERT-FR / US-CERT (selon juridiction)
* Reddit r/netsec — [https://www.reddit.com/r/netsec/](https://www.reddit.com/r/netsec/)

> Astuce : stocke les flux RSS (ou les requêtes Google Alerts) et centralise leur conversion en Markdown via un workflow GitHub Actions.

---

## 🛠️ Outils & plateformes utiles

* **Shodan** — recherche d'assets exposés (API disponible)
* **Censys** — inventory d'actifs
* **VirusTotal** — analyse d'échantillons et threat intelligence
* **AlienVault OTX** — partages IOC
* **Zeek / Suricata / MISP / Arkime** — outils à déployer en local pour collecte et corrélation

---

## ⚙️ Exemple : GitHub Action — RSS → VEILLE.md

Ajoute ce fichier dans `.github/workflows/rss-to-md.yml`. Il récupère un RSS, exécute un script `scripts/rss_to_md.py` et commit `VEILLE.md`.

```yaml
name: RSS to MD
on:
  schedule:
    - cron: '0 7 * * *' # tous les jours à 07:00 UTC (ajuste selon ton fuseau)
  workflow_dispatch: {}

permissions:
  contents: write

jobs:
  update-veille:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install feedparser markdownify

      - name: Run RSS -> MD script
        run: |
          python scripts/rss_to_md.py \
            --feed "https://www.sans.org/rss/newsbites.xml" \
            --output VEILLE.md

      - name: Commit and push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add VEILLE.md || true
          git commit -m "Automated: update VEILLE.md from RSS" || echo "No changes"
          git push
```

> Remplace le paramètre `--feed` par la source RSS que tu veux surveiller.

---

## 🐍 Exemple : script minimal `scripts/shodan_search.py`

Ce script illustre comment appeler l'API Shodan et stocker les résultats en JSON. **NE PAS** mettre ta clé API dans le repo : utilise des secrets GitHub (`SHODAN_API_KEY`).

```python
#!/usr/bin/env python3
"""shodan_search.py
Usage: export SHODAN_API_KEY=... && python scripts/shodan_search.py --query "port:22 country:FR" --out data/raw/shodan_ssh_fr.json
"""
import os
import argparse
import json
from shodan import Shodan


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--query', required=True)
    parser.add_argument('--out', required=True)
    args = parser.parse_args()

    api_key = os.getenv('SHODAN_API_KEY')
    if not api_key:
        raise SystemExit('SHODAN_API_KEY not set')

    api = Shodan(api_key)
    results = api.search(args.query)

    with open(args.out, 'w') as f:
        json.dump(results, f, indent=2)

    print(f"Saved {results.get('total')} results to {args.out}")

if __name__ == '__main__':
    main()
```

---

## 🧰 Exemple : `scripts/rss_to_md.py` (simple)

```python
#!/usr/bin/env python3
"""rss_to_md.py
Converts a feed into a simple markdown summary (title, date, link, summary)
"""
import argparse
import feedparser
from datetime import datetime


def to_md(entries):
    lines = ["# VEILLE — flux RSS\n"]
    for e in entries:
        dt = e.get('published', e.get('updated', ''))
        lines.append(f"## {e.get('title')}\n")
        if dt:
            lines.append(f"- date: {dt}\n")
        lines.append(f"- link: {e.get('link')}\n")
        summary = e.get('summary', '')
        if summary:
            lines.append(f"\n{summary}\n")
        lines.append('---\n')
    return '\n'.join(lines)


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--feed', required=True)
    parser.add_argument('--output', required=True)
    parser.add_argument('--max', type=int, default=10)
    args = parser.parse_args()

    feed = feedparser.parse(args.feed)
    entries = feed.entries[: args.max]
    md = to_md(entries)
    with open(args.output, 'w') as f:
        f.write(md)
    print(f'Wrote {len(entries)} entries to {args.output}')

if __name__ == '__main__':
    main()
```

---

## 🔐 Secrets et bonnes pratiques

* **Ne jamais** mettre les clés d'API en clair dans le repo. Utilise les **GitHub Secrets** (`Settings -> Secrets and variables -> Actions`) : `SHODAN_API_KEY`, `VT_API_KEY`, etc.
* Limiter la fréquence des appels API (rate limits) et prévoir un mécanisme de backoff.
* Versionner uniquement les résumés / rapports, pas les dumps bruts sensibles (`data/raw` peut être exclu via `.gitignore`).

---

## 🧾 Exemple `.gitignore`

```
data/raw/
*.key
*.pem
.env
__pycache__/
```

---

## 🤝 Contribution

Contributions bienvenues — ouvre une *issue* ou une *pull request*.

* Ajout de sources RSS
* Nouveaux scripts d'intégration (ex : MISP, ElasticSearch)
* Playbooks et runbooks d'investigation

---

## ⚖️ Licence

Choisis une licence (MIT recommandée pour un repo public non commercial). Exemple : `LICENSE` contenant MIT.

---

## ✨ Prochaines étapes que je peux faire pour toi

* Générer automatiquement les fichiers `scripts/*` et `.github/workflows/*` (prêts à coller)
* Créer un résumé `VEILLE.md` initial à partir d'un flux RSS précis
* Adapter le README pour une organisation d'entreprise (avec playbooks) ou pour une utilisation personnelle

Si tu veux que je génère directement les fichiers (`rss_to_md.py`, `shodan_search.py`, `rss-to-md.yml`, etc.), dis-moi lesquels — je les ajoute dans des fichiers séparés dans le repo.

---

*Fin du README — bonne veille !*
