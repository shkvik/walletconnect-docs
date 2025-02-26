# Wallet Usage

### Configure Networking and Pair clients

Confirm you have configured the Network Client first.

- [Networking](../core/networking-configuration.md)

### Subscribe for Web3Wallet publishers

When your `Web3Wallet` instance receives requests from a peer it will publish a related event. Set a subscription to handle them.

To track sessions subscribe to `sessionsPublisher` publisher

```swift
Web3Wallet.instance.sessionsPublisher
    .receive(on: DispatchQueue.main)
    .sink { [unowned self] (sessions: [Session]) in
        // reload UI
    }.store(in: &publishers)
```

The following publishers are available to subscribe:

```swift
public var sessionProposalPublisher: AnyPublisher<Session.Proposal, Never>
public var sessionRequestPublisher: AnyPublisher<Request, Never>
public var authRequestPublisher: AnyPublisher<Request, Never>
public var sessionPublisher: AnyPublisher<[Session], Never>
public var socketConnectionStatusPublisher: AnyPublisher<SocketConnectionStatus, Never>
public var sessionSettlePublisher: AnyPublisher<Session, Never>
public var sessionDeletePublisher: AnyPublisher<(String, Reason), Never>
public var sessionResponsePublisher: AnyPublisher<Response, Never>
```

### Connect Clients

Your Wallet should allow users to scan a QR code generated by dapps. You are responsible for implementing it on your own.
For testing, you can use our test dapp at: https://react-app.walletconnect.com/, which is v2 protocol compliant.
Once you derive a URI from the QR code call `pair` method:

```swift
try await Web3Wallet.instance.pair(uri: uri)
```

if everything goes well, you should handle following event:

```swift
Web3Wallet.instance.sessionProposalPublisher
    .receive(on: DispatchQueue.main)
    .sink { [weak self] sessionProposal in
           // present proposal to the user
    }.store(in: &publishers)
```

Session proposal is a handshake sent by a dapp and it's purpose is to define a session rules. Handshake procedure is defined by [CAIP-25](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-25.md).
`Session.Proposal` object conveys set of required `ProposalNamespaces` that contains required blockchains methods and events. Dapp requests with methods and wallet will emit events defined in namespaces.

The user will either approve the session proposal (with session namespaces) or reject it. Session namespaces must at least contain requested methods, events and accounts associated with proposed blockchains.

Accounts must be provided according to [CAIP10](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-10.md) specification and be prefixed with a chain identifier. chain_id + : + account_address. You can find more on blockchain identifiers in [CAIP2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md). Our `Account` type meets the criteria.

```
let account = Account("eip155:1:0xab16a96d359ec26a11e2c2b3d8f8b8942d5bfcdb")!
```

Accounts sent in session approval must at least match all requested blockchains.

Example proposal namespaces request:

```json
{
  "eip155": {
    "chains": ["eip155:137", "eip155:1"],
    "methods": ["eth_sign"],
    "events": ["accountsChanged"]
  },
  "cosmos": {
    "chains": ["cosmos:cosmoshub-4"],
    "methods": ["cosmos_signDirect"],
    "events": ["someCosmosEvent"]
  }
}
```

Example session namespaces response:

```json
{
  "eip155": {
    "accounts": [
      "eip155:137:0xab16a96d359ec26a11e2c2b3d8f8b8942d5bfcdb",
      "eip155:1:0xab16a96d359ec26a11e2c2b3d8f8b8942d5bfcdb"
    ],
    "methods": ["eth_sign"],
    "events": ["accountsChanged"]
  },
  "cosmos": {
    "accounts": [
      "cosmos:cosmoshub-4:cosmos1t2uflqwqe0fsj0shcfkrvpukewcw40yjj6hdc0"
    ],
    "methods": ["cosmos_signDirect", "personal_sign"],
    "events": ["someCosmosEvent", "proofFinalized"]
  }
}
```

##### Approve Session

```swift
 Web3Wallet.instance.approve(proposalId: "proposal_id", namespaces: [String: SessionNamespace])
```

When session is successfully approved `sessionsPublishers` will publish a `Session`

```swift
Web3Wallet.instance.sessionsPublishers
    .receive(on: DispatchQueue.main)
    .sink { [weak self] _ in
        self?.reloadSessions()
    }.store(in: &publishers)
}
```

`Session` object represents an active session connection with a dapp. It contains dapp’s metadata (that you may want to use for displaying an active session to the user), namespaces, and expiry date. There is also a topic property that you will use for linking requests with related sessions.

You can always query settled sessions from the client later with:

```swift
Web3Wallet.instance.getSessions()
```

### Handle requests from dapp

After the session is established, a dapp will request your wallet's users to sign a transaction or a message. Requests will be delivered by the following publisher.

```swift
Web3Wallet.instance.sessionRequestPublisher
    .receive(on: DispatchQueue.main)
    .sink { [weak self] sessionRequest in
        self?.showSessionRequest(sessionRequest)
    }.store(in: &publishers)
```

When a wallet receives a session request, you probably want to show it to the user. It’s method will be in scope of session namespaces. And it’s params are represented by `AnyCodable` type. An expected object can be derived as follows:

```swift
if sessionRequest.method == "personal_sign" {
    let params = try! sessionRequest.params.get([String].self)
} else if method == "eth_signTypedData" {
    let params = try! sessionRequest.params.get([String].self)
} else if method == "eth_sendTransaction" {
    let params = try! sessionRequest.params.get([EthereumTransaction].self)
}
```

Now, your wallet (as it owns your user’s private keys) is responsible for signing the transaction. After doing it, you can send a response to a dapp.

```swift
let response: AnyCodable = sign(request: sessionRequest) // implement your signing method
try await Web3Wallet.instance.respond(topic: request.topic, requestId: request.id, response: .response(response))
```

### Update Session

If you want to update user session's chains, accounts, methods or events you can use session update method.

```swift
try await Web3Wallet.instance.update(topic: session.topic, namespaces: newNamespaces)
```

### Extend Session

By default, session lifetime is set for 7 days and after that time user's session will expire. But if you consider that a session should be extended you can call:

```swift
try await Web3Wallet.instance.extend(topic: session.topic)
```

above method will extend a user's session to a week.

### Disconnect Session

For good user experience your wallet should allow users to disconnect unwanted sessions. In order to terminate a session use `disconnect` method.

```swift
try await Web3Wallet.instance.disconnect(topic: session.topic)
```

### Authorization Request Approval

Authorization request will be published by `authRequestPublisher`. When a wallet receives a request, you want to present it to the user and request a signature. After the user signs the authentication message, the wallet should respond to a dapp.

`Type` parameter represent signature validation method which will be used on dapp side. Supported signature validation methods: [EIP191](https://eips.ethereum.org/EIPS/eip-191), [EIP1271](https://eips.ethereum.org/EIPS/eip-1271). In both cases message will be signed with [EIP191](https://eips.ethereum.org/EIPS/eip-191) standard.

```swift
let signer = MessageSignerFactory.create()
let signature = try signer.sign(message: request.message, privateKey: privateKey, type: .eip191)
try await Web3WalletClient.respond(requestId: request.id, signature: signature, from: account)
```

In case user rejects an authentication request, call:

```swift
try await Web3WalletClient.reject(requestId: request.id)
```

### Get pending requests

if you've missed some requests you can query them with:
```swift 
Web3WalletClient.getPendingRequests()
```

### Sample App

To check more in details go and visit our Web3Wallet implementation app here.
