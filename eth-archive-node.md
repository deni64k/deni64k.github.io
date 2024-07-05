---
layout: page
title: Ethereum Archive Node
permalink: /eth-archive-node/
---

> [!WARNING] :warning: Due to the lack of interest and my BTRFS array being crushed, the node is no longer archive. Please, contact me if you're interested in this project.

Hello stranger,

I maintain an archive node for Ethereum and want to share access to it with you and anyone who is interested.

Feel free to contact me for a snapshot: ~16 TB.

The node is available by this URL: https://geth.deni64k.online/.

Here is an example of an API call:

``` sh
$ curl -sX POST                                                                            \
       -H "Content-Type: application/json"                                                 \
       --data '{"method": "web3_clientVersion", "params": [], "id": 1, "jsonrpc": "2.0"}'  \
       https://geth.deni64k.online/ | jq -CS
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": "Geth/v1.13.2-stable-dc34fe82/Linux-amd64/go1.21.1"
}
```

The benefit of an archive node is you can receive the balance of a wallet at any block. For instance, the balance of `0x388C818CA8B9251b393131C08a736A67ccB19297` (Lido Rewards Vault) at block 18748800 was 8765235897697649868 wei:

``` sh
$ cat <<EOF                                                                                   \
  | curl -sX POST -H "Content-Type: application/json" --data @- https://geth.deni64k.online/  \
  | jq -r .result | xargs printf 'Balance is %d wei\n'
{
 "id": 1,
 "jsonrpc": "2.0",
 "method": "eth_getBalance",
 "params": ["0x388C818CA8B9251b393131C08a736A67ccB19297",
            "$(printf "0x%x" 18748800)"]
}
EOF
Balance is 8765235897697649868 wei
```

You can contact me on Discord: `deni64k`.

If you wish to donate:
* ADA `addr1qxteqqjlhhr00xe4ghnyfuz7v0jgnnkcfdhfvgt65l8xejhagejn870vtla5906tupyzdwfj4r6eju2pv3ady8vuynfslralt3`
* ETH `0x39996591D206144fa6801CdC5FD6D94312F3D9e0`

Please, don't DoS it.
