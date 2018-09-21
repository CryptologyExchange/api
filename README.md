# Welcome to Cryptology community exchange documentation!

# Introduction

Welcome to Cryptology trader and developer documentation. These documents outline exchange functionality, market details, and APIs.

APIs are separated into two categories: trading and feed. Access to APIs requires authentication and provides access for placing orders, provides market data and other account information.

By accessing the Cryptology Data API, you agree to the [Terms & Conditions](https://cryptology.com/legal/terms/)

To access the Trading API server, add an access/secret key pair on [Account](https://cryptology.com/app/account) / [API](https://cryptology.com/app/account/api) section of Cryptology website.
You can generate one or more key pairs.

# General information

## Matching orders

Cryptology market operates a first come-first serve order matching. Orders are executed as received by the matching engine, from the older to newer received orders.

## Self-Trade Prevention

To discourage wash trading, Cryptology exchange cancels smaller order and
decreases larger order size by the smaller order size when two orders
from the same client cross. Yet Cryptology charges taker fee for the smaller order.
 If the two orders are the same size, both will be canceled, yet 
 Cryptology will charge taker fee for one order.

## Rate limit

Currently Cryptology has a rate limit of 10 requests per second. If you reach
this limit you will receive a message with `THROTTLING` response type and
`overflow_level` param which must be used for waiting `overflow_level`
milliseconds.


## Order Lifecycle

Valid orders sent to the matching engine are confirmed immediately and are in the received state. If an order executes against another order immediately, the order is considered done. An order can execute in part or whole. Any part of an order not filled immediately, will be considered open. Orders will stay in the open state until canceled or subsequently filled by new orders. Orders that are no longer eligible for matching (filled or canceled) are in the done state.

Currently WS API supports the following limit orders types: fill or kill (**FOK**) orders, immediate or cancel (**IOK**) orders, good 'til canceled (**GTC**) orders.

A fill or kill (**FOK**) is a type of time-in-force designation used in securities trading that instructs an exchange to execute a transaction immediately and completely or not at all. The order must be filled in its entirety or canceled (killed).

An immediate or cancel order (**IOC**) is an order to buy or sell a security that must be immediately filled. Any unfilled portion of the order is canceled.

A good ’til canceled (**GTC**) describes an order a trader may place to buy or sell a security that remains active until either the order is filled or the trader cancels it.

GTC order supports Time In Force instruction. Time in force is a special instruction used when placing a trade to indicate how long an order will remain active before it is executed or expires. These options are especially important for active traders and allow them to be more specific about the time parameters.


## Fees

### Trading Fees

Cryptology operates a maker-taker model. Orders which provide liquidity are charged different fees from orders taking liquidity. 

**Trading fees:**

Market maker: 0.02%

Market taker: 0.02%

**Deposits:**

Card deposits: 3.5%

Wire transfers: 0%

**Withdrawals:**

BTC: 0.0005;

BCH, LTC: 0.0003;

ETH: 0.009

EUR: €7 per SEPA transfer

## Data Centers

Cryptology data centers are in the Amazon EU region.

## Sandbox

A public sandbox is available for testing API connectivity and web trading. The sandbox provides all of the functionality of the production exchange but allows you to add fake funds for testing.

Login sessions and API keys are separate from production. We will issue some testing keys for you, to use them in the sandbox web interface and the sandbox environment.

To add funds, use the web interface deposit and withdraw buttons as you would on the production web interface.

**Sandbox URLs:**
When testing your API connectivity, make sure to use the following URLs.

Trading API:
wss://api.sandbox.cryptology.com

Market Data API:
wss://marketdata.sandbox.cryptology.com

**Website:**

[https://sandbox.cryptology.com] (https://sandbox.cryptology.com)

## Production

**Production URLs:**
For production use the following URLs.

Trading API:
wss://api.cryptology.com

Market Data API:
wss://marketdata.cryptology.com

**Website:**

[https://cryptology.com] (https://cryptology.com)

# Websocket API


## Installation


``` {.sourceCode .bash}
pip install git+https://github.com/CryptologyExchange/cryptology-ws-client-python.git

```

## Usage

Example of connection through our official Python client library for the Cryptology exchange WebSocket API.

*For more information see our [documentation](https://github.com/CryptologyExchange/cryptology-ws-client-python)
about our official Python client library and 
[Writing Your Own Trading Bot from Scratch](https://github.com/CryptologyExchange/api/blob/master/trading_bot.md) tutorial*

``` {.sourceCode .python3}
import asyncio
import itertools
import os
import logging
import time

from collections import namedtuple
from cryptology import ClientWriterStub, run_client, exceptions
from datetime import datetime
from decimal import Decimal
from typing import Iterable, Dict, List


logging.basicConfig(level='DEBUG')


async def main():

    async def writer(ws: ClientWriterStub, pairs: List, state: Dict) -> None:
        while True:
            client_order_id = int(time.time() * 10)
            await ws.send_message(payload={
                '@type': 'PlaceBuyLimitOrder',
                'trade_pair': 'BTC_USD',
                'price': '1',
                'amount': '1',
                'client_order_id': client_order_id,
                'ttl': 0
            })
            await asyncio.sleep(5)

    async def read_callback(ws: ClientWriterStub, ts: datetime, message_id: int, payload: dict) -> None:
        if payload['@type'] == 'BuyOrderPlaced':
            await ws.send_message(payload={'@type': 'CancelOrder', 'order_id': payload['order_id']})

    while True:
        try:
            await run_client(
                access_key='YOUR ACCESS KEY',
                secret_key='YOUR SECRET KEY',
                ws_addr='wss://api.sandbox.cryptology.com',
                writer=writer,
                read_callback=read_callback,
                last_seen_message_id=-1
            )
        except exceptions.ServerRestart:
            asyncio.sleep(60)


if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
```

# WS Trading protocol

Cryptology API operates over the WebSocket protocol with PING heartbeat being sent every 4 seconds (for details about WebSocket protocol, read [RFC 6455](https://tools.ietf.org/html/rfc6455)).



## Handshake

### Client sends a message with the following format:


```json
 {
    "access_key": "access key",
    "secret_key": "secret key",
    "last_seen_message_id": -1,
    "version": 6,
    "get_balances": true,
    "get_order_books": false
 }

```


| Name       | Description          |
| :-------------: |:-------------|
| `access_key`  | is a client access key |
| `secret_key`  | is a client secret key |
| `last_seen_message_id` | is the last message id which client got from a server in previous sessions or -1|
| `protocol_version` | is a version of protocol|
| `get_balances`| optional flags which ask the server to send user balances and (default: false)|
| `get_order_books`|optional flags which ask the server to send user order books (default: false)|


### Server responds with the following format:

```json
{
    "last_seen_sequence": 100000,
    "server_version": 6,
    "state": {
        "message_id": 5367625,
        "balances": {"BTC": {"available": "1",
                             "holded": "0"
                            },
                     "USD": {"available": "1000",
                             "holded": "120.25"
                            }
                    }
             },
    "greeting": "Welcome to Cryptology API Server",
    "trade_pairs": ["BTC_USD", "LTC_BTC", "ETH_USD"]
  }
```

| Name       | Description          |
| :-------------: |:-------------|
| `last_seen_sequence`  | is a last `sequense_id` which server received from client|
| `sequense_id`  | is the unique ID of current user request to the server|
| `server_version`  | is a version of the server |
| `state` | is an optional field which can contain current order_books and/or balances|
| `trade_pairs` | is an available trade pairs|
| `message_id` | is a message id at which a `state` is created |


## Messages

Request example

```json
{
    "sequence_id": 1212,
    "data": {
         "@type": "CancelOrder",
         "order_id": 42
     }
 }
```

There are the following types of server response messages: **MESSAGE**, **THROTTLING**,
**ERROR**.

**MESSAGE** response example:

```json
{
    "response_type": "MESSAGE",
    "timestamp": 1533214317,
    "message_id": 4763856,
    "data":  {
        "@type": "BuyOrderCancelled",
        "order_id": 1,
        "time": [
            946684800,
            0
        ],
        "trade_pair": "BTC_USD",
        "client_order_id": 123
    }
}
```

**THROTTLING** response example:

```json
{
    "response_type": "THROTTLING",
    "overflow_level": 12800,
    "sequence_id": 45,
    "message": "You have reached the maximum number of requests per second. Please try again in 12.8 seconds."
}
```

**ERROR** response example:

```json
{
    "response_type": "ERROR",
    "error_type": "DUPLICATE_CLIENT_ORDER_ID",
    "error_message": "Client order id 345 already exists"
}
```

**Params:**

| Name            | Type | Description                                                                                                                                                                                                                                                     | Required | Constraints                                          |
| --------------- | ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ---------------------------------------------------- |
| `response_type`  | enum | Response type                                                                                                                                                                                                                                                   | Yes      | Can have the following values: MESSAGE, THROTTLING, ERROR |
| `sequence_id`    | int  | Unique identifier of performed operation. Auto processed by client library                                                                                                                                                                                      | Yes      |                                                      |
| `message_id`     | int  | Is an incremental (but not necessarily sequential) value indicating message order on server and used by the client to skip processed events on reconnect                                                                                                        | Yes      |                                                      |
| `timestamp`       | int  | Time of operation performed                                                                                                                                                                                                                                     | Yes      | Valid timestamp                                      |
| `data`              | json | Body of message                                                                    | Yes |  |
| `overflow_level`   | int  | Amount of orders the client should postpone sending to keep up with the rate limit | Yes |  |
| `client_order_id` | int  | Order id which is sent by client                                                      | No  |  |
| `error_type`       | enum | Error type `ERROR_TYPE`                                                            | Yes |  |
| `error_message`    | str  | Error description                                                                  | Yes |  |

`ERROR_TYPE` determines an error type:

| `ERROR_TYPE`    | Description          |
| :-------------: |:-------------|
| `DUPLICATE_CLIENT_ORDER_ID error`  | `client_order_id` must be a unique field for each order created. `DUPLICATE_CLIENT_ORDER_ID` means that `client_order_id` in the sent message is not unique.|
 `INVALID_PAYLOAD error`  | All client messages must be in a valid JSON format and contain all the required fields. `INVALID_PAYLOAD` means that client sends an invalid JSON or any required parameter is not sent.|
| `UNKNOWN_ERROR error`  | Any other errors. |


### Critical errors
Some errors can cause server connection closing

|Error text | Description |
| :----------:|:-------------|
|Authentication Failed|  Invalid or removed access/secret key pair. Send correct access/secret key pair|
|Trade for This Access Key Not Permitted | Generate another access/secret key pair with trade permission|
|Your IP Address Has No Permission for This Access Key  | Generate another access/secret key pair and set up IPs to access|
|Two Simultaneous Connections| You can connect to API with one access key per one connection only|

# WS Market data protocol

Market data is broadcasted via web socket. It is read only
and has no integrity checks aside of Web Socket built-in mechanisms.

## Messages

-   `OrderBookAgg`
    :   aggregated order book for a given symbol, recalculated after each
    order book change (most likely will be throttled to reasonable interval in future). May have empty dictionaries `buy_levels` or `sell_levels` in case of an empty order book. Both dictionaries use the price as a key and volume as a value. `current_order_id`
    denotes the order that leads to the state of the order book.


    ```json
    {
        "@type": "OrderBookAgg",
        "buy_levels": {
            "1": "1"
        },
        "sell_levels": {
            "0.1": "1"
        },
        "trade_pair": "BTC_USD",
        "current_order_id": 123456
    }
    
    ```


-   `AnonymousTrade`
    :   a trade has taken place. `time` has two parts - integer seconds
        and integer milliseconds UTC. `maker_buy` is true if maker is a buyer. If maker is a seller it is false. If maker is a buyer than taker is a seller and conversely.

    
    ```json
    {
        "@type": "AnonymousTrade",
        "time": [1530093825, 0],
        "trade_pair": "BTC_USD",
        "current_order_id": 123456,
        "amount": "42.42",
        "price": "555",
        "maker_buy": false
    }
    
    ```
    

## Server messages

### Order lifecycle

After a place order message is received by Cryptology the following messages will be sent over web socket connection. All order-related messages are user specific (i.e. you can't receive any of these messages for regular or other user orders). The `time` parameter is a list of two integers. The first one is a UNIX timestamp in the UTC time zone. The second is a number of microseconds.

-   `BuyOrderPlaced`, `SellOrderPlaced`
    :   order was received by cryptology. `closed_inline` indicates an
        order that was fully executed immediately, it’s safe not to
        expect (and, therefore ignore) other messages for this order.
        End of order lifecycle. `initial_amount` equals to the full
        order size while `amount` is the part of the order left after
        instant order execution and placed to the order book.

    ```json
    {
        "@type": "BuyOrderPlaced",
        "amount": "1",
        "initial_amount": "3",
        "closed_inline": false,
        "order_id": 1,
        "price": "1",
        "time": [
            946684800,
            0
        ],
        "trade_pair": "BTC_USD",
        "client_order_id": 123
    }
    ```

-   `BuyOrderAmountChanged`, `SellOrderAmountChanged`
    :   order was partially executed, sets a new amount

    ```json
    {
        "@type": "BuyOrderAmountChanged",
        "amount": "1",
        "order_id": 1,
        "fee": "0.002",
        "time": [
            946684800,
            0
        ],
        "trade_pair": "BTC_USD",
        "client_order_id": 123
    }
    ```

-   `BuyOrderCancelled`, `SellOrderCancelled`
    :   order was canceled (manual, TTL, IOC, FOK, tbd), end of order
        lifecycle
    
    ```json
    {
        "@type": "BuyOrderCancelled",
        "order_id": 1,
        "time": [
            946684800,
            0
        ],
        "trade_pair": "BTC_USD",
        "client_order_id": 123
    }
    ```

-   `BuyOrderClosed`, `SellOrderClosed`
    :   order was fully executed, end of order lifecycle

   
   
    ```json
    {
        "@type": "BuyOrderClosed",
        "order_id": 1,
        "time": [
            946684800,
            0
        ],
        "trade_pair": "BTC_USD",
        "client_order_id": 123
    }
    
    ```
    
    

-   `OrderNotFound`
    :   attempt to cancel a non-existing order was made

    
    
    ```json
    {
        "@type": "OrderNotFound",
        "order_id": 1
    }
    
    ```



### Wallet

-   `SetBalance`
    :   sets a new client balance for a given currency. `reason` can be
        `trade` or `on_hold` for the changes caused by trades,
        `transfer` for balance update by depositing money or `withdraw`
        as a result of a withdrawal.

    
    
    ```json
            {
                "@type": "SetBalance",
                "balance": "1",
                "change": "1",
                "currency": "USD",
                "reason": "trade",
                "time": [
                    946684800,
                    0
                ]
            }
    
    ```
    
    
    `change` is amount by which the balance has changed. Positive if it increased and negative if decreased.

-   `InsufficientFunds`
    :   indicates that an account doesn't have enough funds to place an
        order
    
    
    
    ```json
    {
        "@type": "InsufficientFunds",
        "order_id": 1,
        "currency": "USD"
    }
    
    ```
    
    

-   `WithdrawalInitializedSuccess`
    : indicates that withdrawal is initialized and will be performed.

    
    ```json
    {
        "@type": "WithdrawalInitializedSuccess",
        "currency": "BTC",
        "amount": "0.1",
        "to_wallet": "your_wallet_address",
        "payment_id": "54085bc9-d699-443a-ab3e-2160e9d2e38e"
    }
    
    ```
    
    

    `your_wallett_address`  is your wallet address where payment will
    be transferred, `payment_id` is a unique payment identifier.

-   `DepositAddressGenerated`
    : returns generated address for crypto deposits.

    
    ```json
    {
        "@type": "DepositAddressGenerated",
        "currency": "BTC",
        "wallet_address": "your_deposit_wallet_address"
    }
    
    ```
    
    

    `your_deposit_wallet_address` is your wallet address for deposits.


-   `DepositTransactionAccepted`
    :   indicates transaction information when depositing crypto funds to the
        account


    ```json
    {
        "@type": "DepositTransactionAccepted",
        "currency": "BTC",
        "amount": "0.1",
        "transaction_info": {
            "to_address": "0x49293a856169d46dbf789c89b51b2ca6c7d1c4f50x4",
            "blockchain_tx_ids": [
                "0x124129474b1dcbdb4e39436de49f7e5987f46dc4b8740966655718d7a1da699b"
            ],
            "payment_id": "55085bc3-d699-243a-ab3e-1960e9d2e08e"
        },
        "time": [
            946684800,
            0
        ]
    }
    
    ```
    
    

-   `WithdrawalTransactionAccepted`
    :   indicates transaction information when withdrawing crypto funds from
        the account

    
    
    ```json
    {
        "@type": "WithdrawalTransactionAccepted",
        "currency": "BTC",
        "amount": "0.1",
        "transaction_info": {
            "to_address": "0x49293a856169d46dbf789c89b51b2ca6c7d1c4f50x4",
            "blockchain_tx_ids": [
                "0x124129474b1dcbdb4e39436de49f7e5987f46dc4b8740966655718d7a1da699b"
            ],
            "payment_id": "54085bc9-d699-443a-ab3e-2160e9d2e38e"
        },
        "time": [
            946684800,
            0
        ]
    }
    
    ```
    
    

-   `DepositAddressGeneratingError`
    :    indicates that generating deposit address finished with error
    
    
    ```json
    {
        "@type": "DepositAddressGeneratingError",
        "message": "Multiple addresses is not allowed"
    }
    ```
    
    

-   `WithdrawalError`
    :    indicates that withdrawal not performed because error caused
    
    
    ```json
    {
        "@type": "WithdrawalError",
        "message": "Minimum withdrawal value is not reached"
    }
    ```
    
  
    

### General

-   `OwnTrade`
     :   sent when the account participated in a deal on either side.
         `maker` equals `true` if the account was a maker. `maker_buy`
         equals `true` if the maker side was buying.

    
    ```json
    {
        "@type": "OwnTrade",
        "time": [
            946684800,
            0
        ],
        "trade_pair": "BTC_USD",
        "amount": "1",
        "price": "1",
        "maker": true,
        "maker_buy": false,
        "order_id": 1,
        "client_order_id": 123
    }
    
    ```
    
    

-    `SelfTrade`
     :   sent when the account perform self trading.
         `maker_buy` equals `true` if the opposite order was a buy order.


     ```json
     {
         "@type": "SelfTrade",
         "time": [
             946684800,
             0
         ],
         "trade_pair": "BTC_USD",
         "amount": "1",
         "price": "1",
         "maker_buy": false,
         "opposite_currency": "BTC",
         "opposite_order_id": 0,
         "currency": "USD",
         "order_id": 1,
         "fee": "0.0002",
         "client_order_id": 123
     }
     
     ```
     
     

## Client messages

### Limit Order Placement

- `PlaceBuyLimitOrder`
    :   limit bid

- `PlaceBuyFoKOrder`
    :   fill or kill bid

- `PlaceBuyIoCOrder`
    :   immediate or cancel bid

- `PlaceSellLimitOrder`
    :   limit ask

- `PlaceSellFoKOrder`
    :   fill or kill ask

- `PlaceSellIoCOrder`
    :   immediate or cancel ask

all limit order placement messages share the same structure


```json
{
    "@type": "PlaceBuyLimitOrder",
    "trade_pair": "BTC_USD",
    "amount": "10.1",
    "price": "15000.3",
    "client_order_id": 123,
    "ttl": 0
}

```



### Stop-Limit Order Placement

- `PriceLowerCondition`
    : stop-limit order to sell

- `PriceGreaterCondition`
    : stop-limit order to buy

all stop-limit order placement messages share the same structure



```json5
{
    "@type": "PriceLowerCondition",
    // stop price
     "price": "5000",
     "order": {
         "trade_pair": "BTC_USD",
         // limit price
         "price": "4900",
         "amount": "1",
         "client_order_id": 243,
         "ttl": 0
     }
}


```




`client_order_id` is a tag to relate server messages to client ones.
`ttl` is the time the order is valid for. Measured in seconds (with 1
minute granularity). 0 means valid forever.

### Order cancelation

-   `CancelOrder`
    :   cancel any order

    
    ```json
    {
        "@type": "CancelOrder",
        "order_id": 42
    }
    
    ```
    
    

-   `CancelAllOrders`
    :   cancel all active orders opened by the client

    
    ```json
    {
        "@type": "CancelAllOrders"
    }
    
    ```
    


### Order moving

You can change order `price` or `ttl` without manually cancelling and creating it
with new params.
For this purpose, you can use `MoveOrder` message.



```json
{
    "@type": "MoveOrder",
    "order_id": 42,
    "price": "100.75",
    "ttl": 0
}

```




### Wallet

-   For initializing crypto withdrawal to saved wallet address you can
    use `WithdrawCrypto` message.

    
    ```json
    {
        "@type": "WithdrawCrypto",
        "currency": "BTC",
        "amount": "0.32",
        "to_wallet": "your_wallet_address"
    }
    
    ```


    where `your_wallet_address` is your saved wallet address.
    After sending this message, you can receive
    `WithdrawalInitializedSuccess` or `WithdrawalError` message.
    See Server Messages / [Wallet](#wallet) Section for details.

-   For generating crypto address for deposit to your account you can 
    use `GenerateDepositAddress` message.
    
    
     
     ```json
    {
        "@type": "GenerateDepositAddress",
        "currency": "LTC",
        "create_new": true
    }
    
    
    ```
    
    
    
    where `create_new` flag means that if you already have generated an address,
    it will be returned. Otherwise, a new address will be generated if it is
    permitted for your account. After sending this message, you can receive
    `DepositAddressGenerated` or `DepositAddressGeneratingError` message.
    See Server Messages / [Wallet](#wallet) Section for details.


