# Go SDK for KuCoin API
> The detailed document [https://docs.kucoin.com](https://docs.kucoin.com), in order to receive the latest API change notifications, please `Watch` this repository.

[![Latest Version](https://img.shields.io/github/release/Kucoin/kucoin-go-sdk.svg)](https://github.com/Kucoin/kucoin-go-sdk/releases)
[![GoDoc](https://godoc.org/github.com/Kucoin/kucoin-go-sdk?status.svg)](https://godoc.org/github.com/Kucoin/kucoin-go-sdk)
[![Build Status](https://travis-ci.org/Kucoin/kucoin-go-sdk.svg?branch=master)](https://travis-ci.org/Kucoin/kucoin-go-sdk)
[![Go Report Card](https://goreportcard.com/badge/github.com/Kucoin/kucoin-go-sdk)](https://goreportcard.com/report/github.com/Kucoin/kucoin-go-sdk)
[![Sourcegraph](https://sourcegraph.com/github.com/Kucoin/kucoin-go-sdk/-/badge.svg)](https://sourcegraph.com/github.com/Kucoin/kucoin-go-sdk?badge)
<!-- [![Total Lines](https://tokei.rs/b1/github/Kucoin/kucoin-go-sdk)](https://github.com/Kucoin/kucoin-go-sdk) -->


## Install

```bash
go get github.com/Kucoin/kucoin-go-sdk
```

## Usage

### Choose environment

| Environment | BaseUri |
| -------- | -------- |
| *Production* `DEFAULT` | https://openapi-v2.kucoin.com https://api.kucoin.com https://api.kcs.top |
| *Sandbox* | https://openapi-sandbox.kucoin.com |

### Create ApiService

```go
s := kucoin.NewApiService( 
	// kucoin.ApiBaseURIOption("https://openapi-v2.kucoin.com"), 
	kucoin.ApiKeyOption("key"),
	kucoin.ApiSecretOption("secret"),
	kucoin.ApiPassPhraseOption("passphrase"),
)

// Or add these options into the environmental variable
// Bash: 
// export API_BASE_URI=https://openapi-v2.kucoin.com
// export API_KEY=key
// export API_SECRET=secret
// export API_PASSPHRASE=passphrase
// s := NewApiServiceFromEnv()
```

### Debug mode & logging

```go
// Require package github.com/sirupsen/logrus
// Debug mode will record the logs of API and WebSocket to files in the directory "kucoin.LogDirectory" according to the minimum log level "kucoin.LogLevel".
kucoin.DebugMode = true
// Or export API_DEBUG_MODE=1

// Logging in your code
// kucoin.SetLoggerDirectory("/tmp")
// logrus.SetLevel(logrus.DebugLevel)
logrus.Debugln("I'm a debug message")
```

### Examples
> See the test case for more examples.

#### Example of API `without` authentication

```go
rsp, err := s.ServerTime()
if err != nil {
    log.Printf("Error: %s", err.Error())
    // Handle error
    return
}

var ts int64
if err := rsp.ReadData(&ts); err != nil {
    // Handle error
    return
}
log.Printf("The server time: %d", ts)
```

#### Example of API `with` authentication

```go
// Without pagination
rsp, err := s.Accounts("", "")
if err != nil {
    // Handle error
    return
}

as := kucoin.AccountsModel{}
if err := rsp.ReadData(&as); err != nil {
    // Handle error
    return
}

for _, a := range as {
    log.Printf("Available balance: %s %s => %s", a.Type, a.Currency, a.Available)
}
```

```go
// Handle pagination
rsp, err := s.Orders(map[string]string{}, &kucoin.PaginationParam{CurrentPage: 1, PageSize: 10})
if err != nil {
    // Handle error
    return
}

os := kucoin.OrdersModel{}
pa, err := rsp.ReadPaginationData(&os)
if err != nil {
    // Handle error
    return
}
log.Printf("Total num: %d, total page: %d", pa.TotalNum, pa.TotalPage)
for _, o := range os {
    log.Printf("Order: %s, %s, %s", o.Id, o.Type, o.Price)
}
```

#### Example of WebSocket feed
> Require package [gorilla/websocket](https://github.com/gorilla/websocket)

```bash
go get github.com/gorilla/websocket github.com/pkg/errors
```

```go
rsp, err := s.WebSocketPublicToken()
if err != nil {
    // Handle error
    return
}

tk := &kucoin.WebSocketTokenModel{}
if err := rsp.ReadData(tk); err != nil {
    // Handle error
    return
}

c := s.NewWebSocketClient(tk)

mc, ec, err := c.Connect()
if err != nil {
    // Handle error
    return
}

ch1 := kucoin.NewSubscribeMessage("/market/ticker:KCS-BTC", false)
ch2 := kucoin.NewSubscribeMessage("/market/ticker:ETH-BTC", false)
uch := kucoin.NewUnsubscribeMessage("/market/ticker:ETH-BTC", false)

if err := c.Subscribe(ch1, ch2); err != nil {
    // Handle error
    return
}

var i = 0
for {
    select {
    case err := <-ec:
        c.Stop() // Stop subscribing the WebSocket feed
        log.Printf("Error: %s", err.Error())
        // Handle error
        return
    case msg := <-mc:
        // log.Printf("Received: %s", kucoin.ToJsonString(m))
        t := &kucoin.TickerLevel1Model{}
        if err := msg.ReadData(t); err != nil {
            log.Printf("Failure to read: %s", err.Error())
            return
        }
        log.Printf("Ticker: %s, %s, %s, %s", msg.Topic, t.Sequence, t.Price, t.Size)
        i++
        if i == 5 {
            log.Println("Unsubscribe ETH-BTC")
            if err = c.Unsubscribe(uch); err != nil {
                log.Printf("Error: %s", err.Error())
                // Handle error
                return
            }
        }
        if i == 10 {
            log.Println("Subscribe ETH-BTC")
            if err = c.Subscribe(ch2); err != nil {
                log.Printf("Error: %s", err.Error())
                // Handle error
                return
            }
        }
        if i == 15 {
            log.Println("Exit subscription")
            c.Stop() // Stop subscribing the WebSocket feed
            return
        }
    }
}
```

### API list

<details>
<summary>Account</summary>

| API | Authentication | Description |
| -------- | -------- | -------- |
| ApiService.CreateAccount() | YES | https://docs.kucoin.com/#create-an-account |
| ApiService.Accounts() | YES | https://docs.kucoin.com/#list-accounts |
| ApiService.Account() | YES | https://docs.kucoin.com/#get-an-account |
| ApiService.SubAccountUsers() | YES | https://docs.kucoin.com/#get-user-info-of-all-sub-accounts |
| ApiService.SubAccounts() | YES | https://docs.kucoin.com/#get-the-aggregated-balance-of-all-sub-accounts-of-the-current-user |
| ApiService.SubAccount() | YES | https://docs.kucoin.com/#get-account-balance-of-a-sub-account |
| ApiService.AccountLedgers() | YES | https://docs.kucoin.com/#get-account-ledgers |
| ApiService.AccountHolds() | YES | https://docs.kucoin.com/#get-holds |
| ApiService.InnerTransfer() | YES | `DEPRECATED` https://docs.kucoin.com/#inner-transfer |
| ApiService.InnerTransferV2() | YES | https://docs.kucoin.com/#inner-transfer |
| ApiService.SubTransfer() | YES | https://docs.kucoin.com/#transfer-between-master-account-and-sub-account |

</details>

<details>
<summary>Deposit</summary>

| API | Authentication | Description |
| -------- | -------- | -------- |
| ApiService.CreateDepositAddress() | YES | https://docs.kucoin.com/#create-deposit-address |
| ApiService.DepositAddresses() | YES | https://docs.kucoin.com/#get-deposit-address |
| ApiService.V1Deposits() | YES | https://docs.kucoin.com/#get-v1-historical-deposits-list |
| ApiService.Deposits() | YES | https://docs.kucoin.com/#get-deposit-list |

</details>

<details>
<summary>Fill</summary>

| API | Authentication | Description |
| -------- | -------- | -------- |
| ApiService.Fills() | YES | https://docs.kucoin.com/#list-fills |
| ApiService.RecentFills() | YES | https://docs.kucoin.com/#recent-fills |

</details>

<details>
<summary>Order</summary>

| API | Authentication | Description |
| -------- | -------- | -------- |
| ApiService.CreateOrder() | YES | https://docs.kucoin.com/#place-a-new-order |
| ApiService.CancelOrder() | YES | https://docs.kucoin.com/#cancel-an-order |
| ApiService.CancelOrders() | YES | https://docs.kucoin.com/#cancel-all-orders |
| ApiService.V1Orders() | YES | https://docs.kucoin.com/#get-v1-historical-orders-list |
| ApiService.Orders() | YES | https://docs.kucoin.com/#list-orders |
| ApiService.Order() | YES | https://docs.kucoin.com/#get-an-order |
| ApiService.RecentOrders() | YES | https://docs.kucoin.com/#recent-orders |

</details>

<details>
<summary>WebSocket Feed</summary>

| API | Authentication | Description |
| -------- | -------- | -------- |
| ApiService.WebSocketPublicToken() | NO | https://docs.kucoin.com/#apply-connect-token |
| ApiService.WebSocketPrivateToken() | YES | https://docs.kucoin.com/#apply-connect-token |
| ApiService.NewWebSocketClient() | - | https://docs.kucoin.com/#websocket-feed |

</details>

<details>
<summary>Withdrawal</summary>

| API | Authentication | Description |
| -------- | -------- | -------- |
| ApiService.WithdrawalQuotas() | YES | https://docs.kucoin.com/#get-withdrawal-quotas |
| ApiService.V1Withdrawals() | YES | https://docs.kucoin.com/#get-v1-historical-withdrawals-list |
| ApiService.Withdrawals() | YES | https://docs.kucoin.com/#get-withdrawals-list |
| ApiService.ApplyWithdrawal() | YES | https://docs.kucoin.com/#apply-withdraw |
| ApiService.CancelWithdrawal() | YES | https://docs.kucoin.com/#cancel-withdrawal |

</details>

<details>
<summary>Currency</summary>

| API | Authentication | Description |
| -------- | -------- | -------- |
| ApiService.Currencies() | NO | https://docs.kucoin.com/#get-currencies |
| ApiService.Currency() | NO | https://docs.kucoin.com/#get-currency-detail |
| ApiService.Prices() | NO | https://docs.kucoin.com/#get-fiat-price |

</details>

<details>
<summary>Symbol</summary>

| API | Authentication | Description |
| -------- | -------- | -------- |
| ApiService.Symbols() | NO | https://docs.kucoin.com/#get-symbols-list |
| ApiService.TickerLevel1() | NO | https://docs.kucoin.com/#get-ticker |
| ApiService.Tickers() | NO | https://docs.kucoin.com/#get-all-tickers |
| ApiService.AggregatedPartOrderBook() | NO | https://docs.kucoin.com/#get-part-order-book-aggregated |
| ApiService.AggregatedFullOrderBook() | NO | https://docs.kucoin.com/#get-full-order-book-aggregated |
| ApiService.AtomicFullOrderBook() | NO | https://docs.kucoin.com/#get-full-order-book-atomic |
| ApiService.TradeHistories() | NO | https://docs.kucoin.com/#get-trade-histories |
| ApiService.KLines() | NO | https://docs.kucoin.com/#get-klines |
| ApiService.Stats24hr() | NO | https://docs.kucoin.com/#get-24hr-stats |
| ApiService.Markets() | NO | https://docs.kucoin.com/#get-market-list |

</details>

<details>
<summary>Time</summary>

| API | Authentication | Description |
| -------- | -------- | -------- |
| ApiService.ServerTime() | NO | https://docs.kucoin.com/#server-time |

</details>

## Run tests

```shell
# Add your API configuration items into the environmental variable first
export API_BASE_URI=https://openapi-v2.kucoin.com
export API_KEY=key
export API_SECRET=secret
export API_PASSPHRASE=passphrase

# Run tests
go test -v
```

## License

[MIT](LICENSE)
