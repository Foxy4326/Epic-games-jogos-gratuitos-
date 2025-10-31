// server.js
require('dotenv').config();
const express = require('express');
const session = require('express-session');
const axios = require('axios');
const qs = require('querystring');
const path = require('path');

const app = express();
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Sessão (em produção: store mais robusta, cookie.secure: true e HTTPS)
app.use(session({
  secret: process.env.SESSION_SECRET || 'dev_secret',
  resave: false,
  saveUninitialized: true,
  cookie: { secure: false, maxAge: 24*60*60*1000 }
}));

const PORT = process.env.PORT || 3000;
const CLIENT_ID = process.env.EPIC_CLIENT_ID;
const CLIENT_SECRET = process.env.EPIC_CLIENT_SECRET;
const REDIRECT_URI = process.env.REDIRECT_URI;

// Endpoints (pontos de partida — podem variar conforme docs da Epic)
const AUTHORIZE_URL = 'https://www.epicgames.com/id/authorize';
const TOKEN_URL = 'https://api.epicgames.dev/epic/oauth/v1/token';
const USERINFO_URL = 'https://api.epicgames.dev/epic/oauth/v1/userInfo';
const FREE_GAMES_API = 'https://store-site-backend-static.ak.epicgames.com/freeGamesPromotions?locale=pt-BR';

// ---------- ROTAS ----------

// Página principal (serve HTML + JS inline)
app.get('/', (req, res) => {
  const user = req.session.user || null;
  const html = `
<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>GameAlerts — Epic Login Real</title>
<script src="https://cdn.tailwindcss.com"></script>
<style>
  body{background:#0f172a;color:#fff;font-family:Inter,system-ui,Arial;min-height:100vh;margin:0;padding:20px}
  .btn{background:#0074e4;padding:10px 14px;border-radius:8px;color:#fff;border:none;cursor:pointer}
  .card{background:#0b1220;padding:12px;border-radius:10px;margin-bottom:10px}
  .owned{color:#00ff99;font-weight:700}
  .notowned{color:#ff6b6b;font-weight:700}
</style>
</head>
<body>
  <h1>🎮 GameAlerts — Jogos GRÁTIS (Epic)</h1>
  <div style="margin-top:10px">
    ${ user ? `<div class="card">Logado: <strong>${user.displayName || user.accountId || user.name}</strong> — <a href="/logout" class="btn">Logout</a></div>` 
            : `<a href="/login" class="btn">🔐 Login com Epic Games</a>` }
  </div>

  <div id="status" style="margin-top:16px">Carregando jogos gratuitos...</div>
  <div id="games"></div>

<script>
async function fetchFreeGames(){
  document.getElementById('status').innerText = 'Carregando jogos gratuitos...';
  try{
    const r = await fetch('/api/free-games');
    const json = await r.json();
    const elements = json?.data?.Catalog?.searchStore?.elements || [];
    const freeGames = elements.filter(g => g.price?.totalPrice?.discountPrice === 0 && g.promotions?.promotionalOffers?.length>0);
    renderGames(freeGames);
    document.getElementById('status').innerText = 'Atualizado';
  }catch(e){
    console.error(e);
    document.getElementById('status').innerText = 'Erro ao carregar jogos';
  }
}

function asOfferId(game){
  // tenta extrair um identificador utilizável para consulta — ajuste conforme payload real
  const promo = game.promotions?.promotionalOffers?.[0]?.promotionalOffers?.[0];
  return promo?.offerId || promo?.id || game.catalogNs?.mappings?.[0]?.pageSlug || game.productSlug || game.title;
}

async function renderGames(games){
  const container = document.getElementById('games');
  container.innerHTML = '';
  if(games.length===0){ container.innerHTML = '<p>Nenhum jogo gratuito encontrado.</p>'; return; }
  for(const g of games){
    const offerId = encodeURIComponent(asOfferId(g));
    const namespace = encodeURIComponent(g.catalogNs?.mappings?.[0]?.namespace || '');
    const el = document.createElement('div');
    el.className = 'card';
    el.innerHTML = \`
      <div style="display:flex;gap:12px;align-items:center">
        <img src="\${g.keyImages?.[0]?.url || 'https://via.placeholder.com/200x100?text=No+Image'}" width="160" height="90" style="object-fit:cover;border-radius:6px">
        <div style="flex:1">
          <h3 style="margin:0">\${g.title}</h3>
          <p style="margin:4px 0">De R$\${(g.price?.totalPrice?.originalPrice/100).toFixed(2)} → <strong>GRÁTIS</strong></p>
          <div id="owned-\${offerId}">Verificando posse...</div>
          <div style="margin-top:8px"><a href="https://store.epicgames.com/pt-BR/p/\${g.productSlug}" target="_blank" class="btn">Ir à loja / Resgatar</a></div>
        </div>
      </div>\`;
    container.appendChild(el);

    // checar se o usuário possui (rota backend)
    try {
      const r2 = await fetch('/api/check-owned?offerId=' + offerId + '&offerNamespace=' + namespace);
      const j = await r2.json();
      const statusEl = document.getElementById('owned-' + offerId);
      if(j.error === 'not_logged'){ statusEl.innerHTML = 'Faça login para verificar posse'; }
      else if(j.owned){ statusEl.innerHTML = '<span class="owned">Resgatado ✅</span>'; }
      else { statusEl.innerHTML = '<span class="notowned">Não resgatado ❌</span>'; }
    } catch(err){
      console.error('Erro check-owned', err);
      const statusEl = document.getElementById('owned-' + offerId);
      statusEl.innerText = 'Erro ao verificar';
    }
  }
}

fetchFreeGames();
// opcional: atualizar a cada 10 minutos
setInterval(fetchFreeGames, 600000);
</script>
</body>
</html>
  `;
  res.send(html);
});

