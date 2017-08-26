Tinychain

Start from scratch
Get a private key for our wallet.

## Initiate wallet
bitcoin wallet
public key - a function of the private key
wallet address - a hashed version of the public key

```python
@lru_cache()
def init_wallet(path=None):
    path = path or WALLET_PATH

    # signing_key: the private key.
    if os.path.exists(path):
        with open(path, 'rb') as f:
            signing_key = ecdsa.SigningKey.from_string(
                f.read(), curve=ecdsa.SECP256k1)
    else:
        logger.info(f"generating new wallet: '{path}'")
        signing_key = ecdsa.SigningKey.generate(curve=ecdsa.SECP256k1)
        with open(path, 'wb') as f:
            f.write(signing_key.to_string())

    # verifying_key is the public key
    verifying_key = signing_key.get_verifying_key()
    # address is essentially a hash of the public key.
    my_address = pubkey_to_address(verifying_key.to_string())
    logger.info(f"your address is {my_address}")

    return signing_key, verifying_key, my_address
```

## Two Networks
To successfully run tinycoin, we need two networks. One for accepting commands from Tinychain users (command network), and the other dedicated to mining (mining network).

### Command Network
Command network connects tinychain user with the chain itself.

```python
# The command network
PORT = os.environ.get('TC_PORT', 9999)
server = ThreadedTCPServer(('0.0.0.0', PORT), TCPHandler)

def start_worker(fnc):
    workers.append(threading.Thread(target=fnc, daemon=True))
    workers[-1].start()

start_worker(server.serve_forever)
```

### Mining Network

### Running Servers

## Life of Transaction
Suppose Jack already has 10 Bitcoins. Now, he wants to transfer 1 Bitcoin to Bob. How does the code work?

## Mining and Blocks
What is Proof of Work

## Balance Inquiry
client.get_balance
coinbase transaction: the transaction to send miner fees


## Other Takeaways
namedtuple more like c/c++ struct with strong types. In Py3.6++

```python
class MerkleNode(NamedTuple):
    val: str
    children: Iterable = None
```

How to avoid double spending?
https://www.pandastrike.com/posts/20150429-understanding-bitcoin

## Script
pay-to-pubkey

<!-- https://stackoverflow.com/questions/16981921/relative-imports-in-python-3 -->
