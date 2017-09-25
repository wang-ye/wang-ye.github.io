---
layout: post
title:  "Tinychain - Part One"
date:   2017-09-23 21:31:01 -0800
---
[Bitcoin](https://en.wikipedia.org/wiki/Bitcoin), and its underlying technology blockchain, have become increasingly popular. For some quick Bitcoin introduction, refer to [this](http://chimera.labs.oreilly.com/books/1234000001802/ch07.html#_introduction_2). In this post, we will discuss [Tinychain](https://github.com/jamesob/tinychain), a Python implementation of Bitcoin. The code is nicely written with Python 3.6 and demonstrates key features.

We will divide the Tinychain analysis into two parts. In part one, we will cover several important concepts such as wallet, proof-of-work (POW), Merkle tree, and how miner works.

## What is Wallet?
Bitcoin uses cryptography to create wallets, and Tinychain loyally implements the same logic. Basically, it first calls ecdsa package to get the signing key (private key), and from signing key we can then generate the verifying key (public key). The wallet address we saw is actually a hash of the verifying key. 

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

Note the use of lru_cache in ``init_wallet`` method - it basically maintains a cache for each path and the keys/addresses. As an extended reading, [Mastering Bitcoin](http://chimera.labs.oreilly.com/books/1234000001802/ch04.html) has a dedicated chapter to discuss the wallets and the public/private keys.

## Understanding Proof Of Work (POW)
Another important concept in Bitcoin is proof of work (POW). What is POW? Basically, miners need to assemble (mine) blocks and broadcast them to the whole network. To achieve this, miners must *conduct non-trivial effort* to solve a puzzle with certain difficulty level. Usually, this will take tons of CPU cycles! In this section, we will understand the puzzle and difficulty level by going through Tinychain code.

### The Puzzle
Each block has a header method.

```python
def header(self, nonce=None) -> str:
    """
    This is hashed in an attempt to discover a nonce under the difficulty
    target.
    """
    return (
        f'{self.version}{self.prev_block_hash}{self.merkle_hash}'
        f'{self.timestamp}{self.bits}{nonce or self.nonce}')
```

Once the block is created, all the fields (version, prev_block_hash, bits ..) except nonce is finalized. The puzzle for miner to solve is **finding a nonce, such that the sha256d(block.header(nonce)), 16) is less than a given target.**. The target is determined by a difficulty level, which will be covered later in this section.

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

### Getting Difficulty Level
When a block is generated, its ``bits`` var, aka the difficulty level, is set by ``get_next_work_required`` method call. The difficulty is adjusted every ``DIFFICULTY_PERIOD_IN_BLOCKS`` blocks. Then, we calculate the block generation speed and adjust the difficulty accordingly.

```python
TIME_BETWEEN_BLOCKS_IN_SECS_TARGET = 1 * 60
DIFFICULTY_PERIOD_IN_SECS_TARGET = (60 * 60 * 10)
DIFFICULTY_PERIOD_IN_BLOCKS = (
  DIFFICULTY_PERIOD_IN_SECS_TARGET / TIME_BETWEEN_BLOCKS_IN_SECS_TARGET)

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

## What and Why Merkle Tree?
[Merkle tree](http://chimera.labs.oreilly.com/books/1234000001802/ch07.html#merkle_trees), also known as binary hash tree, is used to securely verify whether a element is in a set or list. You may also notice the merkle_hash field in each block. So how do we implement Merkle tree here and how do we use the merkle_hash? The implementation details are as follows:

```python
def get_merkle_root_of_txns(txns):
    return get_merkle_root(*[t.id for t in txns])


class MerkleNode(NamedTuple):
    val: str
    children: Iterable = None


def _chunks(l, n) -> Iterable[Iterable]:
    return (l[i:i + n] for i in range(0, len(l), n))


@lru_cache(maxsize=1024)
def get_merkle_root(*leaves: Tuple[str]) -> MerkleNode:
    """Builds a Merkle tree and returns the root given some leaf values."""
    if len(leaves) % 2 == 1:
        leaves = leaves + (leaves[-1],)

    def find_root(nodes):
        newlevel = [
            MerkleNode(sha256d(i1.val + i2.val), children=[i1, i2])
            for [i1, i2] in _chunks(nodes, 2)
        return find_root(newlevel) if len(newlevel) > 1 else newlevel[0]

    return find_root([MerkleNode(sha256d(l)) for l in leaves])
```

``get_merkle_root`` calculates the root given a list of transaction ids. When verifying transaction inclusiveness in block, it only need log(N) Merkle nodes as illustrated in Figure 7-5 of [Mastering Bitcoin](http://chimera.labs.oreilly.com/books/1234000001802/ch07.html#merkle_trees).

## Understanding Miner
To successfully operate Tinychain, miner needs to run a service that does two things - one is for accepting commands from Tinychain users (command network), and the other dedicated to mining (mining network). In the following I will illustrate each of them.

### Command Network
Command network connects tinychain user with the chain itself. A user can send commands to the network via Tinychain client. The actions can be getting balance, querying transaction status and transfer value between accounts. We will dive into this when discussing the Tinychain client.

```python
class TCPHandler(socketserver.BaseRequestHandler):

    def handle(self):
        data = read_all_from_socket(self.request)
        peer_hostname = self.request.getpeername()[0]
        peer_hostnames.add(peer_hostname)
        logger.debug(f'peer_hostnames: {peer_hostnames}')

        if hasattr(data, 'handle') and isinstance(data.handle, Callable):
            logger.info(f'received msg {data} from peer {peer_hostname}')
            data.handle(self.request, peer_hostname)
        elif isinstance(data, Transaction):
            logger.info(f"received txn {data.id} from peer {peer_hostname}")
            add_txn_to_mempool(data)
        elif isinstance(data, Block):
            logger.info(f"received block {data.id} from peer {peer_hostname}")
            connect_block(data)

# The command network
PORT = os.environ.get('TC_PORT', 9999)
server = ThreadedTCPServer(('0.0.0.0', PORT), TCPHandler)

def start_worker(fnc):
    workers.append(threading.Thread(target=fnc, daemon=True))
    workers[-1].start()

start_worker(server.serve_forever)
```

Tinychain uses a TCP server to accept requests from users, and conduct corrsponding actions based on request type.

### Mining Server
A miner is created by running the ``mine_forever``. As the name indicates, it runs endlessly to capture new transactions, assemble them into blocks, and broadcast the block to other nodes in the mining network.

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

## Other Learnings
In Py3.6++, we can now support strong types. namedtuple now looks very similar to c/c++ struct now:

```python
class MerkleNode(NamedTuple):
    val: str
    children: Iterable = None
```

It uses the following logging template:
```python
logging.basicConfig(
    level=getattr(logging, os.environ.get('TC_LOG_LEVEL', 'INFO')),
    format='[%(asctime)s][%(module)s:%(lineno)d] %(levelname)s %(message)s')

logger = logging.getLogger(__name__)
```

## Takeaway and Next Steps
In this post, we discussed important Tinychain concepts, including wallet definition, proof of work (POW), Merkle tree in tinychain and how the mining network works. In Part two, we will focus on the client, transaction, block assembling, block validation and how to handle orphan chains.
