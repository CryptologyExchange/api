# INTRODUCTION

Welcome to Cryptology trader and developer documentation. These documents outline exchange functionality, market details, and APIs.

APIs are separated into two categories: trading and feed. Access to APIs requires authentication and provides access for placing orders, provides market data and other account information.

By accessing the Cryptology Data API, you agree to the [Terms & Conditions](https://cryptology.com/legal/terms/)

To access the Trading API server, add an access/secret key pair on [Account](https://cryptology.com/app/account) / [API](https://cryptology.com/app/account/api) section of Cryptology website.
You can generate one or more key pairs.

# GENERAL INFORMATION

## Matching Orders

Cryptology market operates a first come-first serve order matching. Orders are executed as received by the matching engine, from the older to newer received orders.

## Self-Trade Prevention

To discourage wash trading, Cryptology exchange cancels smaller order and
decreases larger order size by the smaller order size when two orders
from the same client cross. Yet Cryptology charges taker fee for the smaller order.
 If the two orders are the same size, both will be canceled, yet 
 Cryptology will charge taker fee for one order.

## Rate Limit

Currently Cryptology has a rate limit of 10 requests per second for WebSocket
API and 1 request per second for HTTPS API. If you reach this limit you will receive
a message with `THROTTLING` response type and
`overflow_level` param which must be used for waiting for `overflow_level`
milliseconds.

## Unfair Price Prevention

We prevent order placement with a price different from the market price by more than 10%.
For example, if current best ask is 1000, you can't make Buy order with a price
greater than 1100. And if the current best bid is 1000, you can't make Sell order
with a price lower than 900.

## Decimal Precision

Each amounts and a prices on Cryptology have 8 decimal places.


## Order Lifecycle

Valid orders sent to the matching engine are confirmed immediately and are in the received state. If an order executes against another order immediately, the order is considered done. An order can execute in part or whole. Any part of an order not filled immediately, will be considered open. Orders will stay in the open state until canceled or subsequently filled by new orders. Orders that are no longer eligible for matching (filled or canceled) are in the done state.

Currently WS API supports the following limit orders types: fill or kill (**FOK**) orders,
immediate or cancel (**IOK**) orders, good 'til canceled (**GTC**) orders, and good 'til day (**GTD**) orders.

A fill or kill (**FOK**) is a type of time-in-force designation used in securities trading that instructs an exchange
to execute a transaction immediately and completely or not at all. The order must be filled in its entirety
or canceled (killed).

An immediate or cancel order (**IOC**) is an order to buy or sell a security that must be immediately filled.
Any unfilled portion of the order is canceled.

A good ’til canceled (**GTC**) describes an order a trader may place to buy or sell a security that remains
active until either the order is filled or the trader cancels it.

A good 'til day order is an order which will be canceled at the time preset by a trader if it is not executed or cancelled until this time.  A GTD order contains the Time To Live (**TTL**) instruction.
Time To Live is a special instruction which shows the time in which an order will be automatically canceled. 


## Fees

[FAQ] (https://intercom.help/cryptologyexchange/english/frequently-asked-questions-faq)

## Data Centers

Cryptology data centers are in the Amazon EU region.

## Sandbox

A public sandbox is available for testing API connectivity and web trading. The sandbox provides all of the functionality of the production exchange but allows you to add fake funds for testing.

Login sessions and API keys are separate from production. We will issue some testing keys for you, to use them in the sandbox web interface and the sandbox environment.

To add funds, use the web interface deposit and withdraw buttons as you would on the production web interface.

**Sandbox URLs:**
When testing your API connectivity, make sure to use the following URLs.

Trading API:
wss://api-sandbox.cryptology.com

Market Data API:
wss://marketdata-sandbox.cryptology.com

HTTPS API:
https://api-sandbox.cryptology.com


**Website:**

[https://sandbox.cryptology.com] (https://sandbox.cryptology.com)

## Production

**Production URLs:**
For production use the following URLs.

Trading API:
wss://api.cryptology.com

Market Data API:
wss://marketdata.cryptology.com

HTTPS API:
https://api.cryptology.com

**Website:**

[https://cryptology.com] (https://cryptology.com)

# WEBSOCKET API


## Installation


```bash
pip install cryptology-ws-client

```

## Usage

Example of connection through our official Python client library for the Cryptology exchange WebSocket API.

*For more information see our [documentation](https://github.com/CryptologyExchange/cryptology-ws-client-python)
about our official Python client library and 
[Writing Your Own Trading Bot from Scratch](https://github.com/CryptologyExchange/api/blob/master/trading_bot.md) tutorial*

```python
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

# Trading protocol

Cryptology API operates over the WebSocket protocol with PING
heartbeat being sent every 4 seconds (for details about WebSocket protocol,
read [RFC 6455](https://tools.ietf.org/html/rfc6455)).



## Authentication

After connecting to WebSocket Trading Server, send a message for authentication on the server.
With client-side authentication message, send `last_seen_message_id`.
After successful authentication, the server sends an authentication message
to WebSocket connection and all messages addressed to you with `message_id`
started from `last_seen_message_id`, but not older than 1 day.

With server-side authentication message, you receive `last_seen_sequence`.
Start sending messages to the server with `sequence_id` more than
`last_seen_sequence` in series. For example, if you receive `last_seen_sequence`
equal to 654,`sequence_id` of next client message which will be sent to the server
is 655. Otherwise, the connection will be closed with "Invalid Sequence" error.
`sequence_id` in all subsequent client messages must be serial.
For example 656, 657, 658 etc.

### Client sends a message in the following format:

> Example of client message:

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


### Server responds in the following format:

> Example of server response:

```json
{
    "last_seen_sequence": 100000,
    "server_version": 6,
    "state": {
        "message_id": 5367625,
        "balances": {"BTC": {"available": "1",
                             "on_hold": "0"
                            },
                     "USD": {"available": "1000",
                             "on_hold": "120.25"
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


## Requests

> Request example

```json
{
    "sequence_id": 1212,
    "data": {
         "@type": "CancelOrder",
         "order_id": 42
     }
 }
```

## Responses

There are the following types of server response messages: **MESSAGE**, **THROTTLING**.

> **MESSAGE** response example:

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

> **THROTTLING** response example:

```json
{
    "response_type": "THROTTLING",
    "overflow_level": 12800,
    "sequence_id": 45,
    "message": "You have reached the maximum number of requests per second. Please try again in 12.8 seconds."
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

### Errors
All protocol errors are cause server connection closing

|Error text | Description |
| :----------:|:-------------|
|Authentication Failed| Invalid or removed access/secret key pair. Send correct access/secret key pair|
|Trade for This Access Key Not Permitted | Generate another access/secret key pair with trade permission|
|Your IP Address Has No Permission for This Access Key  | Generate another access/secret key pair and set up IPs to access|
|Two Simultaneous Connections| You can connect to API with one access key per one connection only|
|Invalid Sequence| You are sends inconsistent `sequence_id`|

## Server Messages

## Order Lifecycle

After a place order message is received by Cryptology the following messages will
be sent over web socket connection. All order-related messages are user specific
(i.e. you can't receive any of these messages for regular or other user orders).
The `time` parameter is a list of two integers. The first one is a UNIX timestamp in
the UTC time zone. The second is a number of microseconds.

-   `BuyOrderPlaced`, `SellOrderPlaced`
    :   order was received by cryptology. `closed_inline` indicates an
        order that was fully executed immediately, it’s safe not to
        expect (and, therefore ignore) other messages for this order.
        End of order lifecycle. `initial_amount` equals to the full
        order size while `amount` is the part of the order left after
        instant order execution and placed to the order book.


> BuyOrderPlaced / SellOrderPlaced


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

-   `PlacingOrderCancelled`
    :   placing order was canceled (no trading volume, price out of market)

> PlacingOrderCancelled


   ```json
    {
        "@type": "PlacingOrderCancelled",
        "order_id": 1,
        "time": [
            946684800,
            0
        ],
        "trade_pair": "BTC_USD",
        "reason": "price_out_of_market",
        "client_order_id": 123
    }
   ```

-   `BuyOrderAmountChanged`, `SellOrderAmountChanged`
    :   order was partially executed, sets a new amount

> BuyOrderAmountChanged / SellOrderAmountChanged

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

> BuyOrderCancelled / SellOrderCancelled    


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

> BuyOrderClosed / SellOrderClosed   

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

    

- `OrderNotFound`
    :   attempt to cancel a non-existing order was made

> OrderNotFound 


   ```json
    {
        "@type": "OrderNotFound",
        "order_id": 1
    }
    
   ```


## Wallet

-   `SetBalance`
    :   sets a new client balance for a given currency. `reason` can be
        `trade` or `on_hold` for the changes caused by trades,
        `transfer` for balance update by depositing money or `withdraw`
        as a result of a withdrawal.

> SetBalance

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


> `change` is amount by which the balance has changed. Positive if it increased and negative if decreased.

-   `InsufficientFunds`
    :   indicates that an account doesn't have enough funds to place an
        order
    
> InsufficientFunds  

   ```json
    {
        "@type": "InsufficientFunds",
        "order_id": 1,
        "currency": "USD"
    }
    
   ```


-   `WithdrawalInitializedSuccess`
    : indicates that withdrawal is initialized and will be performed.


> WithdrawalInitializedSuccess

   ```json
    {
        "@type": "WithdrawalInitializedSuccess",
        "currency": "BTC",
        "amount": "0.1",
        "to_wallet": "your_wallet_address",
        "payment_id": "54085bc9-d699-443a-ab3e-2160e9d2e38e"
    }
    
   ```

    

> `your_wallett_address`  is your wallet address where payment will
    be transferred, `payment_id` is a unique payment identifier.

-   `DepositAddressGenerated`
    : returns generated address for crypto deposits.

> DepositAddressGenerated    

   ```json
    {
        "@type": "DepositAddressGenerated",
        "currency": "BTC",
        "wallet_address": "your_deposit_wallet_address"
    }
    
   ```


> `your_deposit_wallet_address` is your wallet address for deposits.


-   `DepositTransactionAccepted`
    :   indicates transaction information when depositing crypto funds to the
        account

> DepositTransactionAccepted

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

> WithdrawalTransactionAccepted 

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
    

> DepositAddressGeneratingError  

   ```json
    {
        "@type": "DepositAddressGeneratingError",
        "message": "Multiple addresses is not allowed"
    }
   ```

    

-   `WithdrawalError`
    :    indicates that withdrawal not performed because error caused
    
> WithdrawalError    

   ```json
    {
        "@type": "WithdrawalError",
        "message": "Minimum withdrawal value is not reached"
    }
   ```


    

## General

-   `OwnTrade`
     :   sent when the account participated in a deal on either side.
         `maker` equals `true` if the account was a maker. `maker_buy`
         equals `true` if the maker side was buying.


> OwnTrade

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
     :   sent when the account performs self trading.
         `maker_buy` equals `true` if the opposite order was a buy order.

> SelfTrade

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

     

## Client Messages

## Limit Order Placement

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


> Example of order placement message:

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

> `client_order_id` is an optional tag to relate server messages to client ones.
`ttl` is the time the order is valid for. 0 means valid forever. 
`ttl` is available only for limit bid, limit ask orders and for
conditional orders messages.


## Stop-Limit Order Placement

- `PriceLowerCondition`
    : stop-limit order to sell

- `PriceGreaterCondition`
    : stop-limit order to buy

all stop-limit order placement messages share the same structure

> Example of stop-limit placement message:


```json
{
    "@type": "PriceLowerCondition",
    "price": "5000",
    "order": {
         "trade_pair": "BTC_USD",
         "price": "4900",
         "amount": "1",
         "client_order_id": 243,
         "ttl": 0
     }
}
```

>  External `price` is a stop price.
Internal `order` `price` is a price of an order which will be created after the stop `price`is reached.


## Order Cancelation

- `CancelOrder`
    :   cancel any order

> CancelOrder


   ```json
    {
        "@type": "CancelOrder",
        "order_id": 42
    }
    
   ```

    

-   `CancelAllOrders`
    :   cancel all active orders opened by the client

  

> CancelAllOrders

   ```json
    {
        "@type": "CancelAllOrders"
    }
    
   ```



## Order Moving

You can change order `price` or `ttl` without manually cancelling and creating it
with new params.
For this purpose, you can use `MoveOrder` message.


> MoveOrder

```json
{
    "@type": "MoveOrder",
    "order_id": 42,
    "price": "100.75",
    "ttl": 0
}

```




## Wallet

-   For initializing crypto withdrawal to saved wallet address you can
    use `WithdrawCrypto` message.


> WithdrawCrypto

   ```json
    {
        "@type": "WithdrawCrypto",
        "currency": "BTC",
        "amount": "0.32",
        "to_wallet": "your_wallet_address"
    }
    
   ```


> where `your_wallet_address` is your saved wallet address.
    After sending this message, you can receive
    `WithdrawalInitializedSuccess` or `WithdrawalError` message.
    See Server Messages / [Wallet](#wallet) Section for details.

-   For generating crypto address for deposit to your account you can 
    use `GenerateDepositAddress` message.
    

> GenerateDepositAddress 

   ```json
    {
        "@type": "GenerateDepositAddress",
        "currency": "LTC",
        "create_new": true
    }
    
    
   ```


    
> where `create_new` flag means that if you already have generated an address,
    it will be returned. Otherwise, a new address will be generated if it is
    permitted for your account. After sending this message, you can receive
    `DepositAddressGenerated` or `DepositAddressGeneratingError` message.
    See Server Messages / [Wallet](#wallet) Section for details.




# Market Data Protocol

Market data is broadcasted via web socket. It is read only
and has no integrity checks aside of Web Socket built-in mechanisms.

## Messages

-   `OrderBookAgg`
    :   aggregated order book for a given symbol, recalculated after each
    order book change (most likely will be throttled to reasonable interval in future).
    May have empty dictionaries `buy_levels` or `sell_levels` in case of an empty order book.
    Both dictionaries use the price as a key and volume as a value. `current_order_id`
    denotes the order that leads to the state of the order book.

> OrderBookAgg:

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
        and integer milliseconds UTC. `maker_buy` is true if maker is a buyer.
        If maker is a seller it is false. If maker is a buyer than taker is a seller and conversely.


> AnonymousTrade:

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


# HTTPS API

Cryptology HTTPS API allows requests with JSON request body for POST endpoints and
empty request body for GET endpoints.

# Request Format

All POST requests must contain valid JSON body.
All required params must be sent on JSON fields via POST or on URL params via GET.

**POST request body example:**

```json
{
    "trade_pair": "BTC_USD"
}
```

**GET request example:**

```
GET https://api.cryptology.com/v1/public/get-trades?trade-pair=BTC_USD
```

## Response Format

Every endpoint returns a 200 HTTPS response code and JSON response body.

**Successful response**

Successful response has "OK" status and "data" JSON field.

```json
{
    "status": "OK",
    "data": {},
    "error": null
}
```

Where "data" contains the result of request.

**Error response**

If request finishes with an error, response contains an "ERROR" status and "error" JSON field.

```json
{
    "status": "ERROR",
    "error": {"code": "INSUFFICIENT_FUND",
              "message": null},
    "data": null
}
```

Where "error" contains "code" of the error and optionally contains an error "message".

## Error Codes

| Param name      | Description          
| :-------------: |:-------------|
| `INSUFFICIENT_FUND`  | You don't have enough money for operation
| `INVALID_REQUEST`    | Invalid request arguments
| `INVALID_KEY`        | You need to send correct Access-Key and Secret-Key headers
| `INVALID_TIMESTAMP`  | Your timestamp must be within 30 seconds from the api server time. We can use the /public/time endpoint to query for the API server time. 
| `PERMISSION_DENIED`  | Your API key does not have permissions for this request.
| `TOO_MANY_REQUESTS`  | You have reached the rate limit.
| `DUPLICATE_CLIENT_ORDER_ID`  | You can't create two orders with the same `client_order_id`
| `UNKNOWN_ERROR`      | Any other error

## Types

**Order nature**

- `BUY`
- `SELL`

**Order status**

- `PENDING` - an order is being prepared for placing at the exchange
- `NEW` - an order is placed at the exchange 
- `FILLED` - an order is fully executed 
- `CANCELLED` - an order is canceled 

**Time in force**

- `GTC` stands for a good ’til canceled order
- `GTD` stands for a good 'til day order
- `FOK` stands for a fill or kill order
- `IOC` stands for an immediate or cancel order

**Payment status**

- `INITIALIZED` - withdrawal is initialized and will be made soon
- `PENDING` - withdrawal is started 
- `COMPLETE` - withdrawal is successfully completed
- `DECLINED` - withdrawal is declined 

**Order book type**

- `AGGREGATED`
- `BEST` (currently not supported)
- `FULL` (currently not supported)

**Candles interval types**

- `M1` - each candle includes a one-minute interval 
- `M20` - each candle includes a twenty-minute interval 
- `H1` - each candle includes a one-hour interval 
- `H6` - each candle includes a six-hour interval 
- `D1` - each candle includes a one-day interval


# Private

Private endpoints are only available for authorized users.
To make a request for authorisation you need send your access key in Access-Key header and your secret key in Secret-Key header. Each private request must contain Nonce header.
Nonce is a unique five-minute-interval number.

**Request headers example**

```
POST /v1/private/get-balances HTTP/1.1
Content-Type: application/json
Accept: application/json
Content-Type: application/json
Content-Length: 235
Access-Key: KJuyg3bdi3ubycvebijckniugvbuibivujb=
Secret-Key: ih74gfyuevcbernivyeucwevbli3h3y4gr74yrgvb37guvbfb483vfy38iv==
Nonce: 23
```

## Orders Management

At Cryptology exchange you can place and cancel orders, and request for the information regarding the placed, canceled and executed orders. 

### Create order

Creates an order with the specified parameters 

```
POST /v1/private/create-order
```

**REQUEST PARAMETRS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `trade_pair`  | Yes | A valid trading pair name
| `type` | Yes | Only LIMIT orders are currently supported
| `side`  | Yes | BUY/SELL
| `time_in_force` | No | IOC, FOK, GTC or GTD. GTC default
| `ttl` | No | Only for GTD orders
| `amount` | Yes | Decimal amount value
| `client_order_id` | No | A unique id of an order
| `price` | Yes | Decimal price value
| `stop_price` | No | Must be defined for stop-limit orders. Supported only for GTC orders

**RESPONSE FIELDS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `order_id`  | Yes | A number, an internal unique identifier of an order.

**Response example**

```json
{
    "order_id": 265167535
}
```

### Query for order

Returns the information about an order placed by a trader

```
GET /v1/private/get-order
```

**REQUEST PARAMETRS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `order_id`  | Yes | A valid order_id returned by create-order

**RESPONSE FIELDS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `order_id`  | Yes | A number, an internal unique identifier of an order
| `trade_pair`  | Yes | A trading pair name
| `type` | Yes | Only LIMIT orders are currently supported
| `side`  | Yes | BUY/SELL
| `time_in_force` | No | Time In Force of an order. IOC, FOK, GTC or GTD
| `ttl` | No | Only for GTD orders
| `amount` | Yes | Decimal amount value
| `executed_amount` | Yes| A part of an order that is already executed
| `client_order_id` | No | A unique id of an order specified by you at order creation
| `price` | Yes | Decimal price value, specified by you at order creation 
| `stop_price` | No | Must be defined for stop-limit orders. Applicable only to GTC orders
| `status` | Yes | Current order status 
| `created_at` | Yes | Unix timestamp in UTC. Order creation time
| `done_at` | No | Unix timestamp in UTC. Order completion time. Applicable only to completed orders. 

**Response data example**

```json
{
    "trade_pair": "BTC_USD",
    "side" : "BUY",
    "type": "LIMIT",
    "status": "NEW",
    "order_id": 265167535,
    "client_order_id": 543,
    "price": "123.45",
    "amount": "0.21",
    "executed_amount": "0.1",
    "stop_price": "120",
    "created_at": 1536669674
}
```


### Cancel order

Cancel an order previously created by a user. Only orders with the `NEW` status can be canceled.

```
POST /v1/private/cancel-order
```

**REQUEST PARAMETRS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `order_id`  | Yes | A valid order_id returned by create-order

**RESPONSE FIELDS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `cancelled_order`  | Yes | A number, an internal unique identifier of an order


**Response data example**

```json
{
    "cancelled_order": 265167535
}
```

### Get all client orders

Returns the list of all user's orders of all statuses, not more than  `limit` at a time

**REQUEST PARAMETRS**

```
GET /v1/private/get-orders
```
| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `trade_pair`  | No | Valid trading pair name
| `status`  | No | Valid order status
| `limit`  | No | Max 500. Default 100
| `start_created_at`  | No | Unix timestamp in UTC timezone

**RESPONSE FIELDS**

The parameters are the same as for  `/v1/private/get-order`

**Response data example**

```json
[
    {
        "trade_pair": "BTC_USD",
        "side" : "SELL",
        "type": "LIMIT",
        "status": "NEW",
        "order_id": 265167539,
        "client_order_id": 544,
        "price": "128.45",
        "amount": "0.5",
        "executed_amount": "0",
        "created_at": 1536669674,
        "time_in_force": "GTC"
    },
    {
        "trade_pair": "BTC_USD",
        "side" : "BUY",
        "type": "LIMIT",
        "status": "FILLED",
        "order_id": 265167540,
        "price": "128.45",
        "amount": "0.5",
        "executed_amount": "0.5",
        "created_at": 1536669695,
        "done_at": 1536669698,
        "time_in_force": "GTC"
    }
]
```

### Cancel all orders

Cancels all orders with the  `NEW` status

```
POST /v1/private/cancel-all-orders
```

Returns the list `order_id` of all canceled orders


**Response data example**
```json
[
    265167535,
    265167534,
    265167015
]
```

## Trades

Returns the list of user's trades 

## List user trades

```
GET /v1/private/get-trades
```

**REQUEST PARAMETRS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `trade_pair`  | No | Valid trading pair name
| `limit`  | No | Max 500. Default 100
| `start`  | No | Unix timestamp in UTC timezone
| `order_id`  | No | Order identifier, within which trades are needed to be received

**RESPONSE FIELDS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `trade_pair`  | No | Valid trading pair name
| `side`  | Yes | BUY/SELL
| `order_id`  | Yes | A number, an internal unique identifier of an order
| `trade_id`  | Yes | A number, an internal unique identifier of a trader
| `price` | Yes | Decimal price value, at which a trade has been carried out
| `amount` | Yes | Decimal amount value
| `time`  | Yes | Unix timestamp in UTC timezone. Trade execution time 

**Response data example**

```json
[
    {
        "trade_pair": "BTC_USD",
        "side" : "SELL",
        "order_id": 265167539,
        "trade_id": 746837892,
        "price": "128.45",
        "amount": "0.5",
        "time": 1536669696
    },
    {
        "trade_pair": "BTC_USD",
        "side" : "BUY",
        "order_id": 265167540,
        "trade_id": 746837467,
        "price": "128.45",
        "amount": "0.5",
        "time": 1536669695
    }
]
```

## Account


### Get account balances

```
GET /v1/private/get-balances
```

Endpoint returns the dictionary, where keys are currencies and values are balances which include both the available amount of funds, and those on hold, which can not be used at the moment.

**Response data example**

```json
{
    "BTC": {"available": "1.2", "on_hold": "0"},
    "USD": {"available": "342.1", "on_hold": "200"},
    "LTC": {"available": "23.62", "on_hold": "0"}
}
```

## Payments

Payments via HTTPS API are only available for cryptocurrencies.

### Generate new deposit address

To top up a cryptocurrency account at Cryptology you can generate an address of the cryptocurrency wallet 

```
POST /v1/private/create-deposit-address
```

**REQUEST PARAMETRS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `currency`  | Yes | Valid currency name

**RESPONSE FIELDS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `wallet`  | Yes | Crypto address which can be used for depositing 

**Response data example:**

```json
{
    "wallet": "1BoatSLRHtKNngkdXEeobR76b53LETtpyT"
}
```

### Get already generated deposit address

To top up a cryptocurrency account you can also get an address that was previously generated. 
If you have not generated it earlier, it will be generated automatically 

```
GET /v1/private/get-deposit-address
```

**REQUEST PARAMETRS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `currency`  | Yes | Valid currency name

**RESPONSE FIELDS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `wallet`  | Yes | Crypto address which can be used for depositing 

**Response data example**

```json
{
    "wallet": "1BoatSLRHtKNngkdXEeobR76b53LETtpyT"
}
```

### Withdraw

Withdraws user's funds from the account to a specified address 

```
POST /v1/private/create-withdrawal
```

**REQUEST PARAMETRS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `currency`  | Yes | Valid currency name
| `amount`  | Yes | Decimal amount value
| `wallet`  | Yes | Valid wallet address


**RESPONSE FIELDS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `payment_id`  | Yes | Is a unique payment identifier.

**Response data example**

```json
{
    "payment_id": "1d4b2661-35a3-4b73-ae76-121bf38b088c"
}
```

### Get information about withdrawal

Returns the information about a previously made withdrawal 

```
GET /v1/private/get-withdrawal
```

**REQUEST PARAMETRS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `payment_id`  | Yes | Valid payment id

**RESPONSE FIELDS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `status`  | Yes | Withdrawal status
| `currency`  | Yes | Currency name
| `amount`  | Yes | Decimal amount value
| `transaction_info`  | No | Information about a withdrawal in the blockchain. Applicable only to withdrawals with the COMPLETE status
| `created_at` | Yes | Unix timestamp in UTC. Withdrawal initialization time
| `completed_at` | No | Unix timestamp in UTC. Withdrawal completion time. Applicable only to withdrawals with the COMPLETE status

**Response data example**

```json
{
    "status": "COMPLETE",
    "amount": "0.23",
    "currency": "BTC",
    "created_at": 1536669200,
    "completed_at": 1536669200,
    "transaction_info": {
        "blockchain_tx_ids": [
            "0x124129474b1dcbdb4e39436de49f7e5987f46dc4b8740966655718d7a1da699b"
        ]
    }
}
```

# Public

Public endpoints are available without authorization.

## List available trading pairs

Get available trading pairs

```
GET /v1/public/get-trade-pairs
```

**RESPONSE FIELDS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `trade_pair`  | Yes | Trading pair name
| `base_currency`  | Yes | Base currency
| `quoted_currency`  | Yes | Quoted currency


**Response data example**

```json
[
    {
      "trade_pair": "BTC_USD",
      "base_currency": "BTC",
      "quoted_currency": "USD"
    },
    {
      "trade_pair": "LTC_BTC",
      "base_currency": "LTC",
      "quoted_currency": "BTC"
    }
]
```

## Get order book

Returns the current order book for the specified pair in accordance with the bid and ask in a list format containing pairs "price" and  "amount".

```
GET /v1/public/get-order-book
```

**REQUEST PARAMETRS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `trade_pair`  | Yes | Valid trading pair name
| `type`  | Yes | AGGREGATED, BEST or FULL. Currently only AGGREGATED supported


**Response data example**

```json
{ "asks": [
      ["905.95", "0.1"],
      ["906.00", "0.05"],
      ["906.2", "0.2"]
  ],
  "bids": [
      ["905.45", "0.5"],
      ["904.00", "0.9"],
      ["902.00", "0.3"]
    ]
}
```

## List recent trades

```
GET /v1/public/get-trades
```

**REQUEST PARAMETRS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `trade_pair`  | Yes | Valid trading pair name
| `limit`  | No | Max 500. Default 100
| `start`  | No | Unix timestamp in UTC timezone

**RESPONSE FIELDS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `trade_id`  | Yes | A number, an internal unique identifier of a trade
| `price` | Yes | Decimal price value, at which a trade was executed
| `amount` | Yes | Decimal amount value
| `time`  | Yes | Unix timestamp in UTC timezone. Trade execution time 

**Response data example:**

```json
[
    {
        "trade_id": 45678878,
        "price": "128.45",
        "amount": "0.5",
        "time": 1536669632
    },
    {
        "trade_id": 45678879,
        "price": "128.80",
        "amount": "0.2",
        "time": 1536669632
    }
]
```

## List candles

Returns the list of candles according to the specified parameters. Each candle starts from the time rounded  to an interval.

```
GET /v1/public/get-candles
```

**REQUEST PARAMETRS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `trade_pair`  | Yes | Valid trading pair name
| `interval`  | Yes | Supported candles interval
| `start` | Yes | Start UNIX timestamp
| `end` | Yes | End UNIX timestamp in UTC timezone

Maximum data points that can be requested is 300 candles.

**RESPONSE FIELDS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `open` | Yes | Decimal price value, at which the first trade was executed within a candle period
| `high` | Yes | Maximum decimal price value, at which a trade was executed within a candle period 
| `low` | Yes | Minimum decimal price value, at which a trade was executed within a candle period
| `close` | Yes | Decimal price value, at which the last trade was executed within a candle period 
| `avg` | Yes | Average decimal price value of trades executed within a candle period 
| `base_volume` | Yes | Trading volume in a base currency within a candle period
| `time`  | Yes | Unix timestamp in UTC timezone. Candle start time 

**Response data example**

```json
[
    {
      "time": 1536669600,
      "open": "128.76",
      "high": "135.2",
      "low": "127",
      "close": "127",
      "avg": "132.76",
      "base_volume": "12"
    },
    {
      "time": 1536669660,
      "open": "127",
      "high": "130",
      "low": "127",
      "close": "127",
      "avg": "128.2",
      "base_volume": "3"
    }
]
```

## Get 24 hour statistics

Returns statistics within previous 24 hours.

```
GET /v1/public/get-24hrs-stat
```

**REQUEST PARAMETRS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `trade_pair`  | Yes | Valid trading pair name

**RESPONSE FIELDS**

| Param name      | Required | Description          
| :-------------: |:-------------:|:-------------|
| `open` | Yes | Decimal price value, at which the first trade was executed within a candle period
| `high` | Yes | Maximum decimal price value, at which a trade  was executed within a candle period
| `low` | Yes | Minimum decimal price value, at which a trade was executed within a candle period
| `base_volume` | Yes | Trading volume in a base currency within a candle period 

**Response data example**

```json
{
    "open": "128.76",
    "high": "135.2",
    "low": "128",
    "base_volume": "765.2"
}
```
