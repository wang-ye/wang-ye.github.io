---
layout: post
title: "Tinychain Part Two"
date:   2018-01-10 19:40:01 -0800
---

We previously posted about Tinychain. There are still some unresolved questions. What is a client? What is a transaction, and how can we assemble the blocks?

## Tinychain Client
A tinychain client is a utility that interacts with the tinychain on behalf of a given user. It supports three operations:

```python
# client.py
def main():
   ...
   if args['balance']:
        get_balance(args)
    elif args['send']:
        send_value(args)
    elif args['status']:
        txn_status(args)
```

How to check transaction status? What does it mean?

For get_balance method, socket, calls  

```python
def find_utxos_for_address(args: dict):
    # utxo_set contains ALL unspent transactions.
    utxo_set = dict(send_msg(t.GetUTXOsMsg()))
    # Filter by the user's address to get the balance for the user only.
    return [u for u in utxo_set.values() if u.to_address == args['my_addr']]
```

For the sake of space, I will only discuss the most important send_value method. Imagine you have 10 tinycoins, and you want to send 1 tinycoin to Jack, then send_value method would be triggered:

Here is another important concept: like your wealth can be represented by multiple valid checks, in tinycoin your total number of coins are represented by *Unspent Transaction Output*. To send coin to other people, the client would first query available UTXOS in the chains, and then create a transaction to 

```python
def send_value(args: dict):
    """Send value to some address."""
    val, to_addr, sk = int(args['<val>']), args['<addr>'], args['signing_key']
    selected = set()
    my_coins = list(sorted(
        find_utxos_for_address(args), key=lambda i: (i.value, i.height)))

    for coin in my_coins:
        selected.add(coin)
        if sum(i.value for i in selected) > val:
            break

    txout = t.TxOut(value=val, to_address=to_addr)

    txn = t.Transaction(
        txins=[make_txin(sk, coin.outpoint, txout) for coin in selected],
        txouts=[txout])

    logger.info(f'built txn {txn}')
    logger.info(f'broadcasting txn {txn.id}')
    send_msg(txn)

# Communicate with miners using Socket.
def send_msg(data: bytes, node_hostname=None, port=None):
    node_hostname = getattr(send_msg, 'node_hostname', 'localhost')
    port = getattr(send_msg, 'port', 9999)

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((node_hostname, port))
        s.sendall(t.encode_socket_data(data))
        return t.read_all_from_socket(s)
```


The "coinbase transaction" is the transaction inside a block that pays the miner his block reward. Each block starts with a coinbase transaction.


How do we form block? - miner assembles transactions to form a block.

How do we validate a transaction is valid?
A -> B some money, how do we know the transaction is initiated by A?
Transaction validation

### Running Servers With Docker
Here is the ``docker`` file:

```dockerfile
FROM python:3.6.2
ADD requirements.txt ./
RUN pip install -r requirements.txt
ADD tinychain.py ./

CMD ["./tinychain.py", "serve"]
```

and the ``docker-compose.yaml``:
```
version: "3"
services:
  node1:
    build: .
    image: tinychain
    ports:
      - "9999:9999"
  node2:
    image: tinychain
    environment:
      TC_PEERS: 'node1'
```



## Life of Transaction
So, what is a transaction? 

```python
    txn = t.Transaction(
        txins=[make_txin(sk, coin.outpoint, txout) for coin in selected],
        txouts=[txout])

class Transaction(NamedTuple):
    txins: Iterable[TxIn]
    txouts: Iterable[TxOut]

    # The block number or timestamp at which this transaction is unlocked.
    # < 500000000: Block number at which this transaction is unlocked.
    # >= 500000000: UNIX timestamp at which this transaction is unlocked.
    locktime: int = None

    @property
    def is_coinbase(self) -> bool:
        return len(self.txins) == 1 and self.txins[0].to_spend is None

    @classmethod
    def create_coinbase(cls, pay_to_addr, value, height):
        return cls(
            txins=[TxIn(
                to_spend=None,
                # Push current block height into unlock_sig so that this
                # transaction's ID is unique relative to other coinbase txns.
                unlock_sig=str(height).encode(),
                unlock_pk=None,
                sequence=0)],
            txouts=[TxOut(
                value=value,
                to_address=pay_to_addr)],
        )

    @property
    def id(self) -> str:
        return sha256d(serialize(self))

    def validate_basics(self, as_coinbase=False):
        if (not self.txouts) or (not self.txins and not as_coinbase):
            raise TxnValidationError('Missing txouts or txins')

        if len(serialize(self)) > Params.MAX_BLOCK_SERIALIZED_SIZE:
            raise TxnValidationError('Too large')

        if sum(t.value for t in self.txouts) > Params.MAX_MONEY:
            raise TxnValidationError('Spend value too high')
```

Suppose Jack already has 10 Bitcoins. Now, he wants to transfer 1 Bitcoin to Bob. How does the code work?

## Balance Inquiry
client.get_balance
coinbase transaction: the transaction to send miner fees


## Working with Orphan Branches?

## Merkle Node?

select_from_mempool

How to avoid double spending?
https://www.pandastrike.com/posts/20150429-understanding-bitcoin

## Script
pay-to-pubkey


## Docker Basics
It uses a docker command.
Image vs container difference. A running instance of an image is a container.
docker inspect to return low level info


## IBD
Initial block download