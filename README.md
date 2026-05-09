# heliorank-chat

Cloudflare Worker proxy vers Anthropic API (Claude Sonnet 4.6) pour le chatbot HELIORANK sur heliorank.lu.

## Premier déploiement (5 min)

### 1. Installer wrangler local
```bash
cd C:\Dev\heliorank-chat
npm install
```

### 2. Login wrangler
```bash
npx wrangler login
```

### 3. Créer le KV namespace pour rate-limit
```bash
npx wrangler kv namespace create CHAT_KV
```

Cela retourne un ID type `id = "abc123..."`. Copier cet ID dans `wrangler.toml` à la place de `REPLACE_AFTER_WRANGLER_KV_CREATE`.

### 4. Ajouter la clé API Anthropic en Secret
```bash
npx wrangler secret put ANTHROPIC_API_KEY
```
Coller la clé `sk-ant-...` quand demandé. Récupérable sur https://console.anthropic.com → API Keys.

### 5. Déployer
```bash
npm run deploy
```

Le Worker est maintenant accessible à :
- `https://heliorank-chat.<your-subdomain>.workers.dev/api/chat`

Le sous-domaine exact est affiché par wrangler à la fin du déploiement. Note-le pour mettre à jour `chat-widget.js` côté heliorank-static.

### 6. Test rapide
```bash
curl -X POST https://heliorank-chat.<your-subdomain>.workers.dev/health
# attendu : {"status":"ok","service":"heliorank-chat"}

curl -X POST https://heliorank-chat.<your-subdomain>.workers.dev/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Combien coûte un audit SEO ?"}'
# attendu : stream SSE avec la réponse en plusieurs chunks
```

## Maintenance

### Voir les logs en temps réel
```bash
npm run tail
```

### Mettre à jour la clé API
```bash
npx wrangler secret put ANTHROPIC_API_KEY
```

### Mettre à jour le system prompt
Éditer `src/worker.js` constante `SYSTEM_PROMPT`, puis :
```bash
npm run deploy
```

### Modifier le rate limit
Éditer `wrangler.toml` :
```toml
MAX_MESSAGES_PER_HOUR = "10"  # défaut
```
Puis redéployer.

## Coût estimé

- Worker invocations : 10 000 requêtes/mois gratuites, puis 0,30 $/M
- KV reads/writes : 100 000/mois gratuits puis 0,50 $/M
- Anthropic API Sonnet 4.6 : ~3 $ / M tokens input, ~15 $ / M tokens output
- À 50 conversations/mois × 7 KB tokens (input+output) : ~2,25 € de coût Anthropic

Total : ~2-5 €/mois au volume B2B attendu.

## Architecture

```
Browser (heliorank.lu)
   │
   ▼ POST /api/chat (CORS limité à heliorank.lu)
Worker (heliorank-chat)
   │── Rate limit IP (KV CHAT_KV, TTL 1h, max 10/h)
   │── Validation longueur message
   │── Sanitize history
   │── Inject system prompt strict
   │
   ▼ POST api.anthropic.com/v1/messages (stream)
Anthropic API (claude-sonnet-4-5-20250929)
   │
   ▼ stream SSE
Worker forwards stream to browser
   │
   ▼
Browser displays tokens in real-time
```

## Sécurité

- Clé API Anthropic stockée en Cloudflare Secret (jamais dans le code, jamais côté frontend)
- CORS limité à heliorank.lu uniquement
- Rate limit IP 10 messages/heure
- Validation longueur message max 2000 chars
- Validation history max 10 messages, chacun max 2000 chars
- System prompt avec règles strictes anti-injection (tentatives de role-play, demandes hors scope)
- Aucune persistance de conversation côté serveur (stateless, conversation client-side)

## Limites connues

- Rate limit basé IP (un open NAT d'entreprise = 1 IP partagée). Acceptable au volume B2B prévu.
- Pas de logs de conversation (privacy by design). Si besoin de review, ajouter un endpoint `/api/log` qui stocke en KV avec TTL court.
- KV eventually consistent : un utilisateur très rapide peut faire 11-12 requêtes au lieu de 10. Acceptable.
