<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>🎮 GameAlerts — Jogos Pagos Grátis</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            background: linear-gradient(135deg, #0f172a, #1e293b);
            color: white;
            font-family: "Poppins", sans-serif;
            min-height: 100vh;
            margin: 0;
            padding: 0;
        }
        header { text-align: center; padding: 20px 0; }
        h1 { font-size: 2rem; color: #00aaff; font-weight: 700; }
        .subtitle { color: #ccc; margin-top: 5px; }
        #brTime { font-weight: bold; margin-top: 5px; color: #ffcc00; }
        nav { text-align: center; margin-top: 10px; }
        nav button {
            background: #1e40af; color: white; border: none; padding: 10px 15px;
            margin: 5px; border-radius: 8px; cursor: pointer; font-weight: bold;
            transition: 0.3s;
        }
        nav button.active { background: #00aaff; }
        nav button:hover { background: #2563eb; }
        .games-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
            gap: 20px; max-width: 1000px; margin: 20px auto; padding: 0 20px;
        }
        .game-card {
            background: #1c1c1c; border-radius: 12px; overflow: hidden;
            box-shadow: 0 4px 10px rgba(0,0,0,0.4);
            transition: transform 0.3s; position: relative;
        }
        .game-card:hover { transform: translateY(-6px); }
        .game-image { width: 100%; height: 160px; object-fit: cover; background: #333; }
        .game-info { padding: 15px; }
        .price { color: #00ff99; font-weight: bold; }
        .original { text-decoration: line-through; color: #999; }
        .btn {
            background: #0074e4; border: none; padding: 10px 15px; border-radius: 8px;
            color: white; cursor: pointer; transition: 0.3s; font-weight: bold;
            display: block; width: 100%; text-align: center;
        }
        .btn:hover { background: #005bb5; }
        .bug-btn { background: #dc2626; margin-top: 10px; }
        .bug-btn:hover { background: #b91c1c; }
        footer { text-align: center; margin-top: 30px; color: #aaa; padding-bottom: 30px; }
        .loading { text-align: center; font-size: 1.2rem; color: #bbb; margin-top: 40px; }
        .notification {
            position: fixed; top: 20px; right: 20px; background: #222; color: white;
            padding: 15px 20px; border-left: 4px solid #00ff99; border-radius: 6px;
            box-shadow: 0 4px 10px rgba(0,0,0,0.4); transform: translateX(300px);
            transition: transform 0.5s ease; z-index: 9999;
        }
        .notification.show { transform: translateX(0); }
        .countdown { font-weight: bold; margin-top: 5px; color: #ffcc00; }
        .soon-badge {
            position: absolute;
            top: 10px;
            right: 10px;
            background: #ff9900;
            color: #000;
            padding: 5px 10px;
            border-radius: 4px;
            font-weight: bold;
            font-size: 0.8rem;
        }
        .error-message {
            background: #dc2626;
            color: white;
            padding: 10px;
            border-radius: 8px;
            margin: 10px auto;
            max-width: 600px;
            text-align: center;
        }
    </style>
</head>
<body>
    <header>
        <h1>🎁 GameAlerts</h1>
        <p class="subtitle">Jogos pagos que estão ou estarão GRÁTIS na Epic Games Store</p>
        <div id="brTime">Carregando hora...</div>
    </header>

    <nav>
        <button id="tabAtivos" class="active">🎮 Grátis Agora</button>
        <button id="tabFuturos">⏳ Em Breve</button>
    </nav>

    <div id="status" class="text-center mb-4 text-sm text-gray-400">Carregando lista de jogos...</div>
    <div id="errorContainer"></div>
    <div class="games-grid" id="gamesGrid"></div>

    <div class="text-center mt-6">
        <button class="btn" id="refreshBtn">🔄 Atualizar Agora</button>
        <button class="btn bug-btn" id="reportBtn">🐞 Reportar Bug</button>
    </div>

    <footer>
        <p>Dados em tempo real da
            <a href="https://store.epicgames.com/pt-BR/free-games" style="color:#00aaff;" target="_blank">Epic Games Store</a>
        </p>
    </footer>

    <div class="notification" id="notification">Novo jogo gratuito detectado!</div>

    <script>
        // Configuração e variáveis globais
        const apiURLs = [
            "https://api.allorigins.win/raw?url=https://store-site-backend-static.ak.epicgames.com/freeGamesPromotions?locale=pt-BR",
            "https://store-site-backend-static.ak.epicgames.com/freeGamesPromotions?locale=pt-BR",
            "https://corsproxy.io/?https://store-site-backend-static.ak.epicgames.com/freeGamesPromotions?locale=pt-BR"
        ];
        
        const gamesGrid = document.getElementById("gamesGrid");
        const statusText = document.getElementById("status");
        const errorContainer = document.getElementById("errorContainer");
        const refreshBtn = document.getElementById("refreshBtn");
        const reportBtn = document.getElementById("reportBtn");
        const notification = document.getElementById("notification");
        const tabAtivos = document.getElementById("tabAtivos");
        const tabFuturos = document.getElementById("tabFuturos");
        const brTimeEl = document.getElementById("brTime");
        let allGames = { ativos: [], futuros: [] };
        let countdownIntervals = [];

        // Atualizar hora do Brasil
        function updateBrazilTime() {
            const now = new Date();
            brTimeEl.textContent = "🕒 Hora Brasil: " + now.toLocaleString("pt-BR", { timeZone: "America/Sao_Paulo" });
        }
        setInterval(updateBrazilTime, 1000);
        updateBrazilTime();

        // Função para mostrar erro
        function showError(message) {
            errorContainer.innerHTML = `<div class="error-message">${message}</div>`;
        }

        // Função para limpar erro
        function clearError() {
            errorContainer.innerHTML = '';
        }

        // Tentar diferentes APIs até uma funcionar
        async function tryFetch(url) {
            try {
                const response = await fetch(url);
                if (!response.ok) throw new Error(`HTTP ${response.status}`);
                return await response.json();
            } catch (error) {
                console.log(`Falha na URL: ${url}`, error);
                throw error;
            }
        }

        // Buscar jogos gratuitos da API
        async function fetchFreeGames() {
            statusText.textContent = "🔍 Atualizando jogos gratuitos...";
            clearError();
            
            let data = null;
            let lastError = null;

            // Tentar cada URL até uma funcionar
            for (const url of apiURLs) {
                try {
                    data = await tryFetch(url);
                    break; // Se funcionou, para de tentar
                } catch (error) {
                    lastError = error;
                    continue; // Continua para a próxima URL
                }
            }

            if (!data) {
                const errorMsg = "❌ Não foi possível conectar com a Epic Games. Tentando com dados de exemplo...";
                showError(errorMsg);
                statusText.textContent = errorMsg;
                
                // Carregar dados de exemplo como fallback
                loadSampleData();
                return;
            }

            try {
                const elements = data?.data?.Catalog?.searchStore?.elements || [];

                if (elements.length === 0) {
                    throw new Error("Nenhum jogo encontrado na resposta da API");
                }

                // Filtrar jogos atualmente gratuitos
                const ativos = elements.filter(g =>
                    g.price?.totalPrice?.discountPrice === 0 && 
                    g.promotions?.promotionalOffers?.length > 0
                );

                // Filtrar jogos que serão gratuitos no futuro
                const futuros = elements.filter(g =>
                    g.promotions?.upcomingPromotionalOffers?.length > 0 &&
                    g.price?.totalPrice?.discountPrice !== 0
                );

                allGames = { ativos, futuros };
                renderGames(ativos, true);
                statusText.textContent = `✅ Atualizado com sucesso! ${ativos.length} grátis agora, ${futuros.length} em breve`;
                
            } catch (e) {
                console.error("Erro ao processar dados:", e);
                showError("Erro ao processar dados da API. Carregando dados de exemplo...");
                loadSampleData();
            }
        }

        // Dados de exemplo para quando a API falhar
        function loadSampleData() {
            const sampleGames = {
                ativos: [
                    {
                        title: "Fall Guys",
                        price: { totalPrice: { originalPrice: 7999, discountPrice: 0 } },
                        promotions: { promotionalOffers: [{ promotionalOffers: [{ endDate: new Date(Date.now() + 86400000).toISOString() }] }] },
                        keyImages: [{ url: "https://cdn1.epicgames.com/salesEvent/salesEvent/EGS_FallGuys_Mediatonic_S1_2560x1440-5b5f6e5dcf5a7b2c6d7d5a5a5a5a5a5a" }],
                        productSlug: "fall-guys"
                    }
                ],
                futuros: [
                    {
                        title: "Cyberpunk 2077",
                        price: { totalPrice: { originalPrice: 19999, discountPrice: 19999 } },
                        promotions: { upcomingPromotionalOffers: [{ promotionalOffers: [{ startDate: new Date(Date.now() + 172800000).toISOString() }] }] },
                        keyImages: [{ url: "https://cdn1.epicgames.com/offer/0a0c6a6e-8c0a-4c0a-8c0a-4c0a6a6e8c0a/EGS_Cyberpunk2077_CDPROJEKTRED_S1_2560x1440-5b5f6e5dcf5a7b2c6d7d5a5a5a5a5a5a" }],
                        productSlug: "cyberpunk-2077"
                    }
                ]
            };

            allGames = sampleGames;
            renderGames(sampleGames.ativos, true);
            statusText.textContent = "⚠️ Modo offline - Dados de exemplo carregados";
        }

        // Renderizar jogos na tela
        function renderGames(games, isAtivos) {
            gamesGrid.innerHTML = "";
            countdownIntervals.forEach(clearInterval);
            countdownIntervals = [];
            
            if (!games.length) {
                gamesGrid.innerHTML = "<p class='loading'>Nenhum jogo disponível 😔</p>";
                return;
            }

            games.forEach(game => {
                const img = game.keyImages?.[0]?.url || "https://via.placeholder.com/300x200/1e293b/ffffff?text=Sem+Imagem";
                const original = (game.price?.totalPrice?.originalPrice / 100).toFixed(2);
                const promo = isAtivos ? 
                    game.promotions?.promotionalOffers?.[0]?.promotionalOffers?.[0] :
                    game.promotions?.upcomingPromotionalOffers?.[0]?.promotionalOffers?.[0];
                const end = promo?.endDate ? new Date(promo.endDate) : null;
                const start = promo?.startDate ? new Date(promo.startDate) : null;
                const url = game.productSlug ? 
                    `https://store.epicgames.com/p/${game.productSlug}` : 
                    "https://store.epicgames.com/pt-BR/free-games";

                const card = document.createElement("div");
                card.className = "game-card";
                card.innerHTML = `
                    <img src="${img}" class="game-image" onerror="this.src='https://via.placeholder.com/300x200/1e293b/ffffff?text=Imagem+Não+Carregada'">
                    ${!isAtivos ? '<div class="soon-badge">EM BREVE</div>' : ''}
                    <div class="game-info">
                        <h3 class="font-bold text-lg mb-2">${game.title}</h3>
                        <p class="original text-sm mb-1">De R$${original}</p>
                        <p class="price text-lg mb-2">💥 GRÁTIS ${isAtivos ? 'AGORA' : 'EM BREVE'}!</p>
                        <div class="countdown text-sm mb-3"></div>
                        <a href="${url}" target="_blank"><button class="btn">${isAtivos ? 'Resgatar Agora' : 'Ver na Loja'}</button></a>
                    </div>`;
                gamesGrid.appendChild(card);

                const countdown = card.querySelector(".countdown");
                const interval = setInterval(() => {
                    const now = new Date();
                    const diff = isAtivos && end ? end - now : !isAtivos && start ? start - now : 0;
                    
                    if (diff <= 0) {
                        countdown.textContent = isAtivos ? "⏰ Promoção encerrada!" : "🎮 Já disponível!";
                        clearInterval(interval);
                    } else {
                        const d = Math.floor(diff / 86400000);
                        const h = Math.floor((diff % 86400000) / 3600000);
                        const m = Math.floor((diff % 3600000) / 60000);
                        countdown.textContent = isAtivos ? 
                            `${d}d ${h}h ${m}m restantes` : 
                            `Inicia em ${d}d ${h}h ${m}m`;
                    }
                }, 1000);
                countdownIntervals.push(interval);
            });
        }

        // Event Listeners
        tabAtivos.onclick = () => { 
            tabAtivos.classList.add("active"); 
            tabFuturos.classList.remove("active"); 
            renderGames(allGames.ativos, true); 
        };
        
        tabFuturos.onclick = () => { 
            tabFuturos.classList.add("active"); 
            tabAtivos.classList.remove("active"); 
            renderGames(allGames.futuros, false); 
        };
        
        refreshBtn.onclick = fetchFreeGames;
        reportBtn.onclick = () => window.location.href = "mailto:reisjuvenira468@gmail.com?subject=🐞 Reporte de Bug - GameAlerts&body=Descreva o bug que encontrou:";

        // Inicializar aplicação
        fetchFreeGames();
    </script>
</body>
</html>