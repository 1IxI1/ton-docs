# Tutorial: Simple bot with balance

## 👋 Hi there!
In this article, we'll create a simple Telegram bot for accepting payments in TON coins.

The bot will look like this:

<img src="/img/tutorials/bot1.png" alt="drawing"/>

Sources are available on Github:
https://github.com/Gusarich/ton-bot-example

## 📖 What you'll learn
You'll learn how to:
 - create a Telegram bot in Python3 using Aiogram
 - work with SQLITE databases
 - work with public TON API

## ✍️ What do you need to begin
Install [Python](https://www.python.org/) if you haven't yet.

Also you need these PyPi libraries:
 - aiogram
 - requests

You can install them with one command in terminal:
```bash
pip install aiogram==2.21 requests
```

## 🚀 Let's get started!
Create a directory for our bot and 4 files in it:
 - `bot.py` - program to run Telegram bot
 - `config.py` - config file
 - `db.py` - module to interact with sqlite3 database
 - `ton.py` - module to handle payments in TON

Directory should look like this:
```
my_bot
├── bot.py
├── config.py
├── db.py
└── ton.py
```

Now let's begin writing code!

## Config
Let's start with the `config.py` because it is the smallest one. We just need to set a few parameters in it.

**config.py:**
```python
BOT_TOKEN = 'YOUR BOT TOKEN'
DEPOSIT_ADDRESS = 'YOUR DEPOSIT ADDRESS'
API_KEY = 'YOUR API KEY'
RUN_IN_MAINNET = True  # Switch True/False to change mainnet to testnet

if RUN_IN_MAINNET:
    API_BASE_URL = 'https://toncenter.com'
else:
    API_BASE_URL = 'https://testnet.toncenter.com'
```

Here you need to fill your values in first 3 lines.
 - `BOT_TOKEN` is your Telegram Bot token which you can get after [creating a bot](https://t.me/BotFather).
 - `DEPOSIT_ADDRESS` is your project's wallet address which will accept all the payments. You can just create a new TON wallet and copy it's address.
 - `API_KEY` is your API key from Toncenter which you can get in [this bot](https://t.me/tonapibot).

Also you can decide where will your bot work: in testnet or mainnet (4th line)

That's all for Config file so we can move forward!

## Database
Now let's edit the `db.py` file that will work with database of our bot.

Import the sqlite3 library:
```python
import sqlite3
```

Initialize database connection and cursor (you can choose any filename instead of `db.sqlite`):
```python
con = sqlite3.connect('db.sqlite')
cur = con.cursor()
```

To store information about users (their balances in our case), create table "Users" with User ID and balance rows:
```python
cur.execute('''CREATE TABLE IF NOT EXISTS Users (
                uid INTEGER,
                balance INTEGER
            )''')
con.commit()
```

Now we need to declare a few functions to work with database.

`add_user` function will be used to insert new users into database.
```python
def add_user(uid):
    # new user always have balance = 0
    cur.execute(f'INSERT INTO Users VALUES ({uid}, 0)')
    con.commit()
```

`check_user` function will be used to check if user is in database or not.
```python
def check_user(uid):
    cur.execute(f'SELECT * FROM Users WHERE uid = {uid}')
    user = cur.fetchone()
    if user:
        return True
    return False
```

`add_balance` function will be used to increase user balance.
```python
def add_balance(uid, amount):
    cur.execute(f'UPDATE Users SET balance = balance + {amount} WHERE uid = {uid}')
    con.commit()
```

`get_balance` function will be used to retreive user balance.
```python
def get_balance(uid):
    cur.execute(f'SELECT balance FROM Users WHERE uid = {uid}')
    balance = cur.fetchone()[0]
    return balance
```

And that's all for `db.py` file!

Now we can use these 4 functions in our other components of bot to work with database.

## Toncenter API
In `ton.py` file we'll declare a function that will process all new deposits, increase user balances and notify them.

### getTransactions method

We'll use the Toncenter API, their docs are available here:
https://toncenter.com/api/v2/

We need the [getTransactions](https://toncenter.com/api/v2/#/accounts/get_transactions_getTransactions_get) method to get information about latest transactions of a given account.

Let's have a look at what does this method take as input parameters and what does it return.

There is only one mandatory input field `address`, but we also need the `limit` field to specify how many transactions do we want to get in return.

Now let's try to run this method on [Toncenter website](https://toncenter.com/api/v2/#/accounts/get_transactions_getTransactions_get) with any existing wallet address to understand what should we get from the output.

```json
{
  "ok": true,
  "result": [
    {
      ...
    },
    {
      ...
    }
  ]
}
```

Okay, so `ok` field is set to `true` when everything is good, and we have an array `result` with list of `limit` latest transactions. Now let's look at one single transaction:

```json
{
    "@type": "raw.transaction",
    "utime": 1666648337,
    "data": "...",
    "transaction_id": {
        "@type": "internal.transactionId",
        "lt": "32294193000003",
        "hash": "ez3LKZq4KCNNLRU/G4YbUweM74D9xg/tWK0NyfuNcxA="
    },
    "fee": "105608",
    "storage_fee": "5608",
    "other_fee": "100000",
    "in_msg": {
        "@type": "raw.message",
        "source": "EQBIhPuWmjT7fP-VomuTWseE8JNWv2q7QYfsVQ1IZwnMk8wL",
        "destination": "EQBKgXCNLPexWhs2L79kiARR1phGH1LwXxRbNsCFF9doc2lN",
        "value": "100000000",
        "fwd_fee": "666672",
        "ihr_fee": "0",
        "created_lt": "32294193000002",
        "body_hash": "tDJM2A4YFee5edKRfQWLML5XIJtb5FLq0jFvDXpv0xI=",
        "msg_data": {
            "@type": "msg.dataText",
            "text": "SGVsbG8sIHdvcmxkIQ=="
        },
        "message": "Hello, world!"
    },
    "out_msgs": []
}
```

We can see that information that can help us identify the exact transaction is stored in `transaction_id` field. We'll need the `lt` field from it to understand which transaction happened earilier and which later.

And information about coins transfer is in `in_msg` field. We'll need `value` and `message` from it.

Now we're ready to create a payment handler.

### Sending API requests from code

Let's begin with importing required libraries and our two previous files `config.py` and `db.py`:
```python
import requests
import asyncio

# Aiogram
from aiogram import Bot
from aiogram.types import ParseMode

# We also need config and database here
import config
import db
```

Let's think about how payment processing can be implemented.

We can call the API every few seconds and check if there were any new transactions to our wallet address.

For that we need to know what was the last processed transaction. Simplest approach would be just to save info about that transaction in some file and update it every time we process new transaction.

What information about transaction will we store in file? Actually, we only need to store the `lt` value - logical time.
With that value we'll be able to understand what transactions do we need to process.

So we need to define a new async function, let's call it `start`. Why do this function need to be asynchronous? That is because Aiogram library for Telegram bots is also asynchronous and it'll be easier to work with async functions later.

This is how our `start` function should look like:
```python
async def start():
    try:
        # Try to load last_lt from file
        with open('last_lt.txt', 'r') as f:
            last_lt = int(f.read())
    except FileNotFoundError:
        # If file not found, set last_lt to 0
        last_lt = 0

    # We need the Bot instance here to send deposit notifications to users
    bot = Bot(token=config.BOT_TOKEN)

    while True:
        # Here we will call API every few seconds and fetch new transactions.
        ...
```

Now let's write the body of while loop. We need to call Toncenter API there every few seconds.
```python
while True:
    # 2 Seconds delay between checks
    await asyncio.sleep(2)

    # API call to Toncenter that returns last 100 transactions of our wallet
    resp = requests.get(f'{config.API_BASE_URL}/api/v2/getTransactions?'
                        f'address={config.DEPOSIT_ADDRESS}&limit=100&'
                        f'archival=true&api_key={config.API_KEY}').json()

    # If call was not successful, try again
    if not resp['ok']:
        continue
    
    ...
```

After the call with `requests.get` we have a variable `resp` that contain response from API. `resp` is an object and `resp['result']` is a list with last 100 transactions for our address.

Now let's just iterate over these transactions and find the new ones.

```python
while True:
    ...

    # Iterating over transactions
    for tx in resp['result']:
        # LT is Logical Time and Hash is hash of our transaction
        lt, hash = int(tx['transaction_id']['lt']), tx['transaction_id']['hash']

        # If this transaction's logical time is lower than our last_lt,
        # we already processed it, so skip it

        if lt <= last_lt:
            continue
        
        # at this moment, `tx` is some new transaction that we didn't process yet
        ...
```

How do we process a new transaction? We need to:
 - understand which user sent it
 - increase that user's balance
 - notify the user about their deposit

Here is the code that will do all of that:

```python
while True:
    ...

    for tx in resp['result']:
        ...
        # at this moment, `tx` is some new transaction that we didn't process yet

        value = int(tx['in_msg']['value'])
        if value > 0:
            uid = tx['in_msg']['message']

            if not uid.isdigit():
                continue

            uid = int(uid)

            if not db.check_user(uid):
                continue

            db.add_balance(uid, value)

            await bot.send_message(uid, 'Deposit confirmed!\n'
                                    f'*+{value / 1e9:.2f} TON*',
                                    parse_mode=ParseMode.MARKDOWN)
```

Let's have a look at it and understand what it does.

All the information about coins transfer is in `tx['in_msg']`. We just need 'value' and 'message' fields from it.

First of all, we check if value is greater than zero and only continue if it is.

Then we except the transfer to have comment ( `tx['in_msg']['message']` ) to be a user ID from our bot, so we check if it is a valid number and if that UID exists in our database.

After these simple checks, we have a variable `value` with deposit amount, and variable `uid` with id of user that made this deposit. So we can just add balance to their account and send a notification message.

Also note that value in in nanoTONs by default, so we need to divide it by 1 billion. We do that in line with notification:
`{value / 1e9:.2f}`
Here we divide the value by `1e9` (1 billion) and leave only 2 digits after the decimal point to show it to user in friendly format.

Great! The program is now can process new transactions and notify users about deposits. But we should not forget about storing last_lt that we are used before. We must update the last_lt because newer transaction was processed.

It's simple:
```python
while True:
    ...
    for tx in resp['result']:
        ...
        # we have processed this tx

        # lt variable here contain LT of last processed transaction
        last_lt = lt
        with open('last_lt.txt', 'w') as f:
            f.write(str(last_lt))
```

And that's all for the `ton.py` file!
Our bot is now 3/4 done, we only need to create a user interface with a few buttons in bot itself.

## Telegram Bot

### Initialization

Open the `bot.py` file and import all the modules that we need:
```python
# Logging module
import logging

# Aiogram imports
from aiogram import Bot, Dispatcher, types
from aiogram.dispatcher.filters import Text
from aiogram.types import ParseMode, ReplyKeyboardMarkup, KeyboardButton, \
                          InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.utils import executor

# Local modules to work with Database and Ton network
import config
import ton
import db
```

Let's set up logging for our program so that we can see what happens later for debug:
```python
logging.basicConfig(level=logging.INFO)
```

Now we need to initialize the bot object and it's dispatcher with Aiogram:
```python
bot = Bot(token=config.BOT_TOKEN)
dp = Dispatcher(bot)
```

Here we use `BOT_TOKEN` from our config that we made in the beginning of tutorial.

We initialized the bot, but it's still empty. We must add some fucntions for interaction with user.

### Message handlers

#### /start Command

Let's begin with `/start` and `/help` commands handler. This function will be called when user does launch the bot for the first time, restarting it or using the `/help` command.

```python
@dp.message_handler(commands=['start', 'help'])
async def welcome_handler(message: types.Message):
    uid = message.from_user.id  # Not neccessary, just to make code shorter

    # If user doesn't exist in database, insert it
    if not db.check_user(uid):
        db.add_user(uid)

    # Keyboard with two main buttons: Deposit and Balance
    keyboard = ReplyKeyboardMarkup(resize_keyboard=True)
    keyboard.row(KeyboardButton('Deposit'))
    keyboard.row(KeyboardButton('Balance'))

    # Send welcome text and include the keyboard
    await message.answer('Hi!\nI am example bot '
                         'made for [this article](/develop/dapps/payment-processing/accept-payments-in-a-telegram-bot-2).\n'
                         'My goal is to show how simple it is to receive '
                         'payments in Toncoin with Python.\n\n'
                         'Use keyboard to test my functionality.',
                         reply_markup=keyboard,
                         parse_mode=ParseMode.MARKDOWN)
```

Welcome message can be anything you want. Keyboard buttons can also be any text you want, but I've named them in the most obvious way for our bot: `Deposit` and `Balance`.

#### Balance button

Okay, now user can start the bot and see the keyboard with two buttons. But after calling one of these, user won't get any response because we didn't create any function for these.

So let's add a function to request a balance.

```python
@dp.message_handler(commands='balance')
@dp.message_handler(Text(equals='balance', ignore_case=True))
async def balance_handler(message: types.Message):
    uid = message.from_user.id

    # Get user balance from database
    # Also don't forget that 1 TON = 1e9 (billion) NanoTON
    user_balance = db.get_balance(uid) / 1e9

    # Format balance and send to user
    await message.answer(f'Your balance: *{user_balance:.2f} TON*',
                         parse_mode=ParseMode.MARKDOWN)
```

It's pretty simple. We just get the balance from database and send the message to user.

#### Deposit button

And what about second button - `Deposit`? Here is the function for it:

```python
@dp.message_handler(commands='deposit')
@dp.message_handler(Text(equals='deposit', ignore_case=True))
async def deposit_handler(message: types.Message):
    uid = message.from_user.id

    # Keyboard with deposit URL
    keyboard = InlineKeyboardMarkup()
    button = InlineKeyboardButton('Deposit',
                                  url=f'ton://transfer/{config.DEPOSIT_ADDRESS}&text={uid}')
    keyboard.add(button)

    # Send text that explains how to make a deposit into bot to user
    await message.answer('It is very easy to top up your balance here.\n'
                         'Simply send any amount of TON to this address:\n\n'
                         f'`{config.DEPOSIT_ADDRESS}`\n\n'
                         f'And include the following comment: `{uid}`\n\n'
                         'You can also deposit by clicking the button below.',
                         reply_markup=keyboard,
                         parse_mode=ParseMode.MARKDOWN)
```

What do we do here is also easy to understand.

Remember when in `ton.py` file we were determining which user made a deposit by comment with their UID? Now here in the bot we need to ask user to send transaction with a comment containing their UID.

### Bot start

The only thing we have to do now in `bot.py` is to launch the bot itself and also run the `start` function from `ton.py`.

```python
if __name__ == '__main__':
    # Create Aiogram executor for our bot
    ex = executor.Executor(dp)

    # Launch the deposit waiter with our executor
    ex.loop.create_task(ton.start())

    # Launch the bot
    ex.start_polling()
```

At this moment we have wrote all required code for our bot. If you did everything correctly it must work when you run it with `python my-bot/bot.py` command in terminal.

If your bot doesn't work correctly, compare your code with code [from this repository](https://github.com/Gusarich/ton-bot-example).

## References

 - Made for TON as part of [ton-footsteps/8](https://github.com/ton-society/ton-footsteps/issues/8)
 - By Gusarich ([Telegram @dani_goose](https://t.me/dani_goose), [Gusarich on GitHub](https://github.com/Gusarich))