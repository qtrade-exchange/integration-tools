# qTrade Systems Overview

As mentioned in the Development Guidelines, qTrade has 4 primary components:

 - Website: `frontend`
 - Trade engine & DB: `backend`
 - Currency network monitor: `watcher`
 - Isolated currency signing system: `signer`

This document will attempt to provide a comprehensize explaination of the tasks performed by the `watcher` and `signer`, as well as provide some example flows for deposits and withdraws.

Throughout this document you'll see numerous references to well known currencies and their RPC calls. We want to be clear that we're not suggesting you build or modify your project to match currency XYZ, but it is useful to reference well known RPCs, like Bitcoin's, since most developers in this space are familiar with it.

## Isolated Systems

The `signer` and `watcher` programs are highly isolated from eachother, and from
the actual p2p-node software. Effectively they only have HTTP API
inter-communication possible, and have no access to eachothers filesystem,
processes, etc. The `watcher` connectes to the fully synced p2p-node, but the signer
does not. Below we outline the tasks that each component runs to clarify
general implemenation.

### Watcher

* watch addresses - We want to track the network state and network events for
  some addresses. New addresses are created in the signer, so they must be pulled
  from the API and imported into the `watcher` and typically in the p2p node software.
  In bitcoin this is done with `importaddress`.
* discover deposits - We discover new deposit transactions and load them into
  our system as pending. For Bitcoin we use the `listsinceblock` and/or
  `listunspent` RPC call. We can also write a script to scan a the blockchain
  directly- but that is far more work, and can raise rollback related edge cases.
* confirm deposits - We grab all pending deposits and lookup their transaction
  ID to discover if they are safe to credit to our users. Bitcoin's
  implementation uses `gettransaction`
* create withdrdraw transaction - Assenble any info necessary for the creation
  and signing of a transaction. In Bitcoin's implementation we use
  `createrawtransaction`.
* send withdraw transaction - we broadcast a signed transaction to the network.
  Bitcoin uses `sendrawtransaction`
* confirm withdraw transaction - we verify that the transaction was mined into a
  block. Similar implementation to "confirm deposit"

### Signer

* generate address - A new key pair is generated and the pubkey is pushed
  to the address pool. For currencies with data fields this is instead a data string
* sign withdraw transaction - we sign the raw transaction without connection to
  a networked coinserver, and preferrably no coinserver at all. Consider this
  step as being completely offline. For Ethereum or other account based
  cryptocurrencies we serialize the transaction for signing in the "create"
  step, and the signer simply needs to sign the serialized payload, without ever
  connecting with other software. In some cases a simple "signing utility" such
  as `bitcoin-tx` is acceptable if the signing process is more complex.


### Example withdraw flows

#### Bitcoin (UTXO based) withdraws:

 - The `watcher` calls `createrawtransaction` with specified inputs and outputs, and saves the unsigned transaction template and some metadata
 - The `signer` gets that template and runs `signrawtransaction` and saves the signed transaction
 - The `watcher` broadcasts the signed transaction via `sendrawtransaction` 
 
 Ideally `createrawtransaction` would output a byte string (or array of byte strings) to be signed. This is desirable because then our `signer` doesn't need to understand anything about transaction formats, etc, it simply has to sign a byte string.

#### Nano (Account based) withdraws:

 - The `watcher` calls `account_info` to get the information about an account necessary to build a (send or receive) block. This info includes the top height hash of that account's blockchain, preventing double spends.
 - The `signer` builds a block via `create_block`, which includes a signature.
 - The `watcher` broadcasts the signed block via the `process` RPC endpoint

 Ideally there would not be the need for the `create_block` call in the signer, as this adds 3rd party code to the `signer`, but (at the time of writing this) there is poor support for this via the RPC, and no documentation on how to generate the signatures/PoW required.
  
#### Takeaways

 - As you can see, these flows guarantee idempotency - as the transaction being transmitted is always the same.
 - Likewise, only having support for a `send` is insufficient since network interruptions can lead to double sent funds, and private keys for customer funds cannot be properly isolated from third party software.
 - Maintaining strong guarantees about idempotency and secret key isolation is a hard requirement for all currencies we list.
 - Transaction portability makes things easier for integrating