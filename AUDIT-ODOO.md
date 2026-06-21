# Audit de l'intégration Odoo (workflows n8n) — TargetUp

> Audit réalisé le **2026-06-21** sur le dépôt de backups `targetup-n8n-backups`.
> Périmètre : usage d'Odoo **par les workflows n8n**. L'instance Odoo elle-même
> (modules, droits, perfs, base de données) n'a **pas** été auditée car aucun
> connecteur MCP Odoo n'est rattaché à la session. Pour un audit *live* de l'ERP,
> brancher un MCP Odoo (XML-RPC/JSON-RPC) puis relancer.

## Méthode

- 62 fichiers de backup analysés. Chaque constat a été **vérifié 3 fois**
  (extraction `jq`, re-vérification croisée, contrôle de non-régression du diff).
- Les corrections appliquées sont **conservatrices** : elles n'altèrent pas la
  logique métier et préservent l'intégrité JSON (re-importable dans n8n).

> ⚠️ **Important** : ce dépôt est un *miroir de sauvegarde*. Corriger un fichier
> ici n'agit pas sur l'instance n8n de production. **Pour appliquer un correctif,
> il faut ré-importer le workflow corrigé dans n8n** (Workflows → Import from File).

---

## Tableau de synthèse

| # | Constat | Sévérité | Statut |
|---|---------|----------|--------|
| 1 | 52/62 backups corrompus (13 octets identiques) | 🔴 Critique | ⚠️ Action manuelle (process de backup) |
| 2 | `mouloud-commerciale` (seul actif) crée des contacts **sans dédup** | 🔴 Critique | ⚠️ Action manuelle (snippet fourni) |
| 3 | Aucune résilience sur les appels Odoo | 🔴 Haute | ✅ Corrigé (lectures) + ⚠️ doc (écritures) |
| 4 | Doublons de workflows + credential Odoo incohérent | 🔴 Haute | ⚠️ Action manuelle (procédure fournie) |
| 5 | Bug `IIF()` / `==` dans `lead-export-copy` | 🟠 Moyenne | ⚠️ Action manuelle (snippet fourni) |
| 6 | IDs/hash Odoo codés en dur | 🟠 Moyenne | ⚠️ Action manuelle (liste fournie) |
| 7 | `getAll` plafonné sans pagination | 🟠 Moyenne | ⚠️ Action manuelle (procédure fournie) |
| 8 | Aucun secret en clair / pattern create→update correct | 🟢 OK | ✅ Vérifié |

---

## 1. 🔴 52/62 backups sont corrompus

**Preuve :** 52 fichiers `.json` font exactement **13 octets** et sont **tous
byte-identiques** (un seul MD5 pour les 52). Contenu décodé :
`7e 29 5e efbfbd 2b 2d 7a 6f efbfbd` — soit `~)^□+-zo□` où `□` est le caractère
de remplacement UTF-8 (U+FFFD). Il n'existe **qu'un seul commit** dans le dépôt
(`b9fd5cd`) : aucune version saine n'existe en historique.

**Conséquence :** ce ne sont pas des sauvegardes — aucune donnée de workflow n'y
est récupérable. Workflows Odoo perdus, notamment :
- `wf-01-collecte-aos-rss-odoo-targetup`
- `liste-de-produits-odoo`
- `targetup-email-prospects-odoo`
- `veille-aos-world-bank-pnud-matching-consultants-odoo`

