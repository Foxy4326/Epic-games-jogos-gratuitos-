require('dotenv').config();
const express = require('express');
const session = require('express-session');
const axios = require('axios');
const qs = require('querystring');

const app = express();
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
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

const AUTHORIZE_URL = 'https://www.epicgames.com/id/authorize';
const TOKEN_URL = 'https://api.epicgames.dev/epic/oauth/v1/token';
const USERINFO_URL = 'https://api.epicgames.dev/epic/oauth/v1/userInfo';
const FREE_GAMES_API = 'https://store-site-backend-static.ak.epicgames.com/freeGamesPromotions?locale=pt-BR';

// ---------- FRONTEND ----------
app.get('/', (req, res) => {
  const user = req.session.user || null;
  const html = `
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>GameAlerts — Epic Login Real</title>
<script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-900 text-white min-h-screen p-6">
  <h1 class="text-3xl font-bold mb-4">🎮 GameAlerts — Jogos GRÁTIS (Epic)</h1>

  <div class="mb-4">
    ${ user ? `<div class="p-3 bg-gray-800 rounded">Logado: <strong>${user.displayName || user.accountId}</strong> — <a href="/logout" class="px-3 py-1 bg-blue-600 rounded hover:bg-blue-700">Logout</a></div>` 
            : `<a href="/login" class="px-3 py-2 bg-blue-600 rounded hover:bg-blue-700">🔐 Login com Epic Games</a>` }
  </div>

  <div id="status" class="mb-2">Carregando jogos gratuitos...</div>
  <div id="games" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4"></div>

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
    const div = document.createElement('div');
    div.className = 'p-3 bg-gray-800 rounded';
    div.innerHTML = \`
      <img src="\${g.keyImages?.[0]?.url || 'https://via.placeholder.com/400x200?text=No+Image'}" class="w-full h-40 object-cover rounded mb-2">
      <h2 class="text-lg font-bold">\${g.title}</h2>
      <p>De R$\${(g.price?.totalPrice?.originalPrice/100).toFixed(2)} → <span class="font-bold text-green-400">GRÁTIS</span></p>
      <div id="owned-\${offerId}" class="mb-2">Verificando posse...</div>
      <a href="https://store.epicgames.com/pt-BR/p/\${g.productSlug}" target="_blank" class="inline-block px-3 py-1 bg-blue-600 rounded hover:bg-blue-700">Ir à loja / Resgatar</a>
    \`;
    container.appendChild(div);

    // check ownership
    try{
      const r2 = await fetch('/api/check-owned?offerId='+offerId+'&offerNamespace='+namespace);
      const j = await r2.json();
      const statusEl = document.getElementById('owned-'+offerId);
      if(j.error === 'not_logged') statusEl.innerText = 'Faça login para verificar';
      else if(j.owned) statusEl.innerHTML = '<span class="text-green-400 font-bold">Resgatado ✅</span>';
      else statusEl.innerHTML = '<span class="text-red-500 font-bold">Não resgatado ❌</span>';
    }catch(err){
      console.error(err);
      document.getElementById('owned-'+offerId).innerText = 'Erro ao verificar';
    }
  }
}

fetchFreeGames();
setInterval(fetchFreeGames,600000);
</script>
</body>
</html>
  `;
  res.send(html);
});

// Login e callback OAuth
app.get('/login', (req,res)=>{
  const params = { client_id:CLIENT_ID, response_type:'code', scope:'basic_profile offline_access', redirect_uri:REDIRECT_URI };
  res.redirect(AUTHORIZE_URL + '?' + qs.stringify(params));
});

app.get('/callback', async (req,res)=>{
  const code = req.query.code;
  if(!code) return res.status(400).send('No code provided');
  try{
    const tokenResp = await axios.post(TOKEN_URL, qs.stringify({grant_type:'authorization_code', code, redirect_uri:REDIRECT_URI}), {
      auth:{username:CLIENT_ID,password:CLIENT_SECRET},
      headers:{'Content-Type':'application/x-www-form-urlencoded'}
    });
    const tokens = tokenResp.data;
    req.session.tokens = tokens;

    const userInfoResp = await axios.get(USERINFO_URL,{headers:{Authorization:'Bearer '+tokens.access_token}});
    req.session.user = userInfoResp.data;
    res.redirect('/');
  }catch(err){
    console.error(err.response?.data||err.message);
    res.status(500).send('Erro no callback');
  }
});

// Proxy free games
app.get('/api/free-games', async (req,res)=>{
  try{
    const r = await axios.get(FREE_GAMES_API);
    res.json(r.data);
  }catch(err){
    console.error(err);
    res.status(500).json({error:'fetch_error'});
  }
});

// Check ownership
app.get('/api/check-owned', async (req,res)=>{
  const tokens = req.session.tokens;
  const user = req.session.user;
  const { offerId, offerNamespace } = req.query;
  if(!tokens || !user) return res.json({owned:false,error:'not_logged'});
  try{
    const identityId = user.accountId || user.id || (user.account && user.account.id) || null;
    if(!identityId) return res.json({owned:false,error:'no_identity'});
    const entUrl = \`https://api.epicgames.dev/epic/ecom/v1/identities/\${encodeURIComponent(identityId)}/entitlements\`;
    const r = await axios.get(entUrl,{headers:{Authorization:'Bearer '+tokens.access_token}, params:{namespace:offerNamespace||undefined}});
    const list = r.data?.entitlements||r.data?.items||r.data?.results||[];
    const owned = list.some(e=>e.offerId===offerId || e.catalogItemId===offerId || e.productId===offerId || e.id===offerId);
    res.json({owned});
  }catch(err){
    console.error(err);
    res.status(500).json({owned:false,error:'api_error'});
  }
});

// Logout
app.get('/logout',(req,res)=>{req.session.destroy(()=>res.redirect('/'));});

app.listen(PORT,()=>console.log('Servidor rodando em http://localhost:'+PORT));
