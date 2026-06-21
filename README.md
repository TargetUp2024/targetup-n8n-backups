# targetup-n8n-backups

Sauvegardes des workflows n8n de TargetUp.

## Connexion MCP (n8n + Odoo)

Configuration prête à l'emploi pour piloter n8n et Odoo depuis Claude :
voir [`MCP-SETUP.md`](./MCP-SETUP.md) (config dans [`.mcp.json`](./.mcp.json),
secrets dans un `.env` local non committé).

## Audit Odoo

Un audit de l'intégration Odoo (usage par les workflows n8n) est disponible :
voir [`AUDIT-ODOO.md`](./AUDIT-ODOO.md).

> ⚠️ **Alerte intégrité backup** : 52 des 62 fichiers de ce dépôt sont corrompus
> (13 octets identiques, aucune donnée de workflow récupérable). Voir le constat #1
> de l'audit. Ré-exporter les workflows depuis n8n et corriger le workflow de
> backup en priorité.