// Inicia o fluxo OAuth (redireciona para Epic)
app.get('/login', (req, res) => {
  const params = {
    client_id: CLIENT_ID,
    response_type: 'code',
    scope: 'basic_profile offline_access',
    redirect_uri: REDIRECT_URI
  };
  res.redirect(AUTHORIZE_URL + '?' + qs.stringify(params));
});

// Callback - troca code por token e pega user info
app.get('/callback', async (req, res) => {
  const code = req.query.code;
  if(!code) return res.status(400).send('No code provided');

  try {
    // troca code por token
    const tokenResp = await axios.post(TOKEN_URL, qs.stringify({
      grant_type: 'authorization_code',
      code,
      redirect_uri: REDIRECT_URI
    }), {
      auth: { username: CLIENT_ID, password: CLIENT_SECRET },
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
    });

    const tokens = tokenResp.data;
    req.session.tokens = tokens;

    // pega user info (accountId)
    const userInfoResp = await axios.get(USERINFO_URL, {
      headers: { Authorization: 'Bearer ' + tokens.access_token }
    });

    req.session.user = userInfoResp.data;
    // redireciona para a home
    res.redirect('/');
  } catch (err) {
    console.error('callback error', err.response?.data || err.message);
    res.status(500).send('Erro no callback: ' + (err.response?.data?.error_description || err.message));
  }
});

// Proxy para API de free-games (evita CORS e centraliza requests)
app.get('/api/free-games', async (req, res) => {
  try {
    const r = await axios.get(FREE_GAMES_API);
    res.json(r.data);
  } catch (err) {
    console.error('free-games error', err.message || err);
    res.status(500).json({ error: 'fetch_error' });
  }
});

// Verifica se o usuário possui o item (entitlement/ownership)
// Nota: a real estrutura de entitlements pode variar — adapte verificação de campos conforme resposta real.
app.get('/api/check-owned', async (req, res) => {
  const tokens = req.session.tokens;
  const user = req.session.user;
  const { offerId, offerNamespace } = req.query;

  if(!tokens || !user) return res.json({ owned: false, error: 'not_logged' });

  try {
    // identityId pode estar em diferentes campos dependendo do userInfo — adapte se necessário
    const identityId = user.accountId || user.id || (user.account && user.account.id) || null;
    if(!identityId) return res.json({ owned: false, error: 'no_identity' });

    // Endpoint exemplo (Ecom/Entitlements) — pode requerer namespace/deployment
    const entUrl = \`https://api.epicgames.dev/epic/ecom/v1/identities/\${encodeURIComponent(identityId)}/entitlements\`;
    const r = await axios.get(entUrl, {
      headers: { Authorization: 'Bearer ' + tokens.access_token },
      params: { namespace: offerNamespace || undefined }
    });

    const list = r.data?.entitlements || r.data?.items || r.data?.results || [];
    // comparar por campos comuns — ajuste conforme resposta
    const owned = list.some(e => {
      return (e.offerId && e.offerId === offerId) || (e.catalogItemId && e.catalogItemId === offerId)
             || (e.productId && e.productId === offerId) || (e.id && e.id === offerId);
    });

    res.json({ owned });
  } catch (err) {
    console.error('check-owned error', err.response?.data || err.message);
    // Possíveis causas: token expirado — neste caso, implementar refresh token
    res.status(500).json({ error: 'api_error', detail: err.response?.data || err.message });
  }
});

// Logout
app.get('/logout', (req, res) => {
  req.session.destroy(() => res.redirect('/'));
});

// Start
app.listen(PORT, () => {
  console.log('Server rodando em http://localhost:' + PORT);
});
