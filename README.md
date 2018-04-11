# Multinode / Multiwallet Bitcoin regtest network

This repository allows you run a full bitcoin network in an isolated environment. It uses bitcoin's regtest capability to setup an isolated bitcoin network, and then uses docker to setup a network with 3 nodes.

This is useful because normally in regtest mode you would generate all coins in the same wallet as where you'd send the coins. With this setup, you can use one node to generate the coins and then send it to one of the other nodes, which can then again send it to another node to simulate more real-life bitcoin usage.

## Usage

Simple run

`docker-compose up`

to start all the containers. This will start the bitcoin nodes, and expose RPC on all of them. The nodes will run on the following ports:

| Node | P2P port * | RPC port * | RPC Username | RPC Password |
| --- | --- | --- | --- | ---|
| miner1 | 18500 | 18400 | bitcoin | bitcoin |
| node1 | 18501 | 18401 | bitcoin | bitcoin |
| node2 | 18502 | 18402 | bitcoin | bitcoin |

\* Port as exposed on the host running docker.

## Samples

Note these samples use `curl` to exercise the API, but this would usually be `bitcoin-cli`. We're using `curl` so we don't have a dependency on bitcoin in the host.

### Initial block count

Checks that the initial block count is 0.

```
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# docker-compose up -d
Creating bitcoindocker_miner_1 ... done
Creating bitcoindocker_node1_1 ... done
Creating bitcoindocker_node2_1 ... done
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getblockcount"}' -u bitcoin:bitcoin localhost:18400
{"result":0,"error":null,"id":"1"}
```

### Check connected nodes

Check the "miner" node is connected to other nodes.

```
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getpeerinfo","params":[]}' -u bitcoin:bitcoin -s localhost:18400 | jq '.result[] | {addr, inbound} '
{
  "addr": "node1:18444",
  "inbound": false
}
{
  "addr": "172.18.0.3:34908",
  "inbound": true
}
{
  "addr": "node2:18444",
  "inbound": false
}
```

### Mine some blocks and see other nodes are updating their block count

```
# mine the blocks
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"generate","params":[101]}' -u bitcoin:bitcoin -s localhost:18400
{"result":["229e6fc0599e198f6f9bdc04cb9a0d7f494f2c8b36f3cec354c85c2507ae21d2", ... 
<snip> 
...,"39b792af704cad1f26d2d2e34d91d784b88217b663a6d4eff5512fb6f588d4d5"],"error":null,"id":"1"}

# check on node1
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getblockcount","params":[]}' -u bitcoin:bitcoin -s localhost:18401
{"result":101,"error":null,"id":"1"}

# check on node2
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getblockcount","params":[]}' -u bitcoin:bitcoin -s localhost:18402
{"result":101,"error":null,"id":"1"}
```

### Send bitcoin from miner to another node

