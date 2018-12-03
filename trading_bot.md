# Writing Your Own Trading Bot from Scratch

This tutorial is intended to teach you to write a simple Market Maker Bot. 

## Preparatory Step 

1. Install Python of version not older than 3.6 to start.

    For Debian and Ubuntu just execute the following command:
    
    ``` {.sourceCode .bash}
    sudo apt-get install python3
    ```
    
    Then install pip using this command: 
    
    ``` {.sourceCode .bash}
    sudo apt-get install python3-pip
    ```

    For Mac OS and Windows download Python from the [official website](https://www.python.org/downloads/) and install it.

2. After that, install client library Cryptology API:

    ``` {.sourceCode .bash}
        pip install cryptology-ws-client
    ```

## Bot writing

1. To use WebSocket API Cryptology, import client library Cryptology API and write a simple sample of asynchronous program that does nothing:
   
   ``` {.sourceCode .python3}
   import asyncio
   import cryptology
    
   async def main():
       pass
    
   if __name__ == '__main__':
       loop = asyncio.get_event_loop()
       loop.run_until_complete(main())
   ```

2. Connect to API server with the use of imported library. 
   For this you'll need API keys. Generate a key pair on Cryptology website. 

   To connect to the server, use the function `run_client()` of the library, passing it as parameters as follows:
    - Method that will send messages to WebSocket
      
      ``` {.sourceCode .python3}
      async def writer(ws: ClientWriterStub,pairs: List,
                       state: Dict):
          pass
      ```
    
    - Method that will receive and process messages from the server
      ``` {.sourceCode .python3}
      async def read_callback(ws: ClientWriterStub, ts: datetime,
                              message_id: int, payload: dict) -> None:
          pass
      ```
    
    - Address of WS server. We will use Sandbox at:
      `wss://api.sandbox.cryptology.com`
    
    - Access and secret keys, that we have previously generated

   The result shall be the following:

   ``` {.sourceCode .python3}
   import asyncio
   import cryptology
   from decimal import Decimal
    
   async def main():
    
       async def writer(ws: ClientWriterStub,pairs: List,
                        state: Dict):
           pass
        
       async def read_callback(ws: ClientWriterStub, ts: datetime,
                               message_id: int, payload: dict) -> None:
           pass
            
       await cryptology.run_client(
           access_key='MY ACCESS KEY',
           secret_key='MY SECRET KEY',
           ws_addr='wss://api.sandbox.cryptology.com',
           writer=writer,
           read_callback=read_callback
       )
    
   if __name__ == '__main__':
       loop = asyncio.get_event_loop()
       loop.run_until_complete(main())
   ```
   
   'MY ACCESS KEY' and 'MY SECRET KEY' are your access and secret keys generated at the previous step (you'll get your own ones).

   Now run what you've got. The program terminates immediately. But why? 
   This is because the connection closes just after the method  `writer()`
   is completed. Add asynchronous sleep to it. Remember that all methods you
   call within the code shall be asynchronous. You need this not to block
   the program run. It shall look like this:

   ```{.sourceCode .python3}
   async def writer(ws: ClientWriterStub,pairs: List,
                    state: Dict):
       await asyncio.sleep(10)
   ```

   Run the program again. Now it does not terminate within 10 sec.

3. Thus, we have already connected to Cryptology server. Try to make an order.
   Take CTX_BTC trade pair, for instance. Trade using a third of your balance
   not to spend all the money. But how shall you get to know your balance?
   To do this, pass the additional parameter  `get_balances` with value
   `True` to the function `run_client()`. The result shall be as follows:

   ```{.sourceCode .python3}
   await cryptology.run_client(
        access_key='MY ACCESS KEY',
        secret_key='MY SECRET KEY',
        ws_addr='wss://api.sandbox.cryptology.com',
        writer=writer,
        read_callback=read_callback,
        get_balances=True
   )
   ```

   Now save your balances in the variable `balances`.
   All balances are received from the server as the parameter `state` of the method `writer()`.

   ```{.sourceCode .python3}
   balances = {}
   async def writer(ws: ClientWriterStub, pairs: List,
                    state: Dict):
       balances.update(state['balances'])
   ```
   
   To make a buy order, just send `PlaceBuyLimitOrder` type message.
   To do this you need to call the `send_message` in the function `writer()`.
   Buy CTX with a third of all BTC available at 0.01 BTC per CTX:
   
   ```{.sourceCode .python3}
   async def writer(ws: ClientWriterStub, pairs: List,
                    state: Dict):
       btc_balance = Decimal(balances['BTC']['available'])
       price = 0.01
       amount = (btc_balance / price)
   
       # Buy with a third of all BTC available
       amount /= 3
   
       # round off
       amount = int(amount * (10**8)) / (10**8)
       await ws.send_message({'@type': 'PlaceBuyLimitOrder',
                              'trade_pair': 'CTX_BTC',
                              'amount': str(amount),
                              'price': str(price)})
   ```

   Run what you've got. You've made a CTX buy order with a third of BTC
   balance at 0.01 BTC per each.

4. To keep your balance in the current state, add balance change message processing
   into the function `read_callback()`:
   
   ```{.sourceCode .python3}
   async def read_callback(ws: ClientWriterStub, ts: datetime,
                           message_id: int, payload: dict) -> None:
       if payload['@type'] == 'SetBalance':
           balances.setdefault(payload['currency'], {'available': 0})
           balances[payload['currency']]['available'] = payload['balance']
   ```

5. It is convenient to pass the trading, spread and currencies into constants: 

    ```{.sourceCode .python3}
    TRADE_PAIR = 'CTX_BTC'
    COIN_PRICE = 100
    SPREAD = 1  # percent
    BASE_CURRENCY, SECOND_CURRENCY = TRADE_PAIR.split('_')
    ```

6.  Make an operation on creating a CTX buy order with a third of balance
    as a separate function: 

   ```{.sourceCode .python3}
   async def create_bid(ws: ClientWriterStub):
       second_currency_balance = Decimal(balances[SECOND_CURRENCY]['available'])
       bid_price = round_decimal(COIN_PRICE * (100 - SPREAD / 2) / 100)
        
       can_buy = second_currency_balance / bid_price
       bid_amount = round_decimal(can_buy / 3)
        
       await ws.send_message(payload={
           '@type': 'PlaceBuyLimitOrder',
           'trade_pair': TRADE_PAIR,
           'price': str(bid_price),
           'amount': str(bid_amount)
       })
   
   ```

7. Create a similar function to make a sell order:
   
   ```{.sourceCode .python3}
   async def create_ask(ws: ClientWriterStub):
       base_currency_balance = Decimal(balances[BASE_CURRENCY]['available'])
       ask_price = round_decimal(COIN_PRICE * (100 + SPREAD / 2) / 100)
       
       ask_amount = round_decimal(base_currency_balance / 3)
       
       await ws.send_message(payload={
           '@type': 'PlaceSellLimitOrder',
           'trade_pair': TRADE_PAIR,
           'price': str(ask_price),
           'amount': str(ask_amount)
       })
   ```

8. Add a call of these methods into the function `writer()`. For client does
   not stop working immediately, run infinite loop with `sleep()`.
   As a result, the method `writer()` will be as follows: 
   
   ```{.sourceCode .python3}
   async def writer(ws: ClientWriterStub, pairs: List, state: Dict) -> None:
       balances.update(state['balances'])
       
       await create_bid(ws)
       await create_ask(ws)
       while True:
           await asyncio.sleep(5)
   ```

9. If other users redeem the order, you need to recreate them with a third
   of balance available. For this add message processing for`BuyOrderClosed`
   and `SellOrderClosed`:

   ```{.sourceCode .python3}
   async def read_callback(ws: ClientWriterStub, ts: datetime,
                           message_id: int, payload: dict) -> None:
   
      if payload['@type'] == 'BuyOrderClosed':
          await create_bid(ws)
      if payload['@type'] == 'SellOrderClosed':
          await create_ask(ws)
      elif payload['@type'] == 'SetBalance':
          balances.setdefault(payload['currency'], {'available': 0})
          balances[payload['currency']]['available'] = payload['balance']
   ```

10. Market maker is practically ready. It can make orders now,
    but after the restart it possibly will not receive messages on orders closed while the program was off.  To prevent this, save
    `message_id` into the file and read and pass it to the method `run_client()`upon restart.
    To save `message_id` , write the following method:

    ```{.sourceCode .python3}
    import os
   
    def write_last_seen_message_id(last_seen_message_id: int) -> None:
        file_path = os.path.join('.', 'last_seen_message_id')
        with open(file_path, 'w') as f:
            f.write(str(last_seen_message_id))   
    ```
    
    And also a method to read what you've written:
   
    ```{.sourceCode .python3}
    def read_last_seen_message_id() -> int:
        file_path = os.path.join('.', 'last_seen_message_id')
        if os.path.exists(file_path):
            with open(file_path, 'r') as f:
                return int(f.read())
        else:
            return -1
    ```
    
    Now it is necessary to call these methods. Call method `write_last_seen_message_id()`
    into `read_callback()`. It will expand to the following:

    ```{.sourceCode .python3}
    async def read_callback(ws: ClientWriterStub, ts: datetime,
                            message_id: int, payload: dict) -> None:
        write_last_seen_message_id(message_id)
   
        if payload['@type'] == 'BuyOrderClosed':
            await create_bid(ws)
        if payload['@type'] == 'SellOrderClosed':
            await create_ask(ws)
        elif payload['@type'] == 'SetBalance':
            balances.setdefault(payload['currency'], {'available': 0})
            balances[payload['currency']]['available'] = payload['balance']
    ```
    
    Call the method `read_last_seen_message_id()` when calling 
    `run_client()`. It will expand to the following:

    ```{.sourceCode .python3}
    await cryptology.run_client(
        access_key='MY ACCESS KEY',
        secret_key='MY SECRET KEY',
        ws_addr='wss://api.sandbox.cryptology.com',
        writer=writer,
        read_callback=read_callback,
        get_balances=True,
        last_seen_message_id=read_last_seen_message_id()
    )
    ```


## Result

As a result you will have the following:

```{.sourceCode .python3}
import asyncio
import os
import logging

from cryptology import ClientWriterStub, run_client, exceptions
from datetime import datetime
from decimal import Decimal
from typing import Dict, List

SERVER = os.getenv('SERVER', 'wss://api.sandbox.cryptology.com')

logging.basicConfig(level='INFO')
logger = logging.getLogger(__name__)


ACCESS_KEY = 'YOUR ACCESS KEY'
SECRET_KEY = 'YOUR SECRET KEY'


TRADE_PAIR = 'CTX_BTC'
COIN_PRICE = 0.15
SPREAD = 1  # percent
BASE_CURRENCY, QUOTED_CURRENCY = TRADE_PAIR.split('_')


def read_last_seen_message_id() -> int:
    file_path = os.path.join('.', 'last_seen_message_id')
    if os.path.exists(file_path):
        with open(file_path, 'r') as f:
            return int(f.read())
    else:
        return -1


def write_last_seen_message_id(last_seen_message_id: int) -> None:
    file_path = os.path.join('.', 'last_seen_message_id')
    with open(file_path, 'w') as f:
        f.write(str(last_seen_message_id))


def round_decimal(number: Decimal, precision: int = 8):
    return number.quantize(Decimal(10) ** -precision).normalize()


async def main():
    balances = {}

    async def create_bid(ws: ClientWriterStub):
        second_currency_balance = Decimal(balances[QUOTED_CURRENCY]['available'])
        assert second_currency_balance, 'Account has insufficient funds for {}'.format(QUOTED_CURRENCY)
        bid_price = round_decimal(Decimal(COIN_PRICE * (100 - SPREAD / 2) / 100))

        can_buy = second_currency_balance / bid_price
        bid_amount = round_decimal(Decimal(can_buy / 3))

        await ws.send_message(payload={
            '@type': 'PlaceBuyLimitOrder',
            'trade_pair': TRADE_PAIR,
            'price': str(bid_price),
            'amount': str(bid_amount)
        })

    async def create_ask(ws: ClientWriterStub):
        base_currency_balance = Decimal(balances[BASE_CURRENCY]['available'])
        assert base_currency_balance, 'Account has insufficient funds for {}'.format(BASE_CURRENCY)
        ask_price = round_decimal(Decimal(COIN_PRICE * (100 + SPREAD / 2) / 100))

        ask_amount = round_decimal(Decimal(base_currency_balance / 3))

        await ws.send_message(payload={
            '@type': 'PlaceSellLimitOrder',
            'trade_pair': TRADE_PAIR,
            'price': str(ask_price),
            'amount': str(ask_amount)
        })

    async def writer(ws: ClientWriterStub, pairs: List, state: Dict) -> None:
        balances.update(state['balances'])

        await create_bid(ws)
        await create_ask(ws)
        while True:
            await asyncio.sleep(5)

    async def read_callback(ws: ClientWriterStub, ts: datetime, message_id: int, payload: dict) -> None:
        write_last_seen_message_id(message_id)

        if payload['@type'] == 'BuyOrderClosed':
            logger.info('Buy order %i closed', payload['order_id'])
            await create_bid(ws)
        elif payload['@type'] == 'SellOrderClosed':
            logger.info('Sell order %i closed', payload['order_id'])
            await create_ask(ws)
        elif payload['@type'] == 'SetBalance':
            balances.setdefault(payload['currency'], {'available': 0})
            balances[payload['currency']]['available'] = payload['balance']
        elif payload['@type'] == 'BuyOrderPlaced':
            logger.info('Buy order with id %i, amount %s and price %s is placed',
                        payload['order_id'], payload['amount'], payload['price'])
        elif payload['@type'] == 'SellOrderPlaced':
            logger.info('Sell order with id %i, amount %s and price %s is placed',
                        payload['order_id'], payload['amount'], payload['price'])

    try:
        await run_client(
            access_key=ACCESS_KEY,
            secret_key=SECRET_KEY,
            ws_addr=SERVER,
            writer=writer,
            read_callback=read_callback,
            last_seen_message_id=read_last_seen_message_id(),
            get_balances=True
        )
    except exceptions.ServerRestart:
        asyncio.sleep(60)


if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())

```

## Conclusion

Thus, you've written a bot that can trade on the exchange.
Based on this bot you can write your own one and start making money.

We constantly add some new functions, don't miss the updates of
the [API Documentation](https://github.com/CryptologyExchange/api)
