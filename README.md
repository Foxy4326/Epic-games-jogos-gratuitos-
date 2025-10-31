<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>🎮 Jogos Gratuitos da Epic Games</title>
<script src="https://cdn.tailwindcss.com"></script>
<style>
  body { background: #0f172a; color: white; font-family: "Poppins", sans-serif; min-height: 100vh; margin:0; padding:0; }
  header { text-align:center; padding:15px 0; }
  h1 { font-size:2.2rem; color:#00aaff; margin:0; }
  .clock { font-size:1rem; color:#ffcc00; margin-top:5px; }
  .games-grid { display:grid; grid-template-columns:repeat(auto-fit, minmax(260px,1fr)); gap:20px; max-width:1000px; margin:30px auto; padding:0 20px; }
  .game-card { background:#1c1c1c; border-radius:12px; overflow:hidden; box-shadow:0 4px 10px rgba(0,0,0,0.4); transition:transform 0.3s; }
  .game-card:hover { transform: translateY(-6px); }
  .game-image { width:100%; height:160px; object-fit:cover; background:#333; }
  .game-info { padding:15px; }
  .price { color:#00ff99; font-weight:bold; }
  .original { text-decoration:line-through; color:#999; }
  .btn { background:#0074e4; border:none; padding:10px 15px; border-radius:8px; color:white; cursor:pointer; transition:0.3s; font-weight:bold; display:block; width:100%; text-align:center; margin-top:10px; }
  .btn:hover { background:#005bb5; }
  .resgatado { color:#ffcc00; font-weight:bold; margin-left:5px; }
  .login-status { margin-top:10px; color:#00ff99; font-weight:bold; }
</style>
</head>
<body>

<header>
  <h1>🎁 Jogos Gratuitos da Epic Games</h1>
  <div class="clock" id="brClock">Carregando horário...</div>
  <button class="btn" id="loginBtn">🔑 Login com Epic Games</button>
  <div class="login-status" id="loginStatus">Não logado</div>
</header>

<div id="gamesGrid" class="games-grid"></div>

<script>
const apiURL = "https://corsproxy.io/?" + encodeURIComponent("https://store-site-backend-static.ak.epicgames.com/freeGamesPromotions?locale=pt-BR");
const gamesGrid = document.getElementById("gamesGrid");
const brClock = document.getElementById("brClock");
const loginBtn = document.getElementById("loginBtn");
const loginStatus = document.getElementById("loginStatus");

let loggedIn = false;
let userName = "";

// Simula login com Epic Games (sem backend real)
loginBtn.addEventListener("click", () => {
  loggedIn = true;
  userName = "JogadorEpic"; // Simulação
  loginStatus.textContent = `Logado como: ${userName}`;
  loginBtn.style.display = "none";
  fetchFreeGames();
});

function updateBRClock() {
  const now = new Date();
  const brTime = new Date(now.toLocaleString("en-US", { timeZone: "America/Sao_Paulo" }));
  const hours = String(brTime.getHours()).padStart(2,'0');
  const minutes = String(brTime.getMinutes()).padStart(2,'0');
  const seconds = String(brTime.getSeconds()).padStart(2,'0');
  brClock.textContent = `🕒 Horário de Brasília: ${hours}:${minutes}:${seconds}`;
}
setInterval(updateBRClock, 1000);
updateBRClock();

async function fetchFreeGames() {
  gamesGrid.innerHTML = "<p class='loading'>Carregando jogos gratuitos...</p>";
  try {
    const res = await fetch(apiURL);
    if (!res.ok) throw new Error("Erro ao acessar API");

    const data = await res.json();
    const elements = data?.data?.Catalog?.searchStore?.elements || [];

    // Apenas jogos gratuitos agora
    const freeGames = elements.filter(game =>
      game.price?.totalPrice?.discountPrice === 0 &&
      game.promotions?.promotionalOffers?.length > 0
    );

    renderGames(freeGames);

  } catch (err) {
    console.error(err);
    gamesGrid.innerHTML = "<p class='loading'>❌ Erro ao carregar jogos gratuitos.</p>";
  }
}

function renderGames(games) {
  gamesGrid.innerHTML = "";
  if (games.length === 0) {
    gamesGrid.innerHTML = "<p class='loading'>Nenhum jogo gratuito disponível 😔</p>";
    return;
  }

  games.forEach(game => {
    const image = game.keyImages?.[0]?.url || "https://via.placeholder.com/600x400?text=Sem+Capa";
    const originalPrice = (game.price?.totalPrice?.originalPrice / 100).toFixed(2);
    let pageSlug = game.catalogNs?.mappings?.[0]?.pageSlug || game.productSlug || "";
    if (!pageSlug.startsWith("p/")) pageSlug = "p/" + pageSlug;
    const gameUrl = "https://store.epicgames.com/pt-BR/" + pageSlug.replace(/^\/+/, "");

    const card = document.createElement("div");
    card.className = "game-card";

    // Se usuário está logado, simula se resgatou
    const resgatadoText = loggedIn ? `<span class="resgatado">Resgatado ✅</span>` : "";

    card.innerHTML = `
      <img src="${image}" alt="${game.title}" class="game-image">
      <div class="game-info">
        <h3>${game.title}</h3>
        <p class="original">De R$${originalPrice}</p>
        <p class="price">💥 GRÁTIS! ${resgatadoText}</p>
        <a href="${gameUrl}" target="_blank" rel="noopener">
          <button class="btn mt-2">Resgatar</button>
        </a>
      </div>`;
    gamesGrid.appendChild(card);
  });
}

// Atualiza lista a cada 10 minutos
setInterval(fetchFreeGames, 600000);

</script>

</body>
</html>
