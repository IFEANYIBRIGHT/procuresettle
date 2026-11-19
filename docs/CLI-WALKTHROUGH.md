# ProcureSettle CLI walkthrough

This walkthrough uses Stellar CLI 28.1.0 on a local standalone network. It
creates three test identities, deploys a Stellar Asset Contract (SAC) and the
ProcureSettle contract, then creates, accepts, and funds purchase order 101.

The example PO has one line: line ID 1, ordered quantity 50, and unit price
10,000,000. Its stored total is 500,000,000 token base units. Quantities and
unit prices are integers; the CLI encodes the `i128` unit price as a JSON
string.

Run every command from the repository root in the same shell.

## Prerequisites

Install Stellar CLI 28.1.0, Rust 1.95.0 with the `wasm32v1-none` target, and
Docker for the local Quickstart network. Confirm the CLI version:

```sh
set -euo pipefail

export CONFIG_DIR="$PWD/.stellar-walkthrough"
export NETWORK=local
export WASM="$PWD/target/wasm32v1-none/release/procuresettle.wasm"

export PO_ID=101
export LINE_ID=1
export ORDERED_QUANTITY=50
export UNIT_PRICE=10000000
export ISSUE_AMOUNT=1000000000

mkdir -p "$CONFIG_DIR"

stellar --config-dir "$CONFIG_DIR" --version
```

The first version line should report `stellar 28.1.0`.

## Start the local network

```sh
stellar --config-dir "$CONFIG_DIR" container start local \
  --name procuresettle-walkthrough \
  --protocol-version 28

stellar --config-dir "$CONFIG_DIR" network use local
stellar --config-dir "$CONFIG_DIR" network info
```

The network information should show passphrase
`Standalone Network ; February 2017` and protocol version 28.

## Build the contract

```sh
stellar contract build --locked --package procuresettle

test -f "$WASM"
```

## Generate test keys

Create separate identities for the issuer, buyer, and supplier. Friendbot funds
each local account.

```sh
for identity in issuer buyer supplier; do
  stellar --config-dir "$CONFIG_DIR" keys generate "$identity" \
    --network "$NETWORK" --fund
done

export ISSUER="$(stellar --config-dir "$CONFIG_DIR" keys public-key issuer)"
export BUYER="$(stellar --config-dir "$CONFIG_DIR" keys public-key buyer)"
export SUPPLIER="$(stellar --config-dir "$CONFIG_DIR" keys public-key supplier)"

printf 'issuer:   %s\nbuyer:    %s\nsupplier: %s\n' \
  "$ISSUER" "$BUYER" "$SUPPLIER"
```

## Deploy the token SAC

The procurement token is the classic asset `PSET`, issued by the test issuer.
The buyer and supplier need trustlines before the issuer credits the buyer and
the SAC is deployed.

```sh
export ASSET="PSET:$ISSUER"

stellar --config-dir "$CONFIG_DIR" tx new change-trust \
  --source-account buyer \
  --line "$ASSET" \
  --network "$NETWORK"

stellar --config-dir "$CONFIG_DIR" tx new change-trust \
  --source-account supplier \
  --line "$ASSET" \
  --network "$NETWORK"

stellar --config-dir "$CONFIG_DIR" tx new payment \
  --source-account issuer \
  --destination "$BUYER" \
  --asset "$ASSET" \
  --amount "$ISSUE_AMOUNT" \
  --network "$NETWORK"

export SAC_ID="$(
  stellar --config-dir "$CONFIG_DIR" contract asset deploy \
    --asset "$ASSET" \
    --source-account issuer \
    --network "$NETWORK"
)"

printf 'SAC: %s\n' "$SAC_ID"
```

## Deploy ProcureSettle

The issuer deploys the contract in this walkthrough; the issuer is not a party
to the PO.

```sh
export CONTRACT_ID="$(
  stellar --config-dir "$CONFIG_DIR" contract deploy \
    --wasm "$WASM" \
    --source-account issuer \
    --network "$NETWORK"
)"

printf 'contract: %s\n' "$CONTRACT_ID"
```

## Create PO 101

The buyer creates the PO and must authorize the call. The `--lines` argument is
one compact JSON array.

```sh
export LINES="[{\"line_id\":${LINE_ID},\"ordered_quantity\":${ORDERED_QUANTITY},\"unit_price\":\"${UNIT_PRICE}\"}]"

stellar --config-dir "$CONFIG_DIR" contract invoke \
  --contract-id "$CONTRACT_ID" \
  --source-account buyer \
  --network "$NETWORK" \
  -- create_po \
  --po_id "$PO_ID" \
  --buyer "$BUYER" \
  --supplier "$SUPPLIER" \
  --token "$SAC_ID" \
  --lines "$LINES"

stellar --config-dir "$CONFIG_DIR" contract invoke \
  --contract-id "$CONTRACT_ID" \
  --source-account buyer \
  --network "$NETWORK" \
  -- get_po \
  --po_id "$PO_ID"
```

The `get_po` result should show status `Draft` and `total_value` of
`500000000`. A read-only invocation may print a simulation note about rerunning
with `--send=yes`; that note does not indicate an error.

## Accept the PO as supplier

Only the recorded supplier can move the PO from `Draft` to `Accepted`.

```sh
stellar --config-dir "$CONFIG_DIR" contract invoke \
  --contract-id "$CONTRACT_ID" \
  --source-account supplier \
  --network "$NETWORK" \
  -- accept_po \
  --po_id "$PO_ID"

stellar --config-dir "$CONFIG_DIR" contract invoke \
  --contract-id "$CONTRACT_ID" \
  --source-account supplier \
  --network "$NETWORK" \
  -- get_po \
  --po_id "$PO_ID"
```

The status should now be `Accepted`.

## Fund the PO as buyer

`fund_po` transfers the full stored total from the buyer to the contract. The
buyer must have at least 500,000,000 base units of `PSET`; the issuance above
provides 1,000,000,000.

```sh
stellar --config-dir "$CONFIG_DIR" contract invoke \
  --contract-id "$CONTRACT_ID" \
  --source-account buyer \
  --network "$NETWORK" \
  -- fund_po \
  --po_id "$PO_ID"

stellar --config-dir "$CONFIG_DIR" contract invoke \
  --contract-id "$CONTRACT_ID" \
  --source-account buyer \
  --network "$NETWORK" \
  -- get_po \
  --po_id "$PO_ID"

stellar --config-dir "$CONFIG_DIR" token balance \
  --id "$SAC_ID" \
  --account "$CONTRACT_ID" \
  --network "$NETWORK"
```

The PO should now show status `Funded` and `funded_value` of `500000000`. The
contract's SAC balance should be `500000000`.

## Clean up

Stop the local container when the walkthrough is complete:

```sh
stellar --config-dir "$CONFIG_DIR" container stop procuresettle-walkthrough
```

This walkthrough stops after funding. For receipt submission, acceptance,
settlement, cancellation, and refund commands, see the
[local flagship proof transcript](../evidence/local-2026-11-06/local-flagship-proof.txt)
and the [evidence record](EVIDENCE.md).
