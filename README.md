# matrix-geth

`matrix-geth` is the **Execution Client** in the Matrix ecosystem (an execution-only rollup on BNB Chain). It consumes the ordered L2 input stream/batches derived by `matrix-node`, executes the EVM, maintains the L2 state, and exposes standard L2 JSON-RPC endpoints (`eth_*` / `net_*` / `web3_*` / `debug_*`).

---

## Role in the Matrix ecosystem

* **Execution & State**: executes derived inputs and maintains canonical L2 blocks/state
* **Public RPC**: provides standard Ethereum JSON-RPC for wallets, DApps, explorers, and indexers
* **Derivation integration**: works with `matrix-node` via Engine API (Auth RPC / JWT) to advance the chain

In one sentence: **`matrix-node` derives, `matrix-geth` executes.**

---

## How it works

```text
BNB Chain (L1)
  -> inclusion + ordering + data availability
matrix-node (Derivation)
  -> derive ordered L2 inputs/batches
matrix-geth (Execution)
  -> execute batches -> maintain state -> expose L2 RPC
```

---

## Genesis files

During image build, `matrix-chain/*.json.gz` is copied into the container root `/` and decompressed.

Available genesis files inside the container:

* `/matrix-mainnet.json`
* `/matrix-testnet.json`

Select the genesis via `GENESIS_FILE`:

* `GENESIS_FILE=matrix-mainnet.json`
* `GENESIS_FILE=matrix-testnet.json`

---

## Quickstart (Docker)

> The container entrypoint is `/init-geth.sh`. On first run it will automatically `geth init` (only if `/root/ethereum` is empty).
> **Strongly recommended:** mount a persistent volume to `/root/ethereum` to avoid data loss / re-init on restart.

### Build image

```bash
docker build -t matrix-geth .
```

### Run (Testnet example)

```bash
docker run -d --name matrix-geth \
  -p 8545:8545 \
  -p 8551:8551 \
  -e GENESIS_FILE=matrix-testnet.json \
  -e JWT_SECRET="$(openssl rand -hex 32)" \
  -e AUTH_RPC_PORT=8551 \
  -e RPC_GAS_CAP=500000000 \
  -e CACHE_SIZE=10000 \
  -e BLUEBIRD_TIMESTAMP=1763853609 \
  -v matrix-geth-data:/root/ethereum \
  matrix-geth
```

### Run (Mainnet example)

```bash
docker run -d --name matrix-geth \
  -p 8545:8545 \
  -p 8551:8551 \
  -e GENESIS_FILE=matrix-mainnet.json \
  -e JWT_SECRET="$(openssl rand -hex 32)" \
  -e AUTH_RPC_PORT=8551 \
  -e RPC_GAS_CAP=500000000 \
  -e CACHE_SIZE=10000 \
  -e BLUEBIRD_TIMESTAMP=1763853609 \
  -v matrix-geth-data:/root/ethereum \
  matrix-geth
```

> Security note: **Do NOT expose 8551 (Auth RPC / Engine API) to the public internet.** Keep it private/internal, or don’t publish the port at all.

---

## Verify

Follow logs:

```bash
docker logs -f matrix-geth
```

Query current L2 block number:

```bash
curl -sS -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}' \
  http://127.0.0.1:8545 ; echo
```

---

## Key configuration (matches init-geth.sh)

Required:

* `GENESIS_FILE`: must be `matrix-mainnet.json` or `matrix-testnet.json` (must exist under `/` in the container)
* `JWT_SECRET`: Engine API JWT secret

  * if `/tmp/jwtsecret` exists in the container, it will be used
  * otherwise `JWT_SECRET` is required (written into `/tmp/jwtsecret`)

Optional:

* `GENESIS_TIMESTAMP`: override `timestamp` in genesis at startup (accepts hex `0x...` or base-10; script converts automatically)
* `GENESIS_MIX_HASH`: override `mixHash` in genesis at startup
* `AUTH_RPC_PORT`: Auth RPC port, default `8551`
* `RPC_GAS_CAP`: RPC gas cap, default `500000000`
* `CACHE_SIZE`: geth cache size, default `10000`
* `BLUEBIRD_TIMESTAMP`: sets Bluebird fork timestamp (appends `--override.bluebird=<ts>`)

---

## Default ports (inside container)

* `8545`: HTTP JSON-RPC (`0.0.0.0`)
* `8551`: Auth RPC / Engine API (`0.0.0.0`, JWT protected)
* Dockerfile also exposes `8546 / 30303`, but the default runtime flags include `--nodiscover --maxpeers 0` (no P2P discovery/peering by default).

---

## License

See `COPYING` / `COPYING.LESSER`.
