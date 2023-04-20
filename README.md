# liqwid-liquidation-bot docker image

## Overview

The liquidation bot docker image. The bot will continuously loop at a
customizable loop interval. If the bot finds one or more liquidatable positions,
it will execute liquidation transactions beginning at the position with the
highest reward and proceeding down the list of all liquidatable positions.
If a single liquidation fails (such as due to UTxO contention), the bot will
automatically proceed directly to the next highest position.

> NOTE: Users should expect some error messages in their logs. These are
> primarily due to unavoidable UTxO contention and race conditions in the bot.
> We are working on providing better error messages to end-users to distinguish
> between "normal" and "abnormal" exceptions, and will provide updates when this
> work is complete. In the interim, as long as the bot is not exiting prematurely
> and not appearing to stall during a transaction (excluding the wait time
> configured in the `INTERVAL` environment variable), it is likely that the bot is
> working as intended.

Each liquidation bot will liquidate a single market only. If you wish to liquidate
loans in multiple markets, you must run two separate batching bots, set up
individual wallets, and set up their env-files (details below) accordingly.

## Usage

After modifying the env-file, run:

```
docker pull ghcr.io/liqwid-labs/liqwid-liquidation-bot:mainnet
docker run --env-file liquidationBotEnv liqwid-liquidation-bot:mainnet
```

> NOTE: Due to the static parameterization of the protocol two docker images of the
> bot are provided. One image intended for the mainnet and the other intended for
> the preview testnet. Using the incorrect image for the network will result in the
> bot not being able to identify the protocol and subsequently failing.

To interact with the image, modify the environment variables (as described below)
in the `liquidationBotEnv` file.

### Required keys

> NOTE: The key generation process described below will produce _sensitive data_.
> Make sure you properly store the keys, and generate them on a machine you trust.

Users must provide a wallet key in order to sign transactions.

You can generate a new key with `cardano-cli`. For example:

```sh
# Enter a nix shell with the cardano cli available
nix shell 'github:input-output-hk/cardano-node#cardano-cli'

# Generate signing and verification keys
cardano-cli address key-gen --signing-key-file key.skey --verification-key-file key.vkey

# Extract the address for initial funding of the wallet; choose either the mainnet or `preview` testnet via the last option.
cardano-cli address build --payment-verification-key-file key.vkey --out-file payment.addr [--mainnet | --testnet-magic 2]
```

NOTE: Once generated the `payment.addr` must be funded in order to run the bot.
The wallet will require the "underlying" currency of the market to liquidation
loans within that market; i.e., ADA is required to liquidate an ADA loan, and
DJED (plus ADA for transaction fees) is required to liquidate a DJED loan.

Liquidations rewards (either qADA, qDJED, or both) will be paid back to the wallet.
These rewards _are not_ automatically redeemed; you must redeem them manually.
Future version of the bot will include a default redemption strategy.

The environment variable expected by the docker image is then `SKEY` which is the `cborHex` field of the key.skey file:

> NOTE: No environment variables require quotes. `cborHex_signing_key` and NOT `"cborHex_signing_key"`.

```
SKEY=cborHex_signing_key
```

### Optional services

The bot needs access to Kupo, Ogmios and Ogmios Datum Cache.

> NOTE: If a user does not provide their own configuration, the bot will use the default Liqwid services.

These can be configured via the following environment variables:

```
OGMIOS_PORT=1337
OGMIOS_HOST=localhost
OGMIOS_SECURE=false
OGMIOS_PATH=

DATUM_CACHE_PORT=9999
DATUM_CACHE_HOST=localhost
DATUM_CACHE_SECURE=false
DATUM_CACHE_PATH=

KUPO_PORT=1442
KUPO_HOST=localhost
KUPO_SECURE=false
KUPO_PATH=
```

> NOTE: docker must expose connections to these services.


### Optional params

### Verbosity

Set the level of logging for debugging.

```
VERBOSE=false
```

default: false (No debug logging)

### Interval

Set the liquidation loop interval in number of slots.

```
INTERVAL=20
```

default: 20 slots

### Market

Set the Market with which the bot should run on.

Current available Markets:
- Ada
- DJED
- SHEN

```
MARKET=DJED
```

default: Ada

### Heartbeat

The batching bot exposes a heartbeat which will respond with the string "beat" if the service
is active. Note that this will grab a port; when running multiple bots, you must set separate ports
for each.

```
HEARTBEAT_ADDR=127.0.0.1
HEARTBEAT_PORT=2002
```

default: `127.0.0.1`, port `2002`
