# Decentralized ADNL API

Clients connect directly to lite servers (nodes) using a binary protocol.

The client downloads keyblocks, the current state of the account and their **Merkle proofs**, which guarantees validity of the received data.

Read operations (like get-method calls) are made by launching a local TVM with a downloaded and verified state.

There is no need to download the full state of blockchain, the client only downloads what is needed for the operation. Calling local TVM is also lightweight.

You can connect to public lite servers from the global config ([mainnet](https://ton.org/global-config.json) or [testnet](https://ton.org/testnet-global.config.json)) or run your own lite server.

Since it checks Merkle proofs, you can even use untrusted lite servers.

Read more about Merkle proofs at [ADNL Protocol article](/learn/overviews/ADNL) or [TON Whitepaper](https://ton.org/ton.pdf) 2.3.10, 2.3.11.

## Pros & Cons

👍 - Ultra secure API with Merkle proofs.  
👎 - Need more time to figure it out. Not compatible with web frontends (non-HTTP protocol).

## API reference

Requests and responses to the server are described by a TL schema that allows you to generate a typed interface for a specific programming language.

[TonLib TL Schema](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/tonlib_api.tl)

### SDK:

- [C++ TonLib](https://github.com/ton-blockchain/ton/tree/master/example/cpp)
- [Python TonLib wrapper](https://github.com/toncenter/pytonlib)
- [Golang TonLib wrapper](https://github.com/ton-blockchain/tonlib-go)
- [Java TonLib wrapper (JNI)](https://github.com/ton-blockchain/tonlib-java)

### Usage example:

- [Desktop standard wallet](https://github.com/newton-blockchain/wallet-desktop) (C++ and Qt)
- [Android standard wallet](https://github.com/trm-dev/wallet-android) (Java)
- [iOS standard wallet](https://github.com/trm-dev/wallet-ios) (Swift)
- [TonLib CLI](https://github.com/ton-blockchain/ton/blob/master/tonlib/tonlib/tonlib-cli.cpp) (C++)
