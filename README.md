
# Welcome to Cryptology community exchange documentation!

# Introduction

Welcome to Cryptology trader and developer documentation. These documents outline exchange functionality, market details, and APIs.

APIs are separated into two categories: trading and feed. Access to APIs require authentication and provide access to placing orders, provide market data and other account information. 

By accessing the Cryptology Data API, you agree to [Terms & Conditions](https://cryptology.com/legal/terms/)

# General information

## Matching orders

Cryptology market operates a first come - first serve, order matching. Orders are executed as received by the matching engine, from the older to newer received orders.

### Order Lifecycle

Valid orders sent to the matching engine are confirmed immediately and are in the received state. If an order executes against another order immediately, the order is considered done. An order can execute in part or whole. Any part of the order not filled immediately, will be considered open. Orders will stay in the open state until canceled or subsequently filled by new orders. Orders that are no longer eligible for matching (filled or canceled) are in the done state.

## Fees

### Trading Fees

Cryptology operates a maker-taker model. Orders which provide liquidity are charged different fees from orders taking liquidity. 

**Trading fees:**

Market maker: 0.02%

Market taker: 0.02%

**Deposits:**

Card deposits: 3.5%

Wire transfers: 0

**Withdrawals:**

BTC: 0.0005;

BCH, LTC: 0.0003;

ETH: 0,009

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

https://sandbox.cryptology.com

## Production

**Production URLs:**
For production use the following URLs.

Trading API:
wss://api.cryptology.com

Market Data API:
wss://marketdata.cryptology.com

**Website:**

https://cryptology.com

## Installation

``` {.sourceCode .bash}
pip install cryptology-client-python
```

## Usage

``` {.sourceCode .python3}
import asyncio
from datetime import datetime

from cryptology import ClientWriterStub, run_client

async def main() -> None:
    async def writer(ws: ClientWriterStub, state: Dict) -> None:
        client_order_id = 0
        while True:
            await asyncio.sleep(1)
            client_order_id += 1
            await ws.send_signed(
                sequence_id=sequence_id,
                payload={'@type': 'PlaceBuyLimitOrder',
                         'trade_pair': 'BTC_USD',
                         'amount': '2.3',
                         'price': '15000.1',
                         'client_order_id': 123 + client_order_id,
                         'ttl': 0
                        }
            )

    async def read_callback(ts: datetime, message_id: int, payload: dict) -> None:
        print(order, ts, message_id, payload)

    await run_client(
        access_key='YOUR ACCESS KEY',
        secret_key='YOUR SECRET KEY',
        ws_addr='wss://api.sandbox.cryptology.com',
        writer=writer,
        read_callback=read_callback,
        last_seen_message_id=-1
    )
```

## Protocol

## Handshake
Cliend sends message  with next format:

> ```json
> {
>    "access_key": "access key",
>    "secret_key": "secret key",
>    "last_seen_message_id": -1,
>    "version": 6,
>    "get_balances": true,
>    "get_order_books": true
> }
> ```
where `access_key` is a client access key,
      `secret_key` is a client secret key,
      `last_seen_message_id` is the last message id which client got from a server in previous sessions or -1,
      `protocol_version` is a version of protocol,
      `get_balances` and `get_order_books` are optional flags which ask the server to send user balances and/or order books (default false)


Server respond message with next format:

> ```json
> {
>    "last_seen_sequence": 100000,
>    "server_version": 6,
>    "state": {"order_books": {}}
>  }
>```
where `last_seen_sequence` is a last `sequense_id` which server received from client,
      `server_version` is a version of the server,
      `state` is optional field which can contain order_books and/or balances

## Messages

Request example

> ```json
> {
>     "sequence_id": 1212,
>     "data": {
>         "@type": "CancelOrder",
>         "order_id": 42
>     }
> }
> ```

There are following types of server response messages: MESSAGE, THROTTLING,
ERROR.

MESSAGE response example

> ```json
> {
>     "response_type": "MESSAGE",
>     "timestamp": 1533214317,
>     "message_id": 4763856,
>     "data":  {
>         "@type": "BuyOrderCancelled",
>         "order_id": 1,
>         "time": [
>             946684800,
>             0
>         ],
>         "trade_pair": "BTC_USD",
>         "client_order_id": 123
>     }
> }
> ```

THROTTLING response example

> ```json
> {
>     "response_type": "THROTTLING",
>     "overflow_level": 12800,
>     "sequence_id": 45
> }
> ```

ERROR response example

> ```json
> {
>     "response_type": "ERROR",
>     "error_type": "DUPLICATE_CLIENT_ORDER_ID",
>     "error_message": "Client order id 345 is already exists"
> }
> ```

Params:

| Name            | Type | Description                                                                                                                                                                                                                                                     | Required | Constraints                                          |
| --------------- | ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ---------------------------------------------------- |
| response\_type  | enum | Response type                                                                                                                                                                                                                                                   | Yes      | Can have following value: MESSAGE, THROTTLING, ERROR |
| sequence\_id    | int  | Unique identifier of performed operation. Auto processed by client library                                                                                                                                                                                      | Yes      |                                                      |
| message\_id     | int  | Is an incremental (but not necessarily sequential) value indicating message order on server and used by the client to skip processed events on reconnect                                                                                                        | Yes      |                                                      |
| timestamp       | int  | Time of operation performed                                                                                                                                                                                                                                     | Yes      | Valid timestamp                                      |
| data              | json | Body of message                                                                    | Yes |  |
| overflow\_level   | int  | Amount of orders the client should postpone sending to keep up with the rate limit | Yes |  |
| client\_order\_id | int  | Order id which sent by client                                                      | No  |  |
| error\_type       | enum | Error type `ERROR_TYPE`                                                            | Yes |  |
| error\_message    | str  | Error description                                                                  | Yes |  |

and `ERROR_TYPE` determines an error type:

- DUPLICATE\_CLIENT\_ORDER\_ID error
    :   `client_order_id` must be a unique field for each order created.
        `DUPLICATE_CLIENT_ORDER_ID` means that `client_order_id` in the
        sent message is not unique.

- INVALID\_PAYLOAD error
    :   All client messages must be in a valid JSON format and contain
        all the required fields. `INVALID_PAYLOAD` means that client
        sends an invalid JSON or any required parameter is not sent.

- UNKNOWN\_ERROR error
    :   Any other errors.

## Market data protocol

Market data is broadcasted via web socket. It is read only
and has no integrity checks aside of Web Socket built-in mechanisms.

### Messages

- `OrderBookAgg`
    :   aggregated order book for given symbol, recalculated after each
        order book change (most likely will be throttled to reasonable
        interval in future). May have empty dictionaries `buy_levels` or
        `sell_levels` in case of an empty order book. Both dictionaries
        use the price as a key and volume as a value. `current_order_id`
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

- `AnonymousTrade`
    :   a trade has taken place. `time` has two parts - integer seconds
        and integer milliseconds UTC. `maker_buy` shows if the maker was
        the buyer part.

```json
{
    "@type": "AnonymousTrade",
    "time": [1530093825, 0],
    "trade_pair": "BTC_USD",
    "current_order_id": 123456,
    "amount": "42.42",
    "price": "555",
    "maker_buy": false,
}
```

## Server messages

### Order lifecycle

After a place order message is received by Cryptology (TBD) the
following messages will be sent over web socket connection. All order
related messages are user specific (i.e. you can't receive any of
these messages for regular user or other user orders). The `time`
parameter is a list of two integers. The first one is a UNIX timestamp
in the UTC time zone. The second value is a number of microseconds.

- `BuyOrderPlaced`, `SellOrderPlaced`
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

- `BuyOrderAmountChanged`, `SellOrderAmountChanged`
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

- `BuyOrderCancelled`, `SellOrderCancelled`
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

- `BuyOrderClosed`, `SellOrderClosed`
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

- `OrderNotFound`
    :   attempt to cancel a non-existing order was made

```json
{
    "@type": "OrderNotFound",
    "order_id": 1
}
```

### Wallet

- `SetBalance`
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

- `InsufficientFunds`
    :   indicates that an account doesn't have enough funds to place an
        order

```json
{
    "@type": "InsufficientFunds",
    "order_id": 1,
    "currency": "USD"
}
```

- `DepositTransactionAccepted`
    :   indicates transaction information when depositing funds to the
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
        ]
    },
    "time": [
        946684800,
        0
    ]
}
```

- `WithdrawalTransactionAccepted`
    :   indicates transaction information when withdrawing funds from
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
        ]
    },
    "time": [
        946684800,
        0
    ]
}
```

### General

- `OwnTrade`
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

## Client messages

### Order placement

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

all order placement messages share the same structure

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

`client_order_id` is a tag to relate server messages to client ones.
`ttl` is the time the order is valid for. Measured in seconds (with 1
minute granularity). 0 means valid forever.

### Order cancelation

- `CancelOrder`
    :   cancel any order

```json
{
    "@type": "CancelOrder",
    "order_id": 42
}
```

- `CancelAllOrders`
    :   cancel all active orders opened by the client

```json
{
    "@type": "CancelAllOrders"
}
```


