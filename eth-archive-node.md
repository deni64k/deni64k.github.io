---
layout: page
title: Ethereum Archive Node
permalink: /eth-archive-node/
---

Hello stranger,

I got my archive node fully synced and want to share access to it with you and anyone interested.

You can access it via an ngrok tunnel. If I see real interest in it, I will come up with a better solution. I can also share the database by request: ~8.1 TB.

The node is available by this URL: https://geth.deni64k.online/.

Here is an example of an API call:

``` sh
$ curl -sX POST                                                                     \
       --data '{"method":"web3_clientVersion","params":[],"id":1,"jsonrpc":"2.0"}'  \
       -H "Content-Type: application/json"                                          \
       https://geth.deni64k.online/ | jq -CS
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": "Geth/v1.10.26-stable-e5eb32ac/linux-amd64/go1.18.3"
}
```

You can contact me on Discord: `deni64k#5549`.

If you wish to donate:
* ADA `addr1qxteqqjlhhr00xe4ghnyfuz7v0jgnnkcfdhfvgt65l8xejhagejn870vtla5906tupyzdwfj4r6eju2pv3ady8vuynfslralt3`
* ETH `0x39996591D206144fa6801CdC5FD6D94312F3D9e0`

Please, don't DDOS it.
