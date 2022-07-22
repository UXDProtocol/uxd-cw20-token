# Procedure to deploy token contract

## Prerequisites

A local seid node should be already setup and running in the background.
For setup: instructions [here](https://github.com/UXDProtocol/sei-chain/blob/master/uxd_procedure.md)

then run in a separate terminal: `seid start`

## Build

Unit-tests

```sh
RUST_BACKTRACE=1 cargo unit-test
```

Release build:

```sh
cargo wasm
```

Release build, stripped of all the unneeded code, for smallare file size:

```sh
RUSTFLAGS='-C link-arg=-s' cargo wasm
```

Optimized Compilation

```sh
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/rust-optimizer:0.12.6
```

## Upload

Check current list of codes uploaded. here we assume the chain-id was set to `sei-chain`.

```sh
# seid query wasm list-code
seid query wasm list-code --chain-id sei-chain
```

Upload to the chain

```sh
seid tx wasm store artifacts/uxd_cw20_token.wasm --from $ACCOUNT_NAME  --chain-id sei-chain --gas=4000000 --fees=1000000usei -y --broadcast-mode block --output json > artifacts/store.json

CW20_CODE_ID=$(jq -r '.logs[0].events[-1].attributes[0].value' < artifacts/store.json)
echo $CW20_CODE_ID
```

Check that version of contract is not instantiated yet

```sh
seid query wasm list-contract-by-code $CW20_CODE_ID --chain-id sei-chain  --output json
```

```sh
# Download the wasm binary from the chain and compare it to the original one
seid query wasm code $CW20_CODE_ID --chain-id sei-chain artifacts/download.wasm
# The two binaries should be identical
diff artifacts/uxd_cw20_token.wasm artifacts/download.wasm
# diff should be null
```

## Instantiate manually

```sh
# Prepare the instantiation message
# INIT='{"name":"UXD","symbol":"UXD","decimals":6,"initial_balances":[]}'
INIT_MINTABLE='{"name":"UXD","symbol":"UXD","decimals":6,"initial_balances":[],"mint":{"minter":"'$ACCOUNT_ADDRESS'"}}'
echo $INIT_MINTABLE

# Instantiate the contract
seid tx wasm instantiate $CW20_CODE_ID "$INIT_MINTABLE"  --chain-id sei-chain --from $ACCOUNT_NAME --label "uxd-mintable" --gas=5000000 --fees=1000000usei -y --no-admin --broadcast-mode=block --output json > artifacts/mintable.json

# Check the contract details and account balance
seid query wasm list-contract-by-code $CW20_CODE_ID  --chain-id sei-chain --output json
CONTRACT=$(seid query wasm list-contract-by-code $CW20_CODE_ID --output json | jq -r '.contracts[-1]')
echo $CONTRACT

# See the contract details
seid query wasm contract $CONTRACT
# Check the contract balance
seid query bank balances $CONTRACT

# Query the entire contract state
seid query wasm contract-state all $CONTRACT

# Query different information about the contract
INFO_QUERY='{"token_info": {}}'
ALL_ACC_QUERY='{"all_accounts": {}}'
BALANCE_QUERY='{"balance": {"address":"'$ACCOUNT_ADDRESS'"}}'

seid query wasm contract-state smart $CONTRACT "$INFO_QUERY" --chain-id sei-chain --output json
seid query wasm contract-state smart $CONTRACT "$ALL_ACC_QUERY" --chain-id sei-chain --output json
seid query wasm contract-state smart $CONTRACT "$BALANCE_QUERY" --chain-id sei-chain --output json

# Execute a mint to an address
EXEC_MINT='{"mint": {"recipient":"'$ACCOUNT_ADDRESS'","amount":"100000"}}'

seid tx wasm execute $CONTRACT "$EXEC_MINT" --from $ACCOUNT_NAME --gas=4000000 --fees=1000000usei -y --broadcast-mode=block  --chain-id sei-chain --output json >> artifacts/exec_mint.json

# Execute a transfer to Alice
EXEC_TRANSFER='{"transfer": {"recipient":"'$ALICE_ADD'","amount":"5000"}}'

seid tx wasm execute $CONTRACT "$EXEC_TRANSFER" --from $ACCOUNT_NAME --gas=4000000 --fees=1000000usei -y --broadcast-mode=block  --chain-id sei-chain --output json >> artifacts/exec_transfer.json

ALICE_BALANCE='{"balance": {"address":"'$ALICE_ADD'"}}'
seid query wasm contract-state smart $CONTRACT "$ALICE_BALANCE" --chain-id sei-chain --output json

# Execute a burn
EXEC_BURN='{"burn": {"amount":"5000"}}'

seid tx wasm execute $CONTRACT "$EXEC_BURN" --from $ACCOUNT_NAME --gas=4000000 --fees=1000000usei -y --broadcast-mode=block  --chain-id sei-chain --output json >> artifacts/exec_burn.json

```
