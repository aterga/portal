# Chain-key Bitcoin (ckBTC)

## Overview

Chain-key Bitcoin (ckBTC) is an [ICRC-1](https://github.com/dfinity/ICRC-1/blob/aa82e52aaa74cc7c5f6a141e30b708bf42ede1e3/standards/ICRC-1/README.md)-compliant token that is backed 1:1 by Bitcoin held 100% on the IC mainnet.

The ckBTC functionality is provided through an interplay of two canisters:
- The **ckBTC minter**. 
- The **ckBTC ledger**.

The ckBTC ledger is responsible for keeping account balances and for transferring ckBTC between accounts. It provides the following functionality:
- It enables the ckBTC minter to mint and burn ckBTC.
- It enables the transfer of ckBTC among users.

The ckBTC minter is responsible for the minting and burning of ckBTC tokens. A certain amount of tokens is minted if a user transfers the same amount of bitcoin to a specific Bitcoin address under the ckBTC minter's control. The Bitcoin address uniquely identifies the owner of the sent bitcoins, making it possible for the ckBTC minter to associate the minted ckBTC funds with the correct owner. The ckBTC minter needs to receive BTC, based on a large number of confirmations due to the lack of finality in Bitcoin, before it mints the same amount in ckBTC. The ckBTC minter burns the ckBTC before it transfers the corresponding BTC amount using a regular Bitcoin transaction.

A detailed description of the ckBTC minter can be found in its [GitHub repository](https://github.com/dfinity/ic/tree/master/rs/bitcoin/ckbtc/minter).

A simplified overview of the process to mint and transfer ckBTC is depicted in the following figure.

![ckBTC overview](../../../samples/_attachments/ckbtc-overview.png)

For a deeper technical overview, please see [here](bitcoin-how-it-works.md).

This page details the API endpoints that can be used to interact with the ckBTC minter canister. 

## ckBTC ledger
The ckBTC ledger accepts minting and burning requests from the ckBTC minter and records the ckBTC balances of every account with a positive balance. Additionally, the ckBTC ledger handles the ckBTC transactions.

The ckBTC ledger adheres to the ICRC-1 token standard. Technical details can be found on the GitHub repository of the used [ICRC-1 ledger implementation](https://github.com/dfinity/ic/tree/master/rs/rosetta-api/icrc1).

## ckBTC minter
The ckBTC minter is a canister that is controlled by the NNS and running on the [pzp6e](https://dashboard.internetcomputer.org/subnet/pzp6e-ekpqk-3c5x7-2h6so-njoeq-mt45d-h3h6c-q3mxf-vpeq5-fk5o7-yae) subnet. In the configuration of the ckBTC minter canister, the following configurations are set:

- `retrieve_btc_min_amount`: This is the minimum ckBTC amount that can be burned and, correspondingly, the minimum BTC amount that can be withdrawn. The parameter is set to 0.001 BTC, or 100,000 Satoshi.
- `max_time_in_queue_nanos`: Any BTC retrieval request should be kept in a queue for at most this time. Caching requests rather than handling them right away has the advantage that multiple requests can be served in a single transaction, saving Bitcoin miner fees. The parameter is set to 10 minutes, which corresponds to the expected time between Bitcoin blocks.
- `min_confirmations`: The number of confirmations required for the ckBTC minter to accept a Bitcoin transaction. In particular, the ckBTC minter does not mint ckBTC before a transaction transferring BTC to a Bitcoin address managed by the ckBTC minter reaches this number of transactions. The parameter was initially set to 72 but has been reduced to 12 in the meantime.
- `kyt_fee`: The fee that must be paid for KYT checks. It is currently set to 2000 satoshi.

There are other parameters that are self-explanatory and can be found in the [ckBTC minter Candid file](https://github.com/dfinity/ic/blob/master/rs/bitcoin/ckbtc/minter/ckbtc_minter.did).

## ckBTC minter API endpoints
The ckBTC minter provides the following API endpoints that can be used to interact with the canister:

- `get_btc_address`: Returns a specific Bitcoin address that the caller can use to obtain ckBTC by sending BTC to this address.
- `track_balance`: Instructs the ckBTC minter to track a certain Bitcoin address until funds are added, which will trigger the minting of ckBTC. NOTE: This endpoint is work-in-progress and not available yet.
- `update_balance`: Instructs the ckBTC minter to check the balance of a Bitcoin address and mint ckBTC into the account of the owner.
- `estimate_withdrawal_fee`: Returns a current estimate for the fee to be paid when retrieving a certain BTC amount.
- `get_deposit_fee`: Returns the fee charged when minting ckBTC.
- `get_withdrawal_account`: Returns a specific ckBTC account where the owner must transfer ckBTC before being able to retrieve BTC.
- `retrieve_btc`: Instructs the ckBTC minter to burn a certain ckBTC amount and send the corresponding BTC amount, minus fees, to a provided Bitcoin address.
- `retrieve_btc_status`: Returns the status of a previous retrieve_btc call.
- `get_minter_info`: Returns information about the ckBTC minter itself.
- `get_events`: Returns a set of events at the ckBTC minter.
The endpoints are discussed in more detail in the following.

### `get_btc_address(owner: opt principal, subaccount: opt blob)`
The provided principal ID and subaccount are concatenated to form the derivation path for the [ecdsa_public_key](https://internetcomputer.org/docs/current/references/ic-interface-spec#ic-ecdsa_public_key) function, which returns the derived public key. If no principal ID is provided, then the sender’s principal ID is used. If no subaccount is provided, then the default subaccount (all zeros) is used.

This public key is encoded as a pay-to-witness-public-key-hash (P2WPKH) Bitcoin address and returned as a text.

Note that the key derivation is not BIP-32 compliant where 31 bits are used for each derivation level. Instead, a single derivation is performed based on the full principal ID and subaccount. Since the derivation is deterministic, a canister can derive the Bitcoin address for a given principal ID and subaccount itself.

### `track_balance(owner: opt principal, subaccount: opt blob)`
:::info
This endpoint is work-in-progress and not available yet.
:::

The ckBTC minter starts tracking the Bitcoin address derived from the provided principal ID and subaccount using the `get_btc_address` endpoint. If no principal ID is provided, then the sender’s principal ID is used. If no subaccount is provided, then the default subaccount (all zeros) is used.

The balance of the Bitcoin address is not tracked indefinitely. Tracking is stopped if either at least one new unspent transaction output (UTXO) is discovered or there is no new UTXO within a certain time interval (details about balance tracking are provided below).

If at least one new UTXO is discovered, the same amount of ckBTC tokens minus the KYT fee are minted and made available to the owner.

### `update_balance(owner: opt principal, subaccount: opt blob)`
Instead of having the ckBTC minter track the balance of a Bitcoin address, the `update_balance` function can be invoked to instruct the ckBTC minter to check if there are new UTXOs for a particular Bitcoin address.

If there is at least one new UTXO, the corresponding ckBTC amount is minted, otherwise an error is returned.

The ckBTC minter effectively invokes this endpoint itself on a timer when the `track_balance` function is used.

### `estimate_withdrawal_fee(amount: opt nat64)`
The endpoint returns an estimate for the fee that must be paid when retrieving the given BTC amount. Internally, a transaction is built (without valid signatures) to determine the fee, which consists of the Bitcoin miner fee, the ckBTC minter fee, and the KYT fee. If no amount is provided, it is assumed that three inputs are required to build the transaction.

If there is no change to the internal state of the ckBTC minter and the Bitcoin canister before issuing the request to retrieve Bitcoins, the fee will be exactly the returned estimate.

The fee can change when a new Bitcoin block is mined in the meantime, which causes the Bitcoin canister to update the Bitcoin miner fees or when another retrieval request is handled first, spending some of the outputs that were used when estimating the fee.

### `get_deposit_fee`
The endpoint returns the fee that the ckBTC minter charges for minting ckBTC when receiving new UTXOs. Currently, this fee is simply the KYT fee.

### `get_withdrawal_account`
The function returns the caller’s withdrawal account, which is the account derived from the ckBTC minter’s principal ID and the subaccount derived from the caller’s principal ID.

A user can only retrieve BTC by first transferring ckBTC to this particular account.

### `retrieve_btc(address: text, amount: nat64)`
The function instructs the ckBTC minter to burn the specified amount. Once the request is picked up from the queue, a Bitcoin transaction is created containing an output that transfers the requested amount (minus fees as discussed below) to the specified Bitcoin address.

Since Bitcoin retrieval is handled asynchronously, the function returns the block index of the transaction burning the ckBTC tokens.

### `retrieve_btc_status(block_index: nat64)`
The status of a BTC retrieval request can be checked using this function. The different possible statuses are:

- `Unknown`: There is no information available on the ckBTC minter because there is no retrieval request associated with the given block index or the retrieval request is old and the corresponding information has already been removed.
- `Pending`: The BTC retrieval request is queued.
- `Signing`: The BTC retrieval request is acquiring all signatures to serve the BTC retrieval request.
- `Sending`: The Bitcoin transaction serving the BTC retrieval request is being sent.
- `Submitted`: The Bitcoin transaction has been transmitted. The transaction ID is returned.
- `AmountTooLow`: The BTC retrieval request could not be served because the Bitcoin miner fees are prohibitively high.
- `Confirmed`: The Bitcoin transaction serving the BTC retrieval request is confirmed inside the ckBTC minter, which happens when the transaction has at least the minimum required number of confirmations specified in the `min_confirmations` parameter above.

### `get_minter_info`
The function returns the following parameters internal to the ckBTC minter:

- `kyt_fee`
- `min_confirmations`
- `retrieve_btc_min_amount`

### `get_events(start: nat64, length: nat64)`
The ckBTC minter tracks the events that change its internal state. Given a start index and a length parameter, the ckBTC minter returns all events sequentially from the event at the given start index up to the event at index start+length-1.

Note that this endpoint is used for debugging purposes and there is no guarantee that the endpoint will continue to exist in this form.

## Resources

To start building your own apps with Bitcoin see the following tutorials:

- [ckBTC wiki page](https://wiki.internetcomputer.org/wiki/Chain-key_Bitcoin).
- [Deploying your first Bitcoin dapp](../../../samples/deploying-your-first-bitcoin-dapp.md).
- [GitHub repository](https://github.com/dfinity/ic/tree/master/rs/bitcoin/ckbtc/minter).
- [Local development](./local-development.md).
