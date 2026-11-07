# mail-assistant

Agent quasi autonome de réponse aux mails de Maxence Veber, construit avec le SDK Agents de Mistral (`mistralai-agents`).

## Capacités

- Lit et recherche dans les boîtes mail connectées (connecteur `outlook` : search/open/folders/brouillons). Le connecteur ne permet pas l'envoi direct : l'agent crée des **brouillons** dans Outlook, validés puis envoyés manuellement. Boîte principale : compte connecté (`maxence.veber@outlook.fr`) ; boîte secondaire `maxence.veber@viacesi.fr` via le paramètre `mailbox`.
- Décide pour chaque mail s'il faut répondre : brouillon créé directement pour les cas simples, ou texte présenté dans le chat pour validation avant création du brouillon (cas sensibles).
- Trie automatiquement les mails traités dans des dossiers **existants** (`move_email` autorisé) ; la corbeille (`deleteditems`/`junkemail`) reste soumise à approbation (phase d'apprentissage de la suppression).
- Rédige dans le style et avec la signature apprises des mails envoyés (`sentitems`).
- Retrouve et lit des fichiers sur SharePoint/OneDrive (connecteur `sharepoint_mcp`, liens OneDrive via `get_info_by_url`).
- Interpréteur de code : sandbox `MistralSandbox` (shell + Python dans un conteneur isolé hébergé par Mistral) pour calculer/agrésger les données extraites des mails et fichiers. **Prérequis déploiement** : extra `koyeb` (installé) et rôle workspace `cloud_user`.
- Chiffres réels issus des outils uniquement, avec source citée.
- Mémoire : sessions persistantes stockées par Mistral (questions de suivi, fichiers ouverts, journal de confiance).
- Autonomie croissante : « journal de confiance » tenu dans la session — chaque décision validée par l'utilisateur est réappliquée automatiquement aux cas similaires (jamais aux suppressions/envois).

## Politique de permission (vérifiée via `agents info`)

| Règle | Décision |
|---|---|
| lectures, brouillons, outils sandbox | allow |
| `outlook.move_email` (tri) | allow, sauf vers `deleteditems`/`junkemail` → ask |
| `sharepoint_mcp.delete*` | deny (apprentissage uniquement) |
| autres écritures (`send`, `create`, `upload`, `share`…) | ask |

## Déploiement géré (Mistral Cloud)

Prérequis : dépôt GitHub `maxenceveber1112-dev/mail-assistant` contenant ce projet, app GitHub MistralAI installée sur le dépôt, rôle Workflows (Developer/Admin) + `cloud_user` sur le workspace.

```bash
uv run agents deploy        # construit le commit poussé (Dockerfile fourni), hardened (OBO)
```

Ou via l'API MCP : `deploy_workflow_deployment` sur le dépôt, `hardened=true`.

## Interaction via Vibe

- **Vibe web** : `https://chat.mistral.ai/managed-agents/mail-assistant` — chaque conversation = une session persistante ; questions de suivi dans la même conversation ; approbations d'outils dans le chat.
- **CLI** : `uv run agents chat mail-assistant` (après `agents deploy`), ou localement `uv run --env-file .env agents chat` avec une clé `MISTRAL_API_KEY` créée sur console.mistral.ai.
- **API Session** : `agents.Agent.load_from(agent_name="mail-assistant")` puis `create_session()` / `Agent.load_session(session_id)`.
- Sessions et traces : Mistral Studio → Managed Agents → Agent sessions.

## Premier test

1. Ouvrir `https://chat.mistral.ai/managed-agents/mail-assistant` et envoyer : « Liste mes 10 derniers mails non lus et dis-moi lesquels demandent une réponse. »
2. Puis : « Rédige un brouillon de réponse pour le mail de X. » → le brouillon apparaît dans Outlook (Brouillons), rien n'est envoyé.
3. « Quel est le total des montants mentionnés dans mes 20 derniers mails ? » → teste l'interpréteur de code (sandbox).
4. Questions de suivi dans la même conversation : la session se souvient de tout.
5. Traces : Studio → Managed Agents → sessions.

## Configuration

- `MAIL_ASSISTANT_CONNECTORS` (ENV du Dockerfile) : connecteurs attachés, défaut `outlook,sharepoint_mcp`.
- Modèle `mistral-large-latest` et nom `mail-assistant` dans `main.py`.
