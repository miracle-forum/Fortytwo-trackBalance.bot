# 🤖 FortyTwo Track Bot (FOR Balance Checker)

A Telegram bot to track **FOR token** balance on the **Monad testnet**, with real-time alerts when balances change.
try it now https://t.me/FortyTwo_trackbot
---

## 🚀 Features

- 🧾 Monitor FOR token balance in real time
- 👤 One wallet per Telegram user
- 🔔 Automatic notifications on balance updates
- ⚙️ Built with `python-telegram-bot` and `web3.py`

---

## ⚙️ Installation

1. Clone the repository:
```bash
git clone https://github.com/miracle-forum/Fortytwo-trackBalance.bot.git
cd Fortytwo-trackBalance.bot

pip install -r requirements.txt

python bot.py
🛠️ Bot Commands
Command	Description
/start	Greet the user
/addwallet	Set the wallet to track
/totalfor	Show current FOR token balance

📦 Requirements
Python 3.9+

python-telegram-bot

web3.py

nest_asyncio

📄 License
MIT License © 2025 miracle-forum





---


import logging
import asyncio
import json
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes
from web3 import Web3

# === CONFIG ===
BOT_TOKEN = '7670074494:AAEBo7gbIDbX4KTRWNQJM1_RA2cnVttMK4Q'
MONAD_RPC = 'https://testnet-rpc.monad.xyz'
FOR_TOKEN_ADDRESS = '0x22A3d96424Df6f04d02477cB5ba571BBf615F47E'
WALLETS_FILE = 'wallets.json'
BALANCE_CACHE_FILE = 'balances.json'

# === SETUP ===
w3 = Web3(Web3.HTTPProvider(MONAD_RPC))
FOR_TOKEN = w3.to_checksum_address(FOR_TOKEN_ADDRESS)

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

# === DATABASE ===
def load_json(path):
    try:
        with open(path, 'r') as f:
            return json.load(f)
    except:
        return {}

def save_json(path, data):
    with open(path, 'w') as f:
        json.dump(data, f, indent=2)

tracked_wallets = load_json(WALLETS_FILE)
balance_cache = load_json(BALANCE_CACHE_FILE)

# === BALANCE CHECK ===
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
    await update.message.reply_text("👋 Welcome! Use /addwallet <address> to start tracking your wallet.")

async def addwallet(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("Usage: /addwallet <wallet_address>")
        return

    wallet = context.args[0]
    if not w3.is_address(wallet):
        await update.message.reply_text("❌ Invalid address.")
        return

    user_id = str(update.effective_user.id)
    tracked_wallets[user_id] = w3.to_checksum_address(wallet)
    save_json(WALLETS_FILE, tracked_wallets)
    await update.message.reply_text(f"✅ Wallet set for tracking: {wallet}")

async def totalfor(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    wallet = tracked_wallets.get(user_id)

    if not wallet:
        await update.message.reply_text("📭 You haven't added a wallet yet. Use /addwallet.")
        return

    bal = get_balance(wallet)
    bal_str = f"{bal:.4f}" if bal is not None else "0.0000"
    await update.message.reply_text(f"📊 Wallet: {wallet}\n💰 Total FOR: {bal_str} FOR")

# === BACKGROUND TASK FOR NOTIFICATIONS ===
async def monitor_wallets(application):
    while True:
        for user_id, wallet in tracked_wallets.items():
            new_balance = get_balance(wallet)
            if new_balance is None:
                continue

            old_balance = balance_cache.get(user_id)
            if old_balance is None or abs(new_balance - old_balance) > 0.0001:
                balance_cache[user_id] = new_balance
                save_json(BALANCE_CACHE_FILE, balance_cache)
                try:
                    await application.bot.send_message(chat_id=int(user_id), text=f"🔔 Balance update for {wallet}:\nNew Balance: {new_balance:.4f} FOR")
                except Exception as e:
                    logging.error(f"Error sending message to {user_id}: {e}")

        await asyncio.sleep(60)  # Check every 60 seconds

# === MAIN ===
async def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("addwallet", addwallet))
    app.add_handler(CommandHandler("totalfor", totalfor))

    print("✅ Bot is running...")
    asyncio.create_task(monitor_wallets(app))
    await app.run_polling()

if __name__ == '__main__':
    import nest_asyncio
    nest_asyncio.apply()
    asyncio.get_event_loop().run_until_complete(main())

