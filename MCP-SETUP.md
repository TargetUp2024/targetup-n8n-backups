# Brancher les MCP n8n + Odoo sur Claude

Objectif : permettre à Claude de **piloter votre n8n** (créer/valider/exécuter des
workflows) **et votre Odoo** (lire/chercher/créer/mettre à jour des
enregistrements) — pour « gérer le max ».

Deux serveurs MCP sont préconfigurés dans [`.mcp.json`](./.mcp.json) :

| Serveur | Paquet | Ce qu'il permet |
|---|---|---|
| `n8n` | [`n8n-mcp`](https://github.com/czlonkowski/n8n-mcp) (npx) | Doc des 500+ nœuds, validation, **création/MAJ/exécution** de workflows |
| `odoo` | [`mcp-server-odoo`](https://pypi.org/project/mcp-server-odoo/) (uvx) | Recherche, lecture, **création/MAJ** d'enregistrements via XML-RPC |

> ⚠️ **Important — où l'exécuter.** Ce dépôt cloud ne peut pas joindre votre n8n/Odoo
> interne (n8n est exposé en interne, ex. `targetup:8019`). Lancez Claude Code
> **sur une machine du réseau qui atteint n8n et Odoo** (ou un environnement web
> dont la politique réseau autorise ces hôtes). Les secrets ne sont **jamais**
> committés : ils restent dans un `.env` local (ignoré par git).

---

## Prérequis

- **Node.js 18+** (fournit `npx`) pour le MCP n8n.
- **UV** (`uvx`) pour le MCP Odoo : `curl -LsSf https://astral.sh/uv/install.sh | sh`
- Claude Code installé sur la machine cible.

## Étape 1 — Clé API n8n

1. Dans n8n : **Settings → n8n API → Create an API key**.
2. Notez l'URL de l'API (votre instance, ex. `https://n8n.targetup…` ou `http://localhost:5678`).

## Étape 2 — Activer le MCP côté Odoo

Le MCP Odoo nécessite un module **dans** Odoo :

1. Installez le module **`mcp_server`** depuis l'Odoo App Store (Odoo 16.0+).
2. **Paramètres → MCP Server** : activez le MCP.
3. **Paramètres → MCP Server → Enabled Models** : ajoutez les modèles à exposer.
   Pour vos workflows, activez au minimum :
   - `crm.lead`, `res.partner`, `product.product`, `mail.message`
   - vos modèles custom : `ismail.ai.request`, `ismail.kpi.snapshot`
4. **Paramètres → Utilisateurs → (votre user) → onglet « Clés API »** : créez une clé.
   > Conseil : créez un **utilisateur de service dédié** (`svc_mcp`) avec des droits
   > limités aux modèles ci-dessus, plutôt que d'utiliser un compte admin.
5. Notez `ODOO_URL`, `ODOO_DB` et la clé `ODOO_API_KEY`.

## Étape 3 — Renseigner les secrets

```bash
cp .env.example .env
# éditez .env avec vos vraies valeurs (N8N_API_URL, N8N_API_KEY, ODOO_URL, ODOO_DB, ODOO_API_KEY)
```

`.env` est déjà ignoré par git (voir `.gitignore`). Chargez-le avant de lancer Claude :

```bash
set -a; source .env; set +a
claude            # Claude Code lit .mcp.json et substitue ${VARS}
```

## Étape 4 — Vérifier

Dans Claude Code :

```
/mcp           # doit lister les serveurs "n8n" et "odoo" comme connectés
```

Puis testez : « Liste mes workflows n8n » et « Combien de crm.lead créés cette semaine dans Odoo ? ».

---

## Alternative : environnement Claude Code web

Si vous préférez le web, configurez les serveurs MCP et les variables d'environnement
**à la création de l'environnement** (ils ne peuvent pas être ajoutés à chaud dans une
session existante), et choisissez une **politique réseau** autorisant vos hôtes n8n/Odoo.
Doc : https://code.claude.com/docs/en/claude-code-on-the-web

---

## Garde-fous recommandés

- **Compte de service à droits limités** côté Odoo (pas d'admin).
- Côté n8n, une clé API peut tout faire : limitez qui détient le `.env`.
- Démarrez en **lecture seule** (poser des questions, auditer) avant d'autoriser
  les écritures, surtout vu les risques de **doublons** identifiés dans
  [`AUDIT-ODOO.md`](./AUDIT-ODOO.md) (pas de déduplication sur les `create`).
- Une fois branché, l'audit *live* de l'ERP devient possible (doublons réels,
  droits, modules, volumétrie).