**Cause probable :** le workflow de backup (`backup-workflows-github-*`, lui-même
à 13 octets) écrit un blob constant à la place du JSON — typiquement une étape de
chiffrement/encodage appliquée à une entrée vide, puis mal sérialisée en UTF-8
(les `U+FFFD` indiquent une conversion *lossy* d'octets binaires).

**Remédiation (à faire dans n8n) :**
1. Ré-exporter **manuellement** tous les workflows depuis n8n (`...` → *Download*)
   pour reconstituer un backup sain immédiatement.
2. Corriger le workflow de backup :
   - récupérer le JSON via `GET /api/v1/workflows/{id}` (n8n REST API) **sans**
     étape de chiffrement/compression intermédiaire ;
   - committer le `body` brut (`response.data`) en UTF-8 ;
   - ajouter un **garde-fou** : rejeter tout export `< 100 octets` ou qui ne
     `JSON.parse()` pas, et lever une alerte vers le DLQ.
3. Ajouter un contrôle d'intégrité post-backup (ex. `jq empty` sur chaque fichier).

---

## 2. 🔴 `mouloud-commerciale` crée des contacts sans déduplication (workflow ACTIF)

C'est le **seul workflow actif** (`scheduleTrigger`). Il exécute un
`Create a contact` (res.partner) **sans rechercher l'existant** → génération de
doublons à chaque exécution planifiée. Idem pour `lead-export-copy` et
`veille-ao-multi-sources` (inactifs, mais même défaut sur `crm.lead`).

**Remédiation :** insérer un nœud Odoo `Search` **avant** chaque `Create`, puis un
`IF` qui ne crée que si aucun enregistrement n'existe (ou bascule en `update`).

Exemple de nœud de dédup (à placer avant `Create a contact`, à adapter au champ clé) :

```json
{
  "name": "Odoo — Rechercher contact existant",
  "type": "n8n-nodes-base.odoo",
  "parameters": {
    "resource": "custom",
    "customResource": "res.partner",
    "operation": "getAll",
    "returnAll": false,
    "limit": 1,
    "filterRequest": {
      "filter": [
        { "fieldName": "email", "operator": "equal",
          "value": "={{ $json['Email'] }}" }
      ]
    },
    "options": { "fieldsList": "id,email" }
  },
  "retryOnFail": true,
  "maxTries": 3,
  "waitBetweenTries": 2000,
  "credentials": { "odooApi": { "id": "nc94GpArgDtxKthl", "name": "Odoo account API KEY" } }
}
```

Puis un `IF` : `{{ $json.id }}` vide → branche `Create` ; sinon → `Update` (en
passant `customResourceId = {{ $json.id }}`).

> Tant que la dédup n'est pas en place, **ne pas** activer de retry sur les
> `create`/`update` (un retry après timeout dupliquerait l'enregistrement) —
> c'est pourquoi le correctif #3 ne touche que les lectures.

---

## 3. 🔴→✅ Résilience des appels Odoo

**Constat :** les 10 nœuds Odoo étaient en `onError = STOP` et sans retry → toute
erreur transitoire (timeout, lock) stoppait le workflow.

**Correctif appliqué (ce commit)** — retry x3 / attente 2 s sur les **7 lectures
`getAll`** (idempotentes, donc sûres à retenter) :

| Fichier | Nœuds durcis |
|---|---|
| `wf-05-crm-rescore-…` | `Odoo — Récupérer 1000 leads actifs` |
| `wf-07-reporting-direction-eJwfZleJ…` | `Leads N-1`, `AO N-1`, `Appels IA N-1` |
| `wf-07-reporting-direction-shEKlBIjPj4…` | `Leads N-1`, `AO N-1`, `Appels IA N-1` |

Clés ajoutées : `"retryOnFail": true, "maxTries": 3, "waitBetweenTries": 2000`.
Diff vérifié : **aucune autre clé modifiée**, nombre de nœuds inchangé, JSON valide.

**Reste à faire manuellement (écritures `create`/`update`) :** activer le retry
**après** avoir mis en place la déduplication (#2), pour ne pas créer de doublons.
Nœuds concernés : `ismail.kpi.snapshot`, `Archivage Chatter DG` (mail.message),
`veille-ao` create+update, `mouloud` create, `lead-export` create.

---

## 4. 🔴 Doublons de workflows + credential Odoo incohérent

Trois workflows existent en **double** (même nom, configs divergentes) :
`WF-07 REPORTING_DIRECTION`, `WF-03 CRM_SOCIAL_INBOX`, `WF-00 GLOBAL_DLQ_HANDLER`.

Pour WF-07, les deux versions utilisent **des credentials Odoo différents** :

| Fichier | Nœuds | Credential Odoo |
|---|---|---|
| `…eJwfZleJ70sGQN05` | 13 (WhatsApp + snapshot KPI) — **plus complet** | `nc94GpArgDtxKthl` « Odoo account API KEY » |
| `…shEKlBIjPj4tCGlX` | 9 (sans KPI) | `qoDeeleL6DjwWtQp` « Odoo account » |

**Remédiation (dans n8n) :**
1. Garder la version `eJwfZleJ` (la plus complète), archiver/supprimer `shEKlBIjPj4`.
2. **Unifier le credential Odoo** sur un seul (recommandé : `nc94GpArgDtxKthl`)
   après avoir vérifié que les deux pointent bien vers la **même instance/le même
   utilisateur** Odoo. Supprimer le credential redondant pour éviter toute dérive.
3. Idem pour les doublons WF-03 et WF-00.

> La déduplication doit se faire **dans n8n** (la prod), pas en supprimant des
> fichiers de backup : retirer un fichier ici ne change rien à l'instance.

---

## 5. 🟠 Bug `IIF()` / `==` dans `lead-export-copy`

Le champ `description` du `Create an item` commence par `==` et utilise **15 appels
`IIF(...)`**. Or :
- `IIF` n'existe pas en JavaScript (syntaxe VBA/Excel) ;
- une valeur n8n démarrant par `=` est une expression, mais le code n'est pas
  encadré par `{{ }}` → le contenu est inséré **littéralement** (le code source
  brut finit dans la description du lead).

**Remédiation :** encadrer par `={{ ... }}` et remplacer chaque
`IIF(cond, valeur, '')` par un ternaire `(cond ? valeur : '')`. Schéma :

```js
={{
  '<b>Date</b> : ' + $now.format('yyyy-MM-dd') + '<br><br>' +
  '<b>Lead issu du Catalogue Export ZLECAF</b><br>' +
  ( $('Loop Over Items').item.json.Secteur === 'Agro-alimentaire'
      ? '<b>1. Certification ISO 22000 / HACCP</b>…'
      : '' ) +
  ( $('Loop Over Items').item.json.Secteur === 'Agriculture'
      ? '…' : '' )
  /* …un ternaire par secteur… */
}}
```

Workflow inactif → aucun impact prod tant qu'il n'est pas réactivé, mais à
corriger avant toute réactivation.

---

## 6. 🟠 IDs et hash Odoo codés en dur

Valeurs en dur disséminées dans les nœuds (fragiles en cas de migration ou de
régénération des propriétés Studio — échec **silencieux** du mapping) :

- `country_id = 136`, `team_id = 16`, `source_id = 398`, `user_id = 329`,
  `priority = 3`.
- `veille-ao` → `lead_properties` avec des **hash de propriétés Studio** en dur :
  `e7d6e8ea190d4ca6`, `dee56c6c1afbe794`, `6c8a4697f1207a00`, `94725c5f29d14548`,
  `e0648c499207fcd1`, `744143249c04537a`, et valeurs de sélection
  `18d02919b7324473` (GO), `8782e3c3665fee24` (NO_GO), `183e6d717b50c448` (REVIEW).

**Remédiation :** centraliser ces IDs dans un nœud `Set` (ou des variables
d'environnement n8n) en tête de workflow, documentés et référencés par expression.

---

## 7. 🟠 `getAll` plafonné sans pagination

Lectures `crm.lead` / `ismail.ai.request` limitées (`limit` 200/500/1000,
`returnAll` absent). Au-delà du plafond, les enregistrements sont **ignorés sans
alerte** → rescore et KPI partiels.

**Remédiation :** passer `returnAll: true` (avec pagination) **ou** ajouter un
contrôle « nombre de résultats == limite » qui lève une alerte de troncature.

---

## 8. 🟢 Points conformes (vérifiés)

- **Aucun secret en clair** : les credentials Odoo sont référencés par ID, n8n ne
  les exporte pas. Bonne hygiène.
- **Pattern create→update correct** dans `veille-ao` :
  `customResourceId = {{ $json.id }}` récupère bien l'ID du lead créé.
- Bonne intention d'architecture d'erreurs (`WF-00 GLOBAL_DLQ_HANDLER`) et
  d'archivage (Chatter `mail.message`, snapshots `ismail.kpi.snapshot`).

---

## Cartographie Odoo

- **Modèles standard :** `crm.lead` (R/W), `res.partner` (create), `mail.message` (create).
- **Modèles custom :** `ismail.ai.request`, `ismail.kpi.snapshot`.
- **Champs custom :** `x_ai_score`, `x_ai_score_badge`, `x_ai_is_vip`, `x_secteur`,
  `x_pays`, `x_record_type`, `x_score_ao`, `x_go_nogo`, `x_ao_submitted`,
  `x_ao_won`, `x_budget_ao`, `x_ismail_source`, `x_ai_processed_date`.
- **Credentials :** `nc94GpArgDtxKthl` (« Odoo account API KEY », 5 wf) et
  `qoDeeleL6DjwWtQp` (« Odoo account », 1 wf — à unifier).

---

## Prochaines étapes recommandées (ordre de priorité)

1. **Réparer le process de backup** (#1) — priorité absolue : 84 % des sauvegardes
   sont inexploitables.
2. **Ajouter la dédup** dans `mouloud-commerciale` (#2) — seul workflow actif.
3. **Étendre le retry aux écritures** une fois la dédup en place (#3).
4. **Dédupliquer les workflows + unifier le credential** (#4).
5. Corriger `IIF`/`==` (#5), externaliser les IDs (#6), gérer la pagination (#7).
6. **Brancher un MCP Odoo** pour un audit *live* de l'ERP (doublons réels, droits
   d'accès, modules, performances).
