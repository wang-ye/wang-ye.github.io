Tinychain


namedtuple more like c/c++ struct. In Py3.6++

```python
class MerkleNode(NamedTuple):
    val: str
    children: Iterable = None
```

## Initiate wallet
bitcoin wallet - 

address - a hashed version of the public key

## Balance

client.get_balance
coinbase transaction: the transaction to send miner fees