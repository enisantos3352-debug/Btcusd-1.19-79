
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Painel Automático BTC - Tempo/Preço</title>
    <style>
        body {
            background-color: #0b0e11;
            color: #eaecef;
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 10px;
        }
        .painel {
            max-width: 360px;
            background: #181a20;
            border: 1px solid #2b313a;
            border-radius: 10px;
            padding: 16px;
            margin: 0 auto;
            box-shadow: 0 4px 10px rgba(0,0,0,0.5);
        }
        .titulo {
            font-size: 15px;
            font-weight: bold;
            text-align: center;
            margin-bottom: 12px;
            color: #fcd535;
        }
        .linha {
            font-size: 13px;
            margin: 10px 0;
            display: flex;
            justify-content: space-between;
            align-items: center;
            border-bottom: 1px solid #222;
            padding-bottom: 8px;
        }
        .destaque {
            font-weight: bold;
            color: #fcd535;
        }
        .valor-tempo {
            text-align: right;
        }
        .preco-bloco {
            font-weight: bold;
        }
        .hora-bloco {
            font-size: 12px;
            color: #0ecb81;
        }
        .alerta {
            margin-top: 15px;
            padding: 12px;
            text-align: center;
            font-weight: bold;
            border-radius: 6px;
            font-size: 14px;
        }
        .comprar { background-color: #0ecb81; color: #000; }
        .vender { background-color: #f6465d; color: #fff; }
    </style>
</head>
<body>

<div class="painel">
    <div class="titulo" id="tituloTopo">PAINEL AUTOMÁTICO BTC</div>
    
    <div class="linha">
        <span>Preço Atual (Binance):</span>
        <strong id="precoBtc" class="destaque">Buscando...</strong>
    </div>

    <div class="linha">
        <span>Hora Local:</span>
        <strong id="horaLocal" style="color: #0ecb81;">--:--:--</strong>
    </div>
    
    <!-- MÁXIMA NO TOPO -->
    <div class="linha">
        <span style="color: #f6465d; font-weight: bold;">Máxima (Alvo):</span>
        <div class="valor-tempo">
            <div id="valMaxima" class="preco-bloco">--</div>
            <div id="horaMaxima" class="hora-bloco">--</div>
        </div>
    </div>

    <!-- PONTO MÉDIO NO CENTRO -->
    <div class="linha">
        <span style="color: #fcd535; font-weight: bold;">Ponto Médio (Centro):</span>
        <div class="valor-tempo">
            <div id="valMeio" class="preco-bloco">--</div>
            <div id="horaMeio" class="hora-bloco">--</div>
        </div>
    </div>

    <!-- MÍNIMA EMBAIXO -->
    <div class="linha">
        <span style="color: #0ecb81; font-weight: bold;">Mínima (Base):</span>
        <div class="valor-tempo">
            <div id="valMinima" class="preco-bloco">--</div>
            <div id="horaMinima" class="hora-bloco">--</div>
        </div>
    </div>

    <div id="caixaAlerta" class="alerta comprar">
        Analisando mercado...
    </div>
</div>

<script>
function formatarTempo(totalSegundos) {
    let horas = Math.floor(totalSegundos / 3600);
    let min = Math.floor((totalSegundos % 3600) / 60);
    let seg = Math.floor(totalSegundos % 60);
    
    if (horas > 0) {
        return `${horas}:${min < 10 ? '0' : ''}${min}:${seg < 10 ? '0' : ''}${seg}`;
    }
    return `${min}:${seg < 10 ? '0' : ''}${seg}`;
}

async function calcularPainelAutomatico() {
    try {
        let resposta = await fetch('https://api.binance.com/api/v3/ticker/price?symbol=BTCUSDT');
        let dados = await resposta.json();
        let precoDolar = parseFloat(dados.price);
        let precoFormatado = precoDolar.toFixed(2);
        
        document.getElementById("precoBtc").innerText = "$" + precoFormatado;

        // Identifica a casa do milhar atual (ex: 78, 79, 80...)
        let milhar = Math.floor(precoDolar / 1000);
        let minutosBase = milhar - 77; // 78 vira 1 min (1:18), 79 vira 2 min (1:19), etc.
        if (minutosBase < 1) minutosBase = 1;

        // Mínima dinâmica baseada na casa do milhar
        let minimaPreco = (milhar * 1000) + 118; 
        if (milhar === 79) minimaPreco = (milhar * 1000) + 119;
        let minSegundosTotal = (minutosBase * 60) + (milhar === 78 ? 18 : 19);

        // Máxima dinâmica baseada na regra do zero (ciclo dos prints)
        let maximaPreco = minimaPreco + 710.20; 
        let maxSegundosTotal = (8 * 3600) + (milhar === 78 ? 28 * 60 + 20 : 38 * 60 + 30);

        // Ponto Médio (Centro exato entre Mínima e Máxima)
        let pontoMedioPreco = (minimaPreco + maximaPreco) / 2;
        let meioSegundosTotal = (minSegundosTotal + maxSegundosTotal) / 2;

        // Atualiza os valores na tela
        document.getElementById("valMinima").innerText = "$" + minimaPreco.toFixed(2);
        document.getElementById("horaMinima").innerText = formatarTempo(minSegundosTotal);

        document.getElementById("valMeio").innerText = "$" + pontoMedioPreco.toFixed(2);
        document.getElementById("horaMeio").innerText = formatarTempo(meioSegundosTotal);

        document.getElementById("valMaxima").innerText = "$" + maximaPreco.toFixed(2);
        document.getElementById("horaMaxima").innerText = formatarTempo(maxSegundosTotal);

        // Título dinâmico informando a casa atual
        document.getElementById("tituloTopo").innerText = `BTCUSD - FAIXA ${milhar}K`;

        // Hora local
        let agora = new Date();
        document.getElementById("horaLocal").innerText = agora.toLocaleTimeString('pt-BR');

        // Sinal de Compra ou Venda pelo último dígito
        let ultimoDigito = parseInt(precoFormatado.slice(-1));
        let caixaAlerta = document.getElementById("caixaAlerta");

        if (ultimoDigito % 2 === 0) {
            caixaAlerta.className = "alerta comprar";
            caixaAlerta.innerHTML = "🟢 COMPRA (Média Alta)";
        } else {
            caixaAlerta.className = "alerta vender";
            caixaAlerta.innerHTML = "🔴 VENDA (Média Baixa)";
        }

    } catch (e) {
        document.getElementById("precoBtc").innerText = "Erro na conexão";
    }
}

// Atualiza sozinho a cada 10 segundos
setInterval(calcularPainelAutomatico, 10000);
calcularPainelAutomatico();
</script>

</body>
</html>
