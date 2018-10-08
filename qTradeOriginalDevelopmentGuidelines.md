# qTrade Original Development Guidelines

## Software Integration

An exchange has several specific needs that are different than that of a regular end user or businesses looking to accept crypto as payment. We'll attempt to clearly outline our technical needs, offering guidance on how to make integration with our exchange easier.

### Unsigned transactions

Our exchange requires strong isolation between network connected p2p software (such as bitcoin core), and "signing software" that holds private keys. In practice this means we have two pieces of software: one that creates raw transactions and transmits signed transactions to the network, and another that only signs transactions.

Having only API wallet support for `send` is insufficient since network interruptions can lead to double sent funds, and private keys for customer funds cannot be properly isolated from third party software.

Bitcoin's `createrawtransaction`, `signrawtransaction` and `sendrawtransaction` are good examples. Ideally `createrawtransaction` would output a byte string (or array of byte strings) to be signed. This is desirable because then our signing backend doesn't need to understand anything about transaction formats, etc, it simply has to sign a byte string.

### Digital Signature Algorithm (DSA)

Ideally your currency would support ECDSA signatures on the secp256k1 curve. Since this is what Bitcoin uses, it's widely available and easy to use.

If not ECDSA then selecting a well established and widely trusted algorithm such as RSA is preferred. Something with broad language support, especially Python or GoLang.

This is important for easily supporting cold storage as well, since using YubiKey or Trezor hardware devices becomes easier when using widely adopted standards. Quantum resistance is great, but if it comes at the cost of having to roll our own crypto libraries we're unlikely to be able to support your coin without considerable expense.

### Address generation

Support for hierarchical deterministic keys (HD keys) in some fashion, either
directly, or via import support, is a requirement. This is important to us
because it simplifies our secret management. Instead of potentially thousands of
private keys we only have one. Our last resort backups are on paper to mitigate
hardware-related losses, but printing thousands of keys is undesirable due to
logistics with actually using the backups.

If you support ECDSA secp256k1, ideally your software would allow importing
these keys, both in a signing and watching capacity, similar to
`importaddress` and `importprivkey`. We will then handle all address generation
ourselves, and import the keys that we need. Ideally your import function would
accept raw compressed ECDSA keys, but several other formats are acceptable.

### Deposit scanning

Ideally we would be able to lookup all transactions concerning a certain account or address, preferably in an incremental or progressive fashion. Think something like Bitcoin's `importaddress` functionality and `listsinceblock`. Just getting the current balance for a given address is insufficient, we must be able to discover/detect individual transactions, and we should be provided a unique TXID to facilitate looking up status information in the future. We should be able to lookup this information for addresses for which we do not have the private key, so early versions of bitcoin core that didn't support `importaddress` are not supported for instance, since they require the private key in order to index transactions for a given address.

In the worst case we can scan the blockchain ourselves, in which case we need an easy way to get decoded, easily parse-able list of all transactions for a given block height or hash, and then we can do the transaction indexing ourselves by scanning the blockchain.

### Deposit correlation

Our system needs to be able to link a specific deposit transaction to one of our users. With bitcoin-like currencies this happens by giving each user a unique address for which we hold a private key. For account based systems like ethereum it is preferred that there be the ability to include arbitrary identification data in a transaction. The exchange may then operate using a single account by assigning each user a unique identifier to include in all their deposit transactions.

In account based systems that do not support arbitrary identifiers (like ethereum) we perform an additional "sweep" step to move all deposits to a central wallet account. This process adds complexity in several ways, and ideally would be avoided.

### Transaction Lookup

Provided the TXID we need to be able to lookup if that transaction is now safe from a double spend. For Bitcoin-like blockchains this means confirmations, but any suitable status information can be used. We should have strong guarantees that the transaction is irreversible, and guidance on when it can be considered as such.

### Network Health

Ideally we would have easy access to statistics that give us insight into our nodes health, and the health of the p2p network. For example for Bitcoin we monitor block height, current difficulty, and connected peer count.

### Persistence

Blockchain data's storage location should be configurable, such as with the `-datadir` argument on Bitcoin. If not configurable, it should at least be self-contained in a single folder. Having 6 different folders that get created in the same folder as the binary presents a big headache for putting data directories on separate disks from the executables, which is frequently desirable in cloud environments.

### Push notifications

Push is optional, although nice to have. Anything that allows us send an HTTP request of some kind from the node to our software, similar to `walletnotify` + `curl` in Bitcoin core, should work well.

## Functional Flow

We have two separate programs that are highly isolated from eachother, and from
the actual p2p-node software. Effectively they only have HTTP API
inter-communication possible, and have no access to eachothers filesystem,
processes, etc. We refer to these separate components as the "signer" and
"watcher". The watcher connectes to the fully synced p2p-node, but the signer
does not. Below we outline the tasks that each component runs to clarify
general implemenation.

### Watcher

* watch address - 'generated' addresses are pulled from the API and imported
  into the watch wallet. In bitcoin this is done with `importaddress`, with
  Amoveo we have a database local to the watcher that tracks watched addresses
  for reference when manually scanning the blockchain.
* discover deposits - We discover new deposit transactions and load them into
  our system as pending. For Bitcoin this uses `listsinceblock`, in Amoveo we
  scan the blockchain manually, and store checkpoints. In Snowblossom we use the
  equivalent of `listunspent` in Bitcoin.
* confirm deposits - We grab all pending deposits and lookup their transaction
  ID to discover if they are safe to credit to our users. Bitcoin's
  implementation uses `gettransaction`
* create withdrdraw transaction - We generate a raw transaction. Bitcoin's
  implementation uses `createrawtransaction`.
* send withdraw transaction - we broadcast the signed transaction to the network.
  Bitcoin uses `sendrawtransaction`
* confirm withdraw transaction - we verify that the transaction was mined into a
  block. Similar implementation to "confirm deposit"

### Signer

* generate address - A new subkey is generated and pushed to the exchange
  address pool in status 'generated'
* sign withdraw transaction - we sign the raw transaction without connection to
  a networked coinserver, and preferrably no coinserver at all. Consider this
  step as being completely offline. For Ethereum or other account based
  cryptocurrencies we serialize the transaction for signing in the "create"
  step, and the signer simply needs to sign the serialized payload, without ever
  connecting with other software. In some cases a simple "signing utility" such
  as `bitcoin-tx` is acceptable if the signing process is more complex.


## Code Quality and Documentation

As a general rule we like to see the following in currencies we're considering:

* Clear, reproducible, up to date build instructions for most major platforms (Linux, OSX, Windows). We will be deploying on Linux.
* Consistent, established, and preferably codified coding style/conventions across the code base. Logical variable naming, easy to read flow, good commenting to explain complex parts.
* A well documented API
* Examples of multi-language integration, or use of standard protocols for easy (non-library) integration. Using Protobuf instead of custom wire protocol. JSON over HTTP is always good.
