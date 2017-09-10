Tinychain Part1

Bitcoin, and its underlying technology blockchain has become increasingly popular topic.

Important Concepts

In part one, we discuss wallet, client and miner.

## Tinychain Client
A tinychain client supports three operations.

```python
   if args['balance']:
        get_balance(args)
    elif args['send']:
        send_value(args)
    elif args['status']:
        txn_status(args)
```

For the sake of space, I will only discuss the most important send_value method. Imagine you have 10 tinycoins, and you want to send 1 tinycoin to Jack, then send_value method would be triggered:

Here is an important concept: like your wealth can be represented by multiple valid checks, in tinycoin your total number of coins are represented by *Unspent Transaction Output*. To send coin to other people, the client would first query available UTXOS in the chains, and then create a transaction to 

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

## Understanding Miner
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

### Mining Server
A miner is created by running the ``mine_forever``. As the name indicates, it runs endlessly to capture new transactions, assemble them into blocks, and broadcast the block to the mining network.

```python

def assemble_and_solve_block(pay_coinbase_to_addr, txns=None):
    """Construct a Block by pulling transactions from the mempool, then mine it."""
    with chain_lock:
        prev_block_hash = active_chain[-1].id if active_chain else None

    block = Block(
        version=0,
        prev_block_hash=prev_block_hash,
        merkle_hash='',
        timestamp=int(time.time()),
        bits=get_next_work_required(prev_block_hash),
        nonce=0,
        txns=txns or [],
    )

    if not block.txns:
        block = select_from_mempool(block)

    fees = calculate_fees(block)
    my_address = init_wallet()[2]
    coinbase_txn = Transaction.create_coinbase(
        my_address, (get_block_subsidy() + fees), len(active_chain))
    block = block._replace(txns=[coinbase_txn, *block.txns])
    block = block._replace(merkle_hash=get_merkle_root_of_txns(block.txns).val)

    if len(serialize(block)) > Params.MAX_BLOCK_SERIALIZED_SIZE:
        raise ValueError('txns specified create a block too large')

    return mine(block)

def mine_forever():
    while True:
        my_address = init_wallet()[2]
        # As a simplification, only consider
        block = assemble_and_solve_block(my_address)

        if block:
            connect_block(block)
```

## What is a wallet?
Get a private key for our wallet.
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

lru_cache is used to store the


## Understanding Proof Of Work (POW)
How to determine mining difficulty level? It is controlled by difficulty bits.

```python
def get_next_work_required(prev_block_hash: str) -> int:
    """
    Based on the chain, return the number of difficulty bits the next block
    must solve.
    """
    if not prev_block_hash:
        return Params.INITIAL_DIFFICULTY_BITS

    (prev_block, prev_height, _) = locate_block(prev_block_hash)

    if (prev_height + 1) % Params.DIFFICULTY_PERIOD_IN_BLOCKS != 0:
        return prev_block.bits

    with chain_lock:
        # #realname CalculateNextWorkRequired
        period_start_block = active_chain[max(
            prev_height - (Params.DIFFICULTY_PERIOD_IN_BLOCKS - 1), 0)]

    # Adjust difficulty every DIFFICULTY_PERIOD_IN_BLOCKS.
    actual_time_taken = prev_block.timestamp - period_start_block.timestamp

    if actual_time_taken < Params.DIFFICULTY_PERIOD_IN_SECS_TARGET:
        # Increase the difficulty
        return prev_block.bits + 1
    elif actual_time_taken > Params.DIFFICULTY_PERIOD_IN_SECS_TARGET:
        return prev_block.bits - 1
    else:
        # Wow, that's unlikely.
        return prev_block.bits
```

Each block has a difficulty bits.
```python
# Identify a nonce for the given block, such that 
def mine(block):
    start = time.time()
    nonce = 0
    target = (1 << (256 - block.bits))
    mine_interrupt.clear()

    # Solve the puzzle - find a nonce such that sha256d(block.header(nonce) is smaller than target.
    while int(sha256d(block.header(nonce)), 16) >= target:
        nonce += 1

        if nonce % 10000 == 0 and mine_interrupt.is_set():
            logger.info('[mining] interrupted')
            mine_interrupt.clear()
            return None

    # Found the nonce!
    block = block._replace(nonce=nonce)
    duration = int(time.time() - start) or 0.001
    khs = (block.nonce // duration) // 1000
    logger.info(
        f'[mining] block found! {duration} s - {khs} KH/s - {block.id}')

    return block
```


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

## References
http://chimera.labs.oreilly.com/books/1234000001802/ch07.html#_introduction_2
