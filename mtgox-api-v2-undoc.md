# Unofficial Documentation for MtGox's HTTP API v2

---

Tips and donations: `1VvKzcFibZdMyQdbXpSYcyoYvBsLLodDa`

**If you just want information on each method, go [here](#contents)**

### Sections

1. [Background](#background)
2. [General Information](#general-information)
	* [Currencies](#currencies)
	* [Security](#security)
	* [Requests](#requests)
3. [Methods](#methods)
	* [Glossary](#glossary)
	* [Contents](#contents)
	* [Money](#money)
	* [Security](#security-1)
	* [Others](#others)

## Background

I was looking to start building a trading bot using MtGox's API before I started investing in bitcoins, and even though version 1 of the API is available and well-documented, it seems that it will soon be redundant, just as version 0 is, with the advent of version 2. However, there is next to no documentation for version 2, so I have started this project to collect and compile an unofficial documentation for those who need it.

Not all the information may be 100% accurate or complete due to the limited information available, and detective work needed, but hopefully it will be suitably reliable for general use. In addition, I hope that this document will become increasingly accurate in the future as more official information and experience is gained. Currently, there is information on most of the actual trading methods, and there is detailed information on how to access the API. 

Upon completion, each method should give a brief description of its use, a table of arguments if applicable, one or two example requests, and brief but detailed bullet points of any useful information and peculiarities with regard to how the method is used in practice. I also plan to 

Thank you, and good luck in developing your v2 programs!

## General Information

Each request is posted over HTTPS to MtGox's servers, with a generic URL format, e.g:

<https://data.mtgox.com/api/2/BTCUSD/money/ticker>

Each request path needs a currency pair, the first currency being `BTC`, and the second one of the currencies specified below in the currencies section. The currency pair is followed by an API method in URL path form, all of which are currently found in the `money` category.

Each request also needs to be signed using an API key and an HMAC hash constructed from an API secret and the post data.

### Currencies

Symbol | Name | Divisions | SF
--- | --- | ----- | ---
BTC | Bitcoin | 100,000,000 | 1e8
USD | US Dollar | 100,000 | 1e5
GBP | Great British Pound | 100,000 | 1e5
EUR | Euro | 100,000 | 1e5
JPY | Japanese Yen | 1000 | 1e3
AUD | Australian Dollar | 100,000 | 1e5
CAD | Canadian Dollar | 100,000 | 1e5
CHF | Swiss Franc | 100,000 | 1e5
CNY | Chinese Yuan | 100,000 | 1e5
DKK | Danish Krone | 100,000 | 1e5
HKD | Hong Kong Dollar | 100,000 | 1e5
PLN | Polish Złoty | 100,000 | 1e5
RUB | Russian Rouble | 100,000 | 1e5
SEK | Swedish Krona | 1000 | 1e3
SGD | Singapore Dollar | 100,000 | 1e5
THB | Thai Baht | 100,000 | 1e5

Note the divisions column (the SF column is the same, but gives each in standard form with the number of 0s). When communicating with the MtGox server, you will often have the opportunity to work with ints or floats. The float values are deprecated however, and ints are recommended as they are precise. The divisions tells you what units each of these values is in. So if, for example, you request a bitcoin value, and get a response of `{'value_int': 69655509977}`, then the amount in bitcoins is `69655509977/1e8`, or `696.55509977 BTC`. Of course there is no requirement that you perform these conversions, but you should be aware of the differences in value.

### Security

In order to use the API, you must obtain an API key and secret. To do this, login to <https://mtgox.com/security>, and open the `Advanced API Key Creation` section. You can give your API key a name, and specify which permissions (rights) it will have. Once you have customised it to your liking, click `Create Key`. Make sure to copy down your API secret, as the website will not show you it again after this point.

The API keys you have created are shown under `Current API Keys` after you refresh the page. From here, you can retrieve your API key if you forget it, and you can change what rights it has at any time. If you believe it has been comprised, you should **immediately revoke the api key by clicking on the red cross**

Your API key and secret will be used to authenticate with MtGox's servers. This is achieved via the use of two HTTP headers: `Rest-Key` and `Rest-Sign`

`Rest-Key` is your API key, `Rest-Sign` is an HMAC hash constructed from your API secret, your API method path, your post data, and uses the SHA-512 algorithm. The `Rest-Sign` is constructed as follows:

* Your API secret has been base-64 encoded, decode it
* Join your method path, the NULL `\0` character, and your post data into a string
* Generate an HMAC hash string, using your decoded secret as the secret, SHA-512 as the hash function, and your constructed string as the message
* Encode this HMAC string using base-64

Here is an example python implementation of all the above:

	import hmac, base64, hashlib, urllib2
	base = 'https://data.mtgox.com/api/2/'
	
	def makereq(key, secret, path, data):
		hash_data = path + chr(0) + data
		secret = base64.b64decode(secret)
		sha512 = hashlib.sha512
		hmac = str(hmac.new(secret, hash_data, sha512))
		
		header = {
			'User-Agent': 'My-First-Trade-Bot',
			'Rest-Key': key,
			'Rest-Sign': base64.b64encode(hmac),
			'Accept-encoding': 'GZIP',
		}
		
		return urllib2.Request(base + path, data, header)		

You can then perform the request with the following:

	post_data = 'nonce=123'
	request = makreq('abc123..', 'aBc7/+..', 'BTCUSD/money/ticker', post_data)
	response = urllib2.urlopen(request, post_data)
	# if gzip encoding, decode
	# try to decode json into dictionary
	# raise exception if response contains error key

**N.B.** if you choose to include `GZIP` as an accepted encoding, you will have to check if the content-encoding of the response is `'gzip'`, and will then have to decode it.

### Requests

Each request consists of a path, any arguments, and a nonce value. The different methods you can use are defined by their paths and the arguments they accept. The path is the part of the HTTP URL after the base.

**Base**: <https://data.mtgox.com/api/2/> <br/>
**Path**: [BTCUSD/money/ticker](https://data.mtgox.com/api/2/BTCUSD/money/ticker)

The nonce value is important to prevent duplicated requests. In short, each request should be accompanied by a unique nonce value in the post data, and this value should be an integer and larger than any value sent previously. One way to accomplish this is by using millisecond or microsecond unix time, e.g. if your computer supports microsecond resolution:

	import time
	def nonce():
		return str(int(time.time() * 1e6))

This should be added to your argument list, and then submitted using url form encoding, i.e.:

	nonce=123&type=bid&amount_int=765000000&price_int=6061000

Some requests don't need any post data or HMAC authentication, such as the BTCUSD/money/ticker example used, and can be easily obtained in a web browser, but in general, you should always authenticate with a valid API key, and make sure that the key has the necessary rights.

Note that the API will ignore any extraneous arguments.

## Methods

**Disclaimer:** Official documentation for the methods below, at the time of writing, is limited or even non-existent, and the following information has been pooled from multiple sources, found by trial and error, or extrapolated. No responsibility is taken by the author for any incorrect or incomplete information, nor for any unintended effects it may cause. _Use the information below at your own risk._

---

Each of the methods described below consists of a table of key/value pairs if there are any, which are the known acceptable arguments, a sample request, and some brief notes.

### Glossary

Term | Definition
---- | ----
auxiliary currency | The currency specified, other than `BTC`, e.g. the auxiliary currency for the pair `BTCUSD` would be `USD`.
integer value | A currency value given in terms of the minimum division, for example `$ 2.37` would be `237000`. See the currency section for more information. Note, that even though these values are _ostensibly_ meant to be integers, sometimes a float may be returned, for example for small order quotes, so you should round these values if necessary.
bid | A `bid` order typically refers to a buy order for BTC using the auxiliary currency.
ask | An `ask` order typically refers to a sell order of BTC for the auxiliary currency.

#### Code

`**Currency Object**` refers to a JSON block of the form:

```
{
	currency: "BTC",
	display: "17.96800010\u00a0BTC",
	display_short: "17.97\u00a0BTC",
	value: "17.96800010",
	value_int: "1796800010"
}
```

#### Contents

* [money](#money)
	* [info](#moneyinfo)
	* [idkey](#moneyidkey)
	* [orders](#moneyorders)
	* order
		* [quote](#moneyorderquote)
		* [add](#moneyorderadd)
		* [cancel](#moneyordercancel)
		* [result](#moneyorderresult)
		* [lag](#moneyorderlag)
	* [currency](#moneycurrency)
	* [ticker](#moneyticker)
	* [hotp_gen](#moneyhotp_gen)
	* [quote](#moneyquote)
	* [trades](#moneytrades)
		* [fetch](#moneytradesfetch)
		* [cancelled](#moneytradescancelled)
	* [cancelledtrades](#moneycancelledtrades)
	* [depth](#moneydepth)
		* [fetch](#moneydepthfetch)
		* [full](#moneydepthfull)
	* [fulldepth](#moneyfulldepth)
* [security](#security-1)
	* hotp
		* [gen](#securityhotpgen)
* [others](#other-methods)

### Money

#### `money/info`

This method provides a lot of information about your account, the example given below is not actual sample data (as this would be personally identifying!), but is a skeleton of the data you can expect:

1 **Request:** `BTCUSD/money/info` <br/>
**Response:**

	{
		data: {
			Created: "yyyy-mm-dd hh:mm:ss",
			Id: "abc123",
			Index: "123",
			Language: "en_US",
			Last_Login: "yyyy-mm-dd hh:mm:ss",
			Login: "username",
			Monthly_Volume:                   **Currency Object**,
			Rights: ['deposit', 'get_info', 'merchant', 'trade', 'withdraw'],
			Wallets: {
				BTC: {
					Balance:                  **Currency Object**,
					Daily_Withdraw_Limit:     **Currency Object**,
					Max_Withdraw:             **Currency Object**,
					Monthly_Withdraw_Limit: null,
					Open_Orders:              **Currency Object**,
					Operations: 0,
				},
				USD: {
					Balance:                  **Currency Object**,
					Daily_Withdraw_Limit:     **Currency Object**,
					Max_Withdraw:             **Currency Object**,
					Monthly_Withdraw_Limit:   **Currency Object**,
					Open_Orders:              **Currency Object**,
					Operations: 0,
				},
				JPY:{...}, EUR:{...},
				// etc, depends what wallets you have
			},
		},
		result: success
	}

* As of writing, I do not have any open orders, so I cannot be sure of whether `Open_Orders` will be an array or have a different structure if you do have them or not, so it may be worth experimenting to see what happens if this is crucial to your application.
* You do not need to include a currency for this method

[^ Back to contents](#contents)

#### `money/idkey`

Returns an `idKey` used to subscribe to a user's private updates in the websocket API[^1]. The key is valid for 24 hours. Without any arguments, it will return your personal `idKey`, there does not appear to be a way to obtain other user's keys, and this may be intentional in order to protect their privacy.

1 **Request:** `BTCUSD/money/idkey` <br/>
**Response:** `{"result":"success","data":"abc+\123=.."}`

* You do not need to include a currency for this method

[^ Back to contents](#contents)

#### `money/orders`

Get information on your current orders

1 **Request:** `BTCUSD/money/orders` <br/>
**Response:** `{"result":"success","data":[]}`

* Again, I have no open orders to test here, however they may have the following format (you should verify this first before implementing as such):

```
{
	oid: "abc123-def456-..",
	currency: "USD",
	item: "BTC",
	type: "ask",
	status: "open",
	date: "1364691201",
	priority: "1364691201933187",
	actions: [],
	amount:         **Currency Object**,
	valid-amount:   **Currency Object**,
	invalid-amount: **Currency Object**,
	price:          **Currency Object**
}
```

* The `valid-amount` is the amount of the transaction currently funded, the `invalid-amount` is that which is unfunded, and the `amount` is the total of these
* Statuses may be: `pending`, `executing`, `post-pending`, `open`, `stop`, and `invalid`
* It seems priority is a unix timestamp with microsecond precision
* You do not need to include a currency for this method

[^ Back to contents](#contents)

#### `money/currency`

Get information for a given currency

1 **Request:** `BTCUSD/money/currency` <br/>
**Response:**

```
{
	"result":"success",
	"data": {
		"currency":"USD",
		"name":"Dollar",
		"symbol":"$",
		"decimals":"5",
		"display_decimals":"2",
		"symbol_position":"before",
		"virtual":"N",
		"ticker_channel":"abc123-def456",
		"depth_channel":"abc123-def456"
	}
}
```

* There does not seem to be a way to get similar information for BTC using this API
* The most important parts of this data is available in the currency section above, however it may be useful to use this method if you think the information may change
* You may also need this data if you don't want to hard code the way you display currencies, or if you wish to track the currency using the websocket API[^1].

[^ Back to contents](#contents)

#### `money/ticker`

Get the most recent information for a currency pair

1 **Request:** `BTCUSD/money/ticker` <br/>
**Response:**

```
{
	"result":"success",
	"data": {
		"high":       **Currency Object - USD**,
		"low":        **Currency Object - USD**,
		"avg":        **Currency Object - USD**,
		"vwap":       **Currency Object - USD**,
		"vol":        **Currency Object - BTC**,
		"last_local": **Currency Object - USD**,
		"last_orig":  **Currency Object - ???**,
		"last_all":   **Currency Object - USD**,
		"last":       **Currency Object - USD**,
		"buy":        **Currency Object - USD**,
		"sell":       **Currency Object - USD**,
		"now":        "1364689759572564"
	}
}
```

* `vwap` is the [volume-weighted average price](http://en.wikipedia.org/wiki/Volume-weighted_average_price)
* `last_local` is the last trade in your selected auxiliary currency
* `last_orig` is the last trade (any currency)
* `last_all` is that last trade converted to the auxiliary currency
* `last` is the same as `last_all`
* `now` is the unix timestamp, but with a resolution of 1 microsecond

[^ Back to contents](#contents)

#### `money/hotp_gen`
Deprecated - Use [`security/hotp/gen`](#securityhotpgen)

[^ Back to contents](#contents)

#### `money/quote`
Deprecated - Use [`money/order/quote`](#moneyorderquote)

[^ Back to contents](#contents)

#### `money/trades`
Deprecated - Use [`money/trades/fetch`](#moneytradesfetch)

[^ Back to contents](#contents)

#### `money/cancelledtrades`
Deprecated - Use [`money/trades/cancelled`](#moneytradescancelled)

[^ Back to contents](#contents)

#### `money/depth`
Deprecated - Use [`money/depth/fetch`](#moneydepthfetch)

[^ Back to contents](#contents)

#### `money/fulldepth`
Deprecated - Use [`money/depth/full`](#moneydepthfull)

[^ Back to contents](#contents)

---

#### `money/order/quote`

Get an up-to-date quote for a bid or ask transaction

key | value
--- | ---
type | `bid` or `ask`
amount | amount of BTC to buy or sell, as an integer

1 **Request:** `BTCUSD/money/order/quote` ~ `type=bid&amount=100000000` <br/>
**Response:** `{"result":"success","data":{"amount":9197999}}`

2 **Request:** `BTCUSD/money/order/quote` ~ `type=ask&amount=23769001` <br/>
**Response:** `{"result":"success","data":{"amount":2179142.24937}}`

* This method differs from most, as the currency values are _always_ given as integers
* As noted in the glossary, the api isn't strict about giving integers, see the second example - the quote for selling `0.23769001 BTC` is given as `$ 21.7914224937`, more digits than are expected for a USD value, so you may need to round this data to the nearest integer
* Regardless of whether your order is `bid` or `ask`, your input should be the quantity of bitcoins and the response will be in the auxiliary currency 

[^ Back to contents](#contents)

#### `money/order/add`

Creates a currency trade order

key | value
--- | ---
type | `bid` or `ask`
amount_int | amount of BTC to buy or sell, as an integer
price_int | The price per bitcoin in the auxiliary currency, as an integer
_amount_ | Deprecated - Use `amount_int`
_price_ | Deprecated - Use `price_int`

1 **Request:** `BTCUSD/money/order/add` ~ `type=bid&amount_int=50000000&price_int=9250000` <br/>
**Response:** 

* The amount is always given in BTC
* Price is always in the auxiliary currency, and is per bitcoin
* If you omit the price, your trade will be treated as a market order, and completed at the current market price

[^ Back to contents](#contents)

#### `money/order/cancel`

Cancel a trade order

key | value
--- | ---
order | The `id` of the order you wish to cancel

1 **Request:** `BTCUSD/money/order/cancel` ~ `order=` <br/>
**Response:**

* It appear that you do not need to specify what type of order you are cancelling, just the `id`
* It goes without saying that you cannot cancel an order that has already completed
* You *may*, however, be able to cancel a partially completed order, I am not sure though

[^ Back to contents](#contents)

#### `money/order/result`

Get information on a trade order

key | value
--- | ---
type | `bid` or `ask`
order | The `id` of the order

1 **Request:** `BTCUSD/money/order/result` ~ `type=&order=` <br/>
**Response:**

[^ Back to contents](#contents)

#### `money/order/lag`

Returns the lag time for executing orders in microseconds

1 **Request:** `BTCUSD/money/order/lag` <br/>
**Response:** `{"result":"success","data":{"lag":2319790,"lag_secs":2.31979,"lag_text":"2.31979 seconds"}}`

* If the MtGox trade queue is empty, the lag time will be 0

[^ Back to contents](#contents)

---

#### `money/trades/fetch`

Gets up to 24 hours of trades beginning from the date specified

key | value
--- | ---
since | The unix timestamp (in microseconds) from which to fetch the trades; this is optional, if omitted, the most recent 24 hours of trades will be returned

*Time now is 1364767250*

1 **Request:** `BTCUSD/money/trades/fetch` ~ `since=1364767190000000` <br/>
**Response:** 

```
{"result":"success","data":[
	{"date":1364767201,"price":"92.65","amount":"0.47909825","price_int":"9265000","amount_int":"47909825","tid":"1364767201381791","price_currency":"USD","item":"BTC","trade_type":"bid","primary":"Y","properties":"limit"},
	{"date":1364767207,"price":"92.65","amount":"0.01024284","price_int":"9265000","amount_int":"1024284","tid":"1364767207406066","price_currency":"USD","item":"BTC","trade_type":"bid","primary":"Y","properties":"limit"},
	{"date":1364767232,"price":"92.65","amount":"1.25637797","price_int":"9265000","amount_int":"125637797","tid":"1364767232522802","price_currency":"USD","item":"BTC","trade_type":"bid","primary":"Y","properties":"limit"},
	{"date":1364767236,"price":"92.65","amount":"0.01","price_int":"9265000","amount_int":"1000000","tid":"1364767236979043","price_currency":"USD","item":"BTC","trade_type":"bid","primary":"Y","properties":"limit"},
	{"date":1364767245,"price":"92.65","amount":"0.01","price_int":"9265000","amount_int":"1000000","tid":"1364767245879648","price_currency":"USD","item":"BTC","trade_type":"bid","primary":"Y","properties":"limit"}
]}
```

* In the above example, as the since date was 60s ago, only 60s of trades were returned
* If you wish to obtain *all* trade data, first fetch trades since 0, then continue fetching trades, using the date of the last fetched trade
* If there is no last trade (due to no trades happening in a 24 hour period), try adding 86400000000 microseconds to advance the window manually
* Make sure to remove duplicate trades that may lie on the fetch window boundary
* The `primary` field specifies whether the trade was originally ordered in the given currency - since trades may appear on multiple tickers, you may need to ignore trades with `primary: 'N'` values

[^ Back to contents](#contents)

#### `money/trades/cancelled`

Get a list of cancelled trades, supposedly from the last month

key | value
--- | ---
_since_ | It has been assumed that this method may share the same `since` argument as `money/trades/fetch` does, and this should work in a similar way

1 **Request:** `BTCUSD/money/trades/cancelled` <br/>
**Response:** `{"result":"success","data":[]}`

* As far as I can tell, no trade has ever been cancelled, at least for the `BTCUSD` currency pair, as repeatedly calling this API method with `since` values ranging from the first ever trade to March 31st, 2013, yielded no results
* This may also be because the `since` argument may not actually be valid
* For this reason, it is unclear what the response would look like when there are cancelled trades present
* It is likely that they will share the same format as in [`money/trades/fetch`](#moneytradesfetch)

[^ Back to contents](#contents)

---

#### `money/depth/fetch`

Gets recent depth information

1 **Request:** `BTCUSD/money/depth/fetch` <br/>
**Response:**

```
{"result":"success","data":{
	"asks":[...],
	"bids":[...],
	"filter_min_price":{"value":"83.48403","value_int":"8348403","display":"$83.48403","display_short":"$83.48","currency":"USD"},
	"filter_max_price":{"value":"102.03603","value_int":"10203603","display":"$102.03603","display_short":"$102.04","currency":"USD"}}}
}}
```

* The above data ranges from 1359589420235069 (Wed, 30 Jan 2013 23:43:40 GMT) to 1364770856550122 (Sun, 31 Mar 2013 23:00:57 GMT)
* Even though this data is about 6 times smaller than [`money/depth/full`](#moneydepthfull), it is still very big, so it is possible that excessive requests will also result in banning
* There are 852 bids and 839 asks in the above example
* Refer to [`money/depth/full`](#moneydepthfull) for detail on the ask and bid formats
* There is probably an argument that can be used to limit the amount of data returned, however it is currently unknown

[^ Back to contents](#contents)

#### `money/depth/full`

Gets a complete copy of MtGox's full depth data

1 **Request:** `BTCUSD/money/depth/full` <br/>
**Response:** 

```
{"result":"success","data":{
	"asks":[
		{"price":93.06545,"amount":0.07284541,"price_int":"9306545","amount_int":"7284541","stamp":"1364769641591046"},
		{"price":93.06546,"amount":0.81618662,"price_int":"9306546","amount_int":"81618662","stamp":"1364769116973406"},
		{"price":93.4,"amount":0.02,"price_int":"9340000","amount_int":"2000000","stamp":"1364769080120861"},
		...
	],
	"bids":[
		{"price":1.0e-5,"amount":30694957.01,"price_int":"1","amount_int":"3069495700999999","stamp":"1364764907440546"},
		{"price":2.0e-5,"amount":10000.02,"price_int":"2","amount_int":"1000002000000","stamp":"1364418775588024"},
		{"price":3.0e-5,"amount":41515.8015,"price_int":"3","amount_int":"4151580150000","stamp":"1364225564960621"},
		...
	]
}}
```

* This file can be at least as big as 1MB, and so there is a rate limit of up to 5 requests per hour, if you exceed this you risk being banned
* The example above contained 7671 bids and 1929 asks
* The date range was from 1309117574539900 (Sun, 26 Jun 2011 19:46:15 GMT) to 1364769659729678 (Sun, 31 Mar 2013 22:41:00 GMT)

[^ Back to contents](#contents)

---

### Security

**This category does not require the use of a currency pair**

#### `security/hotp/gen`

Generates a new HOTP key, what this is used for is unclear, though it may have some use for application developers

1 **Request:** `security/hotp/gen` <br/>
**Response:**

```
{
	"result":"success",
	"data": {
		"serial":"123",
		"name":"OTP#123",
		"key":"abc456",
		"key_base32":"ABC789"
	}
}
```

[^ Back to contents](#contents)

---

### Other methods

The following methods have yet to be documented. They mainly revolve around bank management, merchant transactions, and direct bitcoin transfers. Any known arguments are listed below each relevant method.

* **Manage registered bank accounts**
	* `money/bank/register`
	* `money/bank/list`:
		* _no arguments_
* **Japanese banks**
	* `money/japan/lookup_bank`
	* `money/japan/lookup_branch`
* **Deal directly with bitcoins**
	* `money/bitcoin/addpriv`:
		* `keytype`
		* `key`
		* `description`
		* `raw`
		* `sha256`
		* `auto`
	* `money/bitcoin/address`:
		* `description`
		* `ipn`
	* `money/bitcoin/wallet_add`:
		* `wallet`
	* `money/bitcoin/send_simple`:
		* `address`
		* `amonut_int`
		* `fee_int`
		* `no_instant`
		* `green`
	* `money/bitcoin/null`
	* `money/bitcoin/block_list_tx`
	* `money/bitcoin/tx_details`
	* `money/bitcoin/addr_details`
	* `money/bitcoin/vanity_lookup`
* **Bitinstant transactions**
	* `money/bitinstant/quote`
	* `money/bitinstant/fee`
* **Merchant transactions**
	* `money/merchant/order/create`
	* `money/merchant/order/pay`
	* `money/merchant/order/details`
	* `money/merchant/order/payment`
	* `money/merchant/pos/order/create`
	* `money/merchant/pos/order/close`
	* `money/merchant/pos/order/get`
	* `money/merchant/pos/order/add_product`
	* `money/merchant/pos/order/edit_product`
	* `money/merchant/product/add`
	* `money/merchant/product/del`
	* `money/merchant/product/get`
	* `money/merchant/product/edit`
* **Misc**
	* `money/code/list`
	* `money/code/redeem`
	* `money/swift/details`
	* `money/ticket/create`:
		* `currency`
		* `amount_int`
		* `partner`
	* `money/token/process`
	* `money/wallet/history`:
		* `currency`

[^ Back to contents](#contents)

---

Tips and donations: `1VvKzcFibZdMyQdbXpSYcyoYvBsLLodDa`

[^1]: https://en.bitcoin.it/wiki/MtGox/API/Streaming