import telebot
import requests
import time
import json
import ta
import pandas as pd
from datetime import datetime

# Configurações
TELEGRAM_TOKEN = "AAFgvUOL3klTnrVZeG6_CBs5qmRL_ZZM_hQ"
QUOTEX_API_KEY = "SUA_QUOTEX_API_KEY"
CHAT_ID = "7663038618"

bot = telebot.TeleBot(TELEGRAM_TOKEN)

# Configurações de operação
config = {
    "stake": 1,  # Valor por operação
    "stop_loss": 10,  # Limite de perda
    "stop_gain": 20,  # Limite de lucro
    "martingale": False,  # Usar martingale
}

# Simulação de API da Quotex (substituir por API real se disponível)
def enviar_ordem(direcao, ativo, valor):
    url = "https://api.quotex.com/v1/trade"
    data = {
        "asset": ativo,
        "amount": valor,
        "direction": direcao,  # "call" ou "put"
        "expiration": 1,
        "api_key": QUOTEX_API_KEY
    }
    response = requests.post(url, json=data)
    return response.json()

# Captura de dados de mercado (simulação, substituir por API real)
def obter_dados_mercado(ativo):
    url = f"https://api.quotex.com/v1/market/{ativo}"
    response = requests.get(url)
    dados = response.json()
    df = pd.DataFrame(dados['candles'])
    return df

# Análise técnica
def analisar_mercado(ativo):
    df = obter_dados_mercado(ativo)
    df['sma'] = ta.trend.sma_indicator(df['close'], window=14)
    df['rsi'] = ta.momentum.RSIIndicator(df['close']).rsi()
    df['upper_bb'], df['middle_bb'], df['lower_bb'] = ta.volatility.BollingerBands(df['close']).bollinger_hband(), ta.volatility.BollingerBands(df['close']).bollinger_mavg(), ta.volatility.BollingerBands(df['close']).bollinger_lband()
    return df

# Verificação de confluência
def verificar_confluencia(ativo):
    df = analisar_mercado(ativo)
    ultimo = df.iloc[-1]
    if ultimo['close'] <= ultimo['lower_bb'] and ultimo['rsi'] < 30:
        return "call"
    elif ultimo['close'] >= ultimo['upper_bb'] and ultimo['rsi'] > 70:
        return "put"
    return None

# Comandos do Telegram
@bot.message_handler(commands=['start'])
def send_welcome(message):
    bot.reply_to(message, "Bot de operações automáticas iniciado! Use /config para definir sua gestão de banca.")

@bot.message_handler(commands=['config'])
def configurar_banca(message):
    bot.reply_to(message, "Envie os valores no formato: stake,stop_loss,stop_gain,martingale (Ex: 1,10,20,False)")

@bot.message_handler(func=lambda message: "," in message.text)
def atualizar_config(message):
    try:
        stake, stop_loss, stop_gain, martingale = message.text.split(",")
        config["stake"] = float(stake)
        config["stop_loss"] = float(stop_loss)
        config["stop_gain"] = float(stop_gain)
        config["martingale"] = martingale.lower() == "true"
        bot.reply_to(message, "Configuração atualizada com sucesso!")
    except:
        bot.reply_to(message, "Formato inválido. Use: stake,stop_loss,stop_gain,martingale")

# Loop principal
def executar_bot():
    while True:
        ativo = "EURUSD"  # Pode ser configurável
        direcao = verificar_confluencia(ativo)
        if direcao:
            resultado = enviar_ordem(direcao, ativo, config['stake'])
            bot.send_message(CHAT_ID, f"Operação realizada: {direcao} no {ativo} com {config['stake']}")
        time.sleep(60)  # Aguarda 1 minuto antes da próxima análise

if __name__ == "__main__":
    bot.send_message(CHAT_ID, "Bot iniciado!")
    executar_bot()
    bot.polling()

