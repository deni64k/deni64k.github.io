---
layout: page
title: Ethereum Archive Node
permalink: /eth-archive-node/
---

Hello stranger,

I got my archive node fully synced and want to share access to it with you and anyone interested.

You can access it via an ngrok tunnel. If I see real interest in it, I will come up with a better solution. In terms of SLA, I think 95% would be reasonable to expect: my Internet connection can be flaky. I can also share the database by request: ~8.1 TB.

The node is available by this URL: https://ec453b92cd8c.ngrok.io/

Here is an example of an API call:

``` sh
$ curl -sX POST                                                                     \
       --data '{"method":"web3_clientVersion","params":[],"id":1,"jsonrpc":"2.0"}'  \
       -H "Content-Type: application/json"                                          \
       -L https://ec453b92cd8c.ngrok.io/ | jq -CS
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": "Geth/v1.10.5-stable-33ca98ec/linux-amd64/go1.16.3"
}
```

You can contact me on Discord: `deni64k#5549`.

If you wish to donate: `0x39996591D206144fa6801CdC5FD6D94312F3D9e0`.

Please, don't DDOS it.
