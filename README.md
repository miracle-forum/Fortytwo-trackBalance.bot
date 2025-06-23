# Fortytwo-trackBalance.bot
python-telegram-bot==20.3
web3
nest_asyncio
# 🛰️ FortyTwo Track Bot (FOR Balance Checker)

A Telegram bot to check your FOR token balance on the Monad testnet.

## Features
- Set your EVM wallet address
- Instantly check your 42T token balance
- Uses Monad testnet and 42T token smart contract

## Telegram Bot
👉 [Click here to try it](https://t.me/FortyTwo_trackbot)

## Setup

```bash
git clone https://github.com/MIRACLE6.git
cd monad-telegram-bot
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python main.py

---



import logging
import asyncio
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes
from web3 import Web3

# === CONFIG ===
BOT_TOKEN = '7670074494:AAEBo7gbIDbX4KTRWNQJM1_RA2cnVttMK4Q'
MONAD_RPC = 'https://testnet-rpc.monad.xyz'
FOR_TOKEN_ADDRESS = '0x22A3d96424Df6f04d02477cB5ba571BBf615F47E'

# === SETUP ===
w3 = Web3(Web3.HTTPProvider(MONAD_RPC))
FOR_TOKEN = w3.to_checksum_address(FOR_TOKEN_ADDRESS)
tracked_wallets = set()

ERC20_ABI = [
    {
        "constant": True,
        "inputs": [{"name": "_owner", "type": "address"}],
        "name": "balanceOf",
        "outputs": [{"name": "balance", "type": "uint256"}],
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "decimals",
        "outputs": [{"name": "", "type": "uint8"}],
        "type": "function"
    },
]

token_contract = w3.eth.contract(address=FOR_TOKEN, abi=ERC20_ABI)

def get_balance(wallet):
    try:
        checksum_wallet = w3.to_checksum_address(wallet)
        balance = token_contract.functions.balanceOf(checksum_wallet).call()
        decimals = token_contract.functions.decimals().call()
        return balance / (10 ** decimals)
    except Exception as e:
        logging.error(f"Error getting balance: {e}")
        return None

# === TELEGRAM BOT COMMANDS ===

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("👋 Welcome! Use /addwallet <address> to start tracking.")

async def addwallet(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("Usage: /addwallet <wallet_address>")
        return

    wallet = context.args[0]
    if not w3.is_address(wallet):
        await update.message.reply_text("❌ Invalid address.")
        return

    tracked_wallets.add(w3.to_checksum_address(wallet))
    await update.message.reply_text("✅ Wallet added successfully!")

async def listwallets(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not tracked_wallets:
        await update.message.reply_text("📭 No wallets tracked yet.")
        return

    response = "📋 Tracked wallets and FOR balances:\n"
    for wallet in tracked_wallets:
        bal = get_balance(wallet)
        bal_str = f"{bal:.4f} FOR" if bal is not None else "Error"
        response += f"{wallet} - {bal_str}\n"

    await update.message.reply_text(response)

async def totalfor(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not tracked_wallets:
        await update.message.reply_text("📭 No wallets tracked yet.")
        return

    response = "📊 Total FOR received per wallet:\n"
    for wallet in tracked_wallets:
        bal = get_balance(wallet)
        bal_str = f"{bal:.4f}" if bal is not None else "0.0000"
        response += f"{wallet} - {bal_str} FOR\n"

    await update.message.reply_text(response)

# === MAIN ===
async def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("addwallet", addwallet))
    app.add_handler(CommandHandler("listwallets", listwallets))
    app.add_handler(CommandHandler("totalfor", totalfor))

    print("✅ Bot is running...")
    await app.run_polling()

if __name__ == '__main__':
    import nest_asyncio
    nest_asyncio.apply()
    asyncio.get_event_loop().run_until_complete(main())