Now we're going to generate an address in another node. Note that we use port **18401** (node1) instead of 18400:
```
# Mine blocks first
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"generate","params":[101]}' -u bitcoin:bitcoin -s localhost:18400
{"result":["2929287f219fda71445219269081364373b4bdec73187da92126150feac39cd6",
... <snip> ..., 
"2b1ffc0528fac9e759097a9213198b1c8540545b67ea1dc4c0ca68b76f02f8f1"],"error":null,"id":"1"}

# Check balance
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getbalance","params":[]}' -u bitcoin:bitcoin -s localhost:18400
{"result":50.00000000,"error":null,"id":"1"}

# Generate address on node1
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getnewaddress","params":[]}' -u bitcoin:bitcoin -s localhost:18401
{"result":"2N444M93zqwyhhFSDJxgzPL5h9XMdwLUibz","error":null,"id":"1"}

# Send from miner to node1
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"sendtoaddress","params":["2N444M93zqwyhhFSDJxgzPL5h9XMdwLUibz", "3.14"]}' -u bitcoin:bitcoin -s localhost:18400
{"result":"008f138f10e80aaae2a5211bf2891ad522a1dd7b85d3f26cbbdabfa63c60ced0","error":null,"id":"1"}

# Now, since the block was not yet mined, we usually don't see the balance yet, unless  we specify 0 confirmations.
# First with the default (1) confirmation:
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getbalance","params":[]}' -u bitcoin:bitcoin -s localhost:18401
{"result":0.00000000,"error":null,"id":"1"}

# No coins, so let's try 0 confirmations (the first parameter "" means default account)
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getbalance","params":["", 0]}' -u bitcoin:bitcoin -s localhost:18401
{"result":3.14000000,"error":null,"id":"1"}

# This also means that node1 has it in the mempool, which shows there is exactly one transaction in it
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getmempoolinfo","params":[]}' -u bitcoin:bitcoin -s localhost:18401 | jq .
{
  "result": {
    "size": 1,
    "bytes": 187,
    "usage": 1024,
    "maxmempool": 300000000,
    "mempoolminfee": 1e-05,
    "minrelaytxfee": 1e-05
  },
  "error": null,
  "id": "1"
}

# Finally, let's mine the block and see that getbalance will show the balance by default.
# node1:
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getbalance","params":[]}' -u bitcoin:bitcoin -s localhost:18401
{"result":3.14000000,"error":null,"id":"1"}

# miner
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getbalance","params":[]}' -u bitcoin:bitcoin -s localhost:18400
{"result":96.85996240,"error":null,"id":"1"}

# Some extras

# List wallet affecting transactions:
# miner
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"listtransactions","params":["", 150]}' -u bitcoin:bitcoin -s localhost:18400 | jq '.result [] | {amount, confirmations, txid, category}'
{
  "amount": 50,
  "confirmations": 102,
  "txid": "7bd5b1c805dc1ec6eab533242f56c0685c713c709e7f3e7cec858773163c7d48",
  "category": "generate"
}
{
  "amount": 50,
  "confirmations": 101,
  "txid": "d32be1fce3dd33cb46701cfcaf9de1ddfde9e18dcbe35984799b6341e38a9c53",
  "category": "generate"
}
<snip>
{
  "amount": 50,
  "confirmations": 3,
  "txid": "2e3922018911136f8474f741956e45f24fd1d100908bee059aacaaf5d815d4b6",
  "category": "immature"
}
{
  "amount": 50,
  "confirmations": 2,
  "txid": "e328e32106d04121ab32300c5e35b3880a21ab3ebecd6c7af34417278bf2da49",
  "category": "immature"
}
{
  "amount": -3.14,
  "confirmations": 1,
  "txid": "008f138f10e80aaae2a5211bf2891ad522a1dd7b85d3f26cbbdabfa63c60ced0",
  "category": "send"
}
{
  "amount": 50.0000376,
  "confirmations": 1,
  "txid": "dafe13b91d80fd899e627fb481c22bdd10f9d2da13c55ec0b7af7f30175ce96c",
  "category": "immature"
}

# node1
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"listtransactions","params":["", 150]}' -u bitcoin:bitcoin -s localhost:18401 | jq '.result [] | {amount, confirmations, txid, category}'
{
  "amount": 3.14,
  "confirmations": 1,
  "txid": "008f138f10e80aaae2a5211bf2891ad522a1dd7b85d3f26cbbdabfa63c60ced0",
  "category": "receive"
}

# node2
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"listtransactions","params":["", 150]}' -u bitcoin:bitcoin -s localhost:18402
{"result":[],"error":null,"id":"1"}

# Try getting the transaction
# node1:
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"gettransaction","params":["008f138f10e80aaae2a5211bf2891ad522a1dd7b85d3f26cbbdabfa63c60ced0"]}' -u bitcoin:bitcoin -s localhost:18401 | jq .
{
  "result": {
    "amount": 3.14,
    "confirmations": 1,
    "blockhash": "00f3b3923a14a69a78a419660e20d63a4f91370d6099c3bbbdc5daad953e2985",
    "blockindex": 1,
    "blocktime": 1523421980,
    "txid": "008f138f10e80aaae2a5211bf2891ad522a1dd7b85d3f26cbbdabfa63c60ced0",
    "walletconflicts": [],
    "time": 1523421666,
    "timereceived": 1523421666,
    "bip125-replaceable": "no",
    "details": [
      {
        "account": "",
        "address": "2N444M93zqwyhhFSDJxgzPL5h9XMdwLUibz",
        "category": "receive",
        "amount": 3.14,
        "label": "",
        "vout": 0
      }
    ],
    "hex": "0200000001487d3c16738785ec7c3e7f9e703c715c68c0562f2433b5eac61edc05c8b1d57b0000000048473044022065b2d5c649bb6882f73654398ebaa9bf29281088493ad56f1ee0400bfcbeb8b30220402fea8148de73c14ba332693f7d5c9e2f2bf4332299a2df87115fdabaf1607b01feffffff028042b7120000000017a914768cc2b85515737b398e2613fd7fe9c3d44f73f187d0a04e170100000017a9146a82eef49043551afd83cb286d801c0725ba24318765000000"
  },
  "error": null,
  "id": "1"
}

# node2, here it fails because that transaction isn't in the index. Setting txindex=1 in bitcoin.conf _would_ give a result here.
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"gettransaction","params":["008f138f10e80aaae2a5211bf2891ad522a1dd7b85d3f26cbbdabfa63c60ced0"]}' -u bitcoin:bitcoin -s localhost:18402 | jq .
{
  "result": null,
  "error": {
    "code": -5,
    "message": "Invalid or non-wallet transaction id"
  },
  "id": "1"
}

```


## Requirements

To run this you need both `docker` and `docker-compose`. It was tested on a clean Ubuntu 16.04 with the docker-ce package from Docker. 
