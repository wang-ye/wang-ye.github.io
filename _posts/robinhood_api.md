---
layout: post
title:  "Robinhood Crypto APIs"
date:   2018-04-04 20:49:03 -0800
---

Recently Robinhood launched the [crypto trading](https://crypto.robinhood.com/) for Bitcoin and ETH. I did some research and uncovered a couple of Robinhood Crypto trading APIs, including quotes, currrency pairs and price history. However, I still could not decode the buy/sell APIs. In this post, I listed my findings and the unsuccessful trials for the trade APIs. Please note that, the best resource to discover the Crypto APIs is actually the [Robinhood Web](https://robinhood.com/), although Robinhood does not support web Crypto trading!

## List Currency Pairs

With cURL, you can get all currency pairs using the endpoint `nummus.robinhood.com/currency_pairs`. Note the use `-L` to conduct the necessary redirection.

```bash
curl -L --request GET https://nummus.robinhood.com/currency_pairs  -H "Accept: application/json"
```

The returned value is in JSON format. It contains multiple currency pairs including BTC-USD, ETH-USD and various non-tradable pairs such as LTC-USD. For simplicity, we only show the BTC-USD pair here. The id for this pair `3d961844-d360-45fc-989b-f6fca761d511` will be used later.

```json
{
"results":
    [
        {
            "asset_currency":{
                "code":"BTC","id":"d674efea-e623-4396-9026-39574b92b093","increment":"0.000000010000000000","name":"Bitcoin","type":"cryptocurrency"},
            "display_only":false,
            "id":"3d961844-d360-45fc-989b-f6fca761d511",
            "max_order_size":"5.0000000000000000",
            "min_order_size":"0.000010000000000000",
            "min_order_price_increment":"0.010000000000000000",
            "min_order_quantity_increment":"0.000000010000000000",
            "name":"Bitcoin to US Dollar",
            "quote_currency":{
                "code":"USD","id": "1072fc76-1862-41ab-82c2-485837590762",
                "increment":"0.010000000000000000","name":"US Dollar","type":"fiat"},
            "symbol":"BTC-USD",
            "tradability":"tradable"
        }
    ]
}
```

## Authorization
Most of the Crypto APIs require Oath2 authentications. To get the bearer, first get token by running:

```bash
curl -v https://api.robinhood.com/api-token-auth/ \
   -H "Accept: application/json" \
   -d "username={username}&password={password}"
```

This gives you something like `{"token":"abcdefg"}`. Now let's trade for the  Bearer token:

```bash
curl -v https://api.robinhood.com/oauth2/migrate_token/ \
   -H "Accept: application/json" \
   -H "Authorization: Token abcdefg" \
   -d ""
```

You will get the bear token like this: 
```json
{
    "token_type":"Bearer",
    "access_token":"xxxaccess",
    "expires_in":300,
    "refresh_token":"refresh_token",
    "scope":"web_limited"
}
```

The bearer token has a 300 second expiration time. For many APIs involving cryptos, we need to pass bearer token in the Authorization header.

## Historical Market Data
Robinhood provides the historical market data. 

```bash
curl -v "https://api.robinhood.com/marketdata/forex/historicals/3d961844-d360-45fc-989b-f6fca761d511/?interval=5minute&span=day&bounds=24_7" \
   -H "Accept: application/json" \
   -H "Authorization: Bearer xxxaccess"
```

## Quote Data
The quote data can be accessed using the following endpoint:

```bash
curl -v https://api.robinhood.com/marketdata/forex/quotes/3d961844-d360-45fc-989b-f6fca761d511/ \
   -H "Accept: application/json" \
   -H "Authorization: Bearer xxxaccess"
```

## Trade Endpoints
Unfortunately, I could not crack the trade endpoints even after trying different approaches. My first attempt was enumerating the parameters and endpoints I could think of:

```python
endpoints = [
    "https://api.robinhood.com/orders/",
    "https://api.robinhood.com/trade/forex/",
    "https://api.robinhood.com/order/forex/",
    "https://api.robinhood.com/orders/forex/",
]

fixed_payloads = [('price', 100), ('symbol', 'BTC-USD'), ('side', 'buy'), ('type', 'market')]

flex_payloads = [
    (['quantity', 'amount'], [100]),
    (['trigger', None], ['immediate']),
    (['time_in_force', None], ['gfd']),
    (['account', None], ['https://api.robinhood.com/accounts/5UG65498/']),
    (['instrument', None], [unquote('https://nummus.robinhood.com/currency_pairs/?symbol=BTC')])
]

def all_flex_candidates():
    flex_all = []
    for pair in flex_payloads:
        flex_all.append(list(itertools.product(*pair)))
    final_flex_payloads = []
    for candidate in itertools.product(*flex_all):
        filtered_candidate = []
        for x in candidate:
            if x[0] and x[1]:
                filtered_candidate.append(x)
        final_flex_payloads.append(filtered_candidate)
    return final_flex_payloads


flex_payload_candidates = all_flex_candidates()
final_payloads = []
for candidate in flex_payload_candidates:
    candidate.extend(fixed_payloads)
    final_payloads.append(candidate)


for payload in final_payloads:
    for endpoint in endpoints:
        print('data = ', payload)
        res0 = self.session.post(endpoint, data=payload, timeout=5)
        print(res0.status_code)
        if res0.status_code == 200:
            return
     print(res0.reason)
```

Another approach I tried was Robinhood Web. It states clearly Crypto trading is not currently supported,, but in the near future Robinhood will support it. So just wait! I also used Postman to conduct a MITM attack, and intercept the API endpoints in the middle. To achieve this, you can set up a Postman proxy using your own machine, and route App requests to this proxy. Then, you can intercept the request in the middle. Unluckily, this still did not work, most likely it is because Robinhood does some extra authentication and check whether the connecting server is known. The last approach I tested was accessing Robinhood from Nox simulator and capture the actions/responses. It failed as Crypto access was not supported in the simulator. I am am still waiting for Robinhood support on this issue.

## Summary
In this post I listed some Robinhood Crypto findings and some failed attempt to crack the buy/sell APIs. Hope this helps for your trading journey.