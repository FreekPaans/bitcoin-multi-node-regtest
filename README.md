# Multinode Bitcoin regtest network

This repository allows you run a full bitcoin network in an isolated environment. It uses bitcoin's regtest capability to setup an isolated bitcoin network, and then uses docker to setup a network with 3 nodes.

This is useful because normally in regtest mode you would generate all coins in the same wallet as where you'd send the coins. With this setup, you can use one node to generate the coins and then send it to one of the other nodes, which can then again send it to another node to simulate more real-life bitcoin usage.

## Usage

Simple run

`docker-compose up`

to start all the containers. This will start the bitcoin nodes, and expose RPC on all of them. See the [table below for port information](#Connection-info).

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
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getpeerinfo"}' -u bitcoin:bitcoin -s localhost:18400  | jq ' .result[].addr'
"node1:18444"
"node2:18444"
```

### Mine some blocks and send rewards to an address
Mine the blocks
```
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"generate","params":[101]}' -u bitcoin:bitcoin -s localhost:18400
{"result":["229e6fc0599e198f6f9bdc04cb9a0d7f494f2c8b36f3cec354c85c2507ae21d2", ... 
<snip> 
...,"39b792af704cad1f26d2d2e34d91d784b88217b663a6d4eff5512fb6f588d4d5"],"error":null,"id":"1"}
```
Now we're going to generate an address in another node. Note that we use port **18401** (node1) instead of 18400:
```
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getnewaddress","params":[]}' -u bitcoin:bitcoin -s localhost:18401
{"result":"2MxWvETqhPZtksiAgaw1qqCSu3eQSRHjLaW","error":null,"id":"1"}
```
Then send some coins from the node on **18400**, mine the block and see that the balance is updated.
```
# send the coins on 400
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"sendtoaddress","params":["2MxWvETqhPZtksiAgaw1qqCSu3eQSRHjLaW", "3.14"]}' -u bitcoin:bitcoin -s localhost:18400
{"result":"4d23b35ba4af961de9873216a7a3465573223e4c6f69319235d60ddc997948fa","error":null,"id":"1"}

# mine the block
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"generate","params":[1]}' -u bitcoin:bitcoin -s localhost:18400
{"result":["6583d7e58471b13c0db743f8612793263407e11becb98c38c01f100be8232a9d"],"error":null,"id":"1"}

# see the balance
root@ubuntu-xenial:/home/vagrant/bitcoin-docker# curl -d '{"jsonrpc":"2.0","id":"1","method":"getbalance","params":[]}' -u bitcoin:bitcoin -s localhost:18401
{"result":3.14000000,"error":null,"id":"1"}
```

## Connection info


| Node | P2P port * | RPC port * | RPC Username | RPC Password |
| --- | --- | --- | --- | ---|
| miner1 | 18500 | 18400 | bitcoin | bitcoin |
| node1 | 18501 | 18401 | bitcoin | bitcoin |
| node2 | 18502 | 18402 | bitcoin | bitcoin |

\* Port as exposed on the host running docker.

## Requirements

To run this you need both `docker` and `docker-compose`.
