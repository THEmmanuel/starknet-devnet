## Introduction

A Flask wrapper of Starknet state. Similar in purpose to Ganache.

Aims to mimic Starknet's Alpha testnet, but with simplified functionality.

## Contents

- [Install](#install)
- [Disclaimer](#disclaimer)
- [Run](#run)
- [Interaction](#interaction)
- [Dumping and Loading](#dumping)
- [Hardhat Integration](#hardhat-integration)
- [L1-L2 Postman Communication](#postman-integration)
- [Block Explorer](#block-explorer)
- [Lite Mode](#lite-mode)
- [Restart](#restart)
- [Contract debugging](#contract-debugging)
- [Development](#development)

## Install

```text
pip install starknet-devnet
```

### Requirements

Works with Python versions >=3.7.2 and <=3.9.10.

On Ubuntu/Debian, first run:

```text
sudo apt install -y libgmp3-dev
```

On Mac, you can use `brew`:

```text
brew install gmp
```

## Disclaimer

- Devnet should not be used as a replacement for Alpha testnet. After testing on Devnet, be sure to test on testnet (alpha-goerli)!
- Specifying a block by its hash/number is not supported. All interaction is done with the latest block.
- Read more in [interaction](#interaction).

## Run

Installing the package adds the `starknet-devnet` command.

```text
usage: starknet-devnet [-h] [-v] [--host HOST] [--port PORT]

Run a local instance of Starknet Devnet

optional arguments:
  -h, --help            show this help message and exit
  -v, --version         Print the version
  --host HOST           Specify the address to listen at; defaults to localhost (use the address the program outputs on start)
  --port PORT, -p PORT  Specify the port to listen at; defaults to 8080
  --load-path LOAD_PATH
                        Specify the path from which the state is loaded on
                        startup
  --dump-path DUMP_PATH
                        Specify the path to dump to
  --dump-on DUMP_ON     Specify when to dump; can dump on: exit, transaction
```

You can run `starknet-devnet` in a separate shell, or you can run it in background with `starknet-devnet &`. 
Check that it's alive by running the following (address and port my vary if you specified a different one with `--host` or `--port`):

```
curl http://localhost:8080/is_alive
```

### Run with Docker

Devnet is available as a Docker image ([shardlabs/starknet-devnet](https://hub.docker.com/repository/docker/shardlabs/starknet-devnet)):

#### Versions and Tags

Image tags correspond to Devnet versions as on PyPI and GitHub, with the `latest` tag used for the latest image. These images are built for linux/amd64. To use the arm64 versions, since `0.1.23` you can append `-arm` to the tag. E.g.:

- `shardlabs/starknet-devnet:0.1.23` - image for the amd64 architecture
- `shardlabs/starknet-devnet:0.1.23-arm` - image for the arm64 architecture

```text
docker pull shardlabs/starknet-devnet
```

The server inside the container listens to the port 8080, which you need to publish to a desired `<PORT>` on your host machine:

```text
docker run -p [HOST:]<PORT>:8080 shardlabs/starknet-devnet
```

E.g. if you want to use your host machine's `127.0.0.1:8080`, you need to run:

```text
docker run -p 127.0.0.1:8080:8080 shardlabs/starknet-devnet
```

You may ignore any address-related output logged on container startup (e.g. `Running on all addresses` or `Running on http://172.17.0.2:8080`). What you will use is what you specified with the `-p` argument.

If you don't specify the `HOST` part, the server will indeed be available on all of your host machine's addresses (localhost, local network IP, etc.), which may present a security issue if you don't want anyone from the local network to access your Devnet instance.

## Interaction

- Interact with Devnet as you would with the official Starknet [Alpha testnet](https://www.cairo-lang.org/docs/hello_starknet/amm.html?highlight=alpha#interaction-examples).
- The exact underlying API is not exposed for the same reason Alpha testnet does not expose it.
- To use Devnet with Starknet CLI, provide Devnet's URL to the `--gateway_url` and `--feeder_gateway_url` options of Starknet CLI commands.
- The following Starknet CLI commands are supported:
  - `call`
  - `deploy`
  - `estimate_fee`
  - `get_block`
  - `get_code`
  - `get_full_contract`
  - `get_state_update`
  - `get_storage_at`
  - `get_transaction`
  - `get_transaction_receipt`
  - `get_transaction_trace`
  - `invoke` (currently will fail for max_fee > 0)
  - `tx_status`
- The following Starknet CLI commands are **not** supported:
  - `get_contract_addresses`

## Hardhat integration

- If you're using [the Hardhat plugin](https://github.com/Shard-Labs/starknet-hardhat-plugin), see [here](https://github.com/Shard-Labs/starknet-hardhat-plugin#testing-network) on how to edit its config file to integrate Devnet.

## Postman integration

Postman is a Starknet utility that allows testing L1 <> L2 interactions. To utilize this, you can use [`starknet-hardhat-plugin`](https://github.com/Shard-Labs/starknet-hardhat-plugin), as witnessed in [this example](https://github.com/Shard-Labs/starknet-hardhat-example/blob/master/test/postman.test.ts). Or you can directly interact with the two Postman-specific endpoints:

- Load a `StarknetMockMessaging` contract. The `address` parameter is optional; if provided, the `StarknetMockMessaging` contract will be fetched from that address, otherwise a new one will be deployed:

  - `POST /postman/load_l1_messaging_contract`
  - body: `{ "networkUrl": "http://localhost:5005", "address": "0x83D76591560d9CD02CE16c060c92118d19F996b3" }`
  - `networkUrl` - the URL of the L1 network you've run locally or that already exists; possibilities include, and are not limited to:
    - [Goerli testnet](https://goerli.net/)
    - [Ganache node](https://www.npmjs.com/package/ganache)
    - [Hardhat node](https://hardhat.org/hardhat-network/#running-stand-alone-in-order-to-support-wallets-and-other-software).

- Flush. This will go through the new enqueued messages sent from L1 and send them to L2. This has to be done manually for L1 -> L2, but for L2 -> L1, it is done automatically:
  - `POST /postman/flush`
  - body: None

This method of L1 <> L2 communication testing differs from Starknet Alpha networks. Taking the [L1L2Example.sol](https://www.cairo-lang.org/docs/_static/L1L2Example.sol) contract in the [starknet documentation](https://www.cairo-lang.org/docs/hello_starknet/l1l2.html):

```
constructor(IStarknetCore starknetCore_) public {
        starknetCore = starknetCore_;
}
```

The constructor takes an `IStarknetCore` contract as argument, however for Devnet L1 <> L2 communication testing, this will have to be replaced with the [MockStarknetMessaging.sol](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/starknet/testing/MockStarknetMessaging.sol) contract:

```
constructor(MockStarknetMessaging mockStarknetMessaging_) public {
    starknetCore = mockStarknetMessaging_;
}
```

## Dumping

To preserve your Devnet instance for future use, there are several options:

- Dumping on exit (handles Ctrl+C, i.e. SIGINT, doesn't handle SIGKILL):

```
starknet-devnet --dump-on exit --dump-path <PATH>
```

- Dumping after each transaction (done in background, doesn't block):

```
starknet-devnet --dump-on transaction --dump-path <PATH>
```

- Dumping on request (replace `<HOST>`, `<PORT>` and `<PATH>` with your own):

```
curl -X POST http://<HOST>:<PORT>/dump -d '{ "path": <PATH> }' -H "Content-Type: application/json"
```

### Loading

To load a preserved Devnet instance, run:

```
starknet-devnet --load-path <PATH>
```

### Enabling dumping and loading with Docker

To enable dumping and loading if running Devnet in a Docker container, you must bind the container path with the path on your host machine.

This example:

- Relies on [Docker bind mount](https://docs.docker.com/storage/bind-mounts/); try [Docker volume](https://docs.docker.com/storage/volumes/) instead.
- Assumes that `/actual/dumpdir` exists. If unsure, use absolute paths.
- Assumes you are listening on `127.0.0.1:8080`.

If there is `dump.pkl` inside `/actual/dumpdir`, you can load it with:

```
docker run \
  -p 127.0.0.1:8080:8080 \
  --mount type=bind,source=/actual/dumpdir,target=/dumpdir \
  shardlabs/starknet-devnet \
  --load-path /dumpdir/dump.pkl
```

To dump to `/actual/dumpdir/dump.pkl` on Devnet shutdown, run:

```
docker run \
  -p 127.0.0.1:8080:8080 \
  --mount type=bind,source=/actual/dumpdir,target=/dumpdir \
  shardlabs/starknet-devnet \
  --dump-on exit --dump-path /dumpdir/dump.pkl
```

## Block explorer

A local block explorer (Voyager), as noted [here](https://voyager.online/local-version/), apparently cannot be set up to work with Devnet. Read more in [this issue](https://github.com/Shard-Labs/starknet-devnet/issues/60).

## Lite mode

To improve Devnet performance, consider passing these CLI flags on Devnet startup:

- `--lite-mode` enables all of the optimizations described below (same as using all of the following flags);
- `--lite-mode-deploy-hash` disables the calculation of the transaction hash for deploy transactions. It will instead be a simple sequence of numbers;
- `--lite-mode-block-hash` disables the calculation of the block hash. It will instead be a simple sequence of numbers;

## Restart

Devnet can be restarted by making a `POST /restart` request. All of the deployed contracts, blocks and storage updates will be restarted to the empty state.

## Contract debugging

If your contract is using `print` in cairo hints (it was compiled with the `--disable-hint-validation` flag), Devnet will output those lines together with its regular server output. To filter out just your debugging print lines, redirect stderr to /dev/null when starting Devnet:

```
starknet-devnet 2> /dev/null
```

To enable printing with a dockerized version of Devnet set `PYTHONUNBUFFERED=1`:

```
docker run -p 127.0.0.1:8080:8080 -e PYTHONUNBUFFERED=1 shardlabs/starknet-devnet
```

## Development

If you're a developer willing to contribute, be sure to have installed [Poetry](https://pypi.org/project/poetry/) and all the dependency packages.

```text
pip install poetry
poetry install
```

### Development - Run

```text
poetry run starknet-devnet
```

### Development - Test

When running tests locally, do it from the project root:

```text
poetry run pytest test/
```

or for a single file

```text
poetry run pytest test/<TEST_FILE>
```

### Development - Build

You don't need to build anything to be able to run locally, but if you need the `*.whl` or `*.tar.gz` artifacts, run

```text
poetry build
```
