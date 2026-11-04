# OpenDHT: a Distributed Hash Table

OpenDHT is a Kademlia-based distributed hash table for storing and retrieving
small values over a peer-to-peer network without a central server. Common uses
include peer discovery, service announcements, and decentralized applications.

The `opendht` npm package provides Node.js bindings to
[OpenDHT](https://opendht.net), with promises, async iterators, and TypeScript
declarations. It supports value subscriptions, signed and encrypted values, and
direct or HTTP(S) proxy connections.

## Get Started

Install with Node.js 22.9 or later:

```sh
npm install opendht
```

The package uses ES modules. Packaged binaries include OpenDHT and its non-system
native dependencies; no compiler or separate OpenDHT installation is needed on
supported platforms:

| Platform | Binary baseline |
|----------|-----------------|
| Linux x64 | Ubuntu 24.04 / glibc 2.39 |
| macOS ARM64 and x64 | macOS 15 |
| Windows x64 | Windows Server 2022 / MSVC |

Alpine/musl and other architectures are not currently bundled. Binaries target
Node-API version 8, but still require a compatible OS and libc. For source builds
or custom native options, see the
[build and test guide](https://github.com/savoirfairelinux/opendht/blob/master/nodejs/BUILD.md).

`DhtRunner` manages a node, `hash()` turns an application key into a DHT key, and
values carry a binary payload and metadata. `put()` publishes values, `get()`
retrieves them, and `listen()` subscribes to updates and expiration events.

Save this example as `hello.mjs` and run it with `node hello.mjs`. It starts a
node on an OS-assigned port, bootstraps into the public network, publishes a
value, and reads it back.

```js
import { DhtRunner, hash } from 'opendht';

async function main() {
    const runner = new DhtRunner();
    try {
        runner.run({ port: 0 });
        runner.bootstrap('bootstrap.jami.net', 4222);
        const key = hash('my-app/hello');
        const stored = await runner.put(key, Buffer.from('Hello from Node.js!'));
        if (!stored) throw new Error('DHT put failed');
        for await (const value of runner.get(key)) {
            console.log(value.id, value.data.toString());
        }
    } finally {
        await runner.close();
    }
}

main().catch(error => {
    console.error(error);
    process.exitCode = 1;
});
```

A bootstrap server supplies initial peers. Applications can use public bootstrap
servers or known peers on a private network. Values expire, and their availability
depends on participating nodes; the DHT does not provide durable storage guarantees.

## Working With Values

- `hash(string | Uint8Array)` returns a SHA-1 hash as a 40-character hex string.
  All get/put/listen keys must be hex hashes, not unhashed application keys.
- `version` is the underlying OpenDHT version.
- `run({port, ipv4, ipv6, network, persistPath, identity})` starts a threaded node. Port 0
  selects a free port. The defaults bind both IP families on all interfaces.
  `persistPath` is optional; use an empty string to disable persistence.
  `proxyServer` and `proxyUserAgent` configure proxy mode (see below).
- `getBoundPort(4 | 6)` defaults to IPv4; `getNodeId()` returns the node's hex ID.
- `getStatus()` returns the latest JS-dispatched `{ipv4, ipv6}` connectivity
  snapshot. Each family is `'disconnected'`, `'connecting'`, or `'connected'`.
  `runner.on('statusChanged', status => {})` receives changes on the JS thread.
  New and closing/closed runners report both families as disconnected. No status
  events are delivered after close begins; close does not emit a final event.
- `bootstrap(host, port)` queues bootstrap discovery. Returning does not mean the
  peer is connected; subsequent DHT operations report their own completion.
- `put(key, Uint8Array, {id, type, userType, permanent})` copies the input and
  resolves to the native success boolean. IDs are unsigned 64-bit `bigint` values.
- `get(key)` returns an async iterator of values as they arrive. The iterator
  ends when the lookup completes, or rejects with `ERR_DHT_GET_FAILED` on failure.
  `getAll(key)` returns a single promise resolving to the complete value array.
  Values contain `id`, `type`, `seq`, `userType`, copied `Buffer` data, `isSigned`,
  `isEncrypted`, and an `owner` long fingerprint (or null for unsigned values).
- `listen(key, (value, expired) => {})` returns a listener with idempotent
  `cancel()`. Queued deliveries are suppressed after cancellation. Callback
  return values are ignored; callbacks may cancel or close the runner.
- `listen(key)` without a callback returns an async iterator of `{value, expired}`
  events. It continues until cancelled or the runner closes.
- `close()` is idempotent and returns the same promise on repeated calls. It
  waits off the JS thread for graceful shutdown, then joins the native threads.
  After two seconds it forces the join if graceful shutdown has not completed.
  Pending operations reject with `ERR_DHT_CLOSED`. A closed runner cannot restart.
- `ref()` and `unref()` control whether the runner keeps Node's event loop alive.
  Running nodes are referenced by default. `Symbol.asyncDispose` calls `close()`.

Callbacks run only on their owning JS thread. There is no polling timer or worker
thread waiting on each get/put. Explicitly close runners; GC cleanup and Node
environment cleanup hooks are fallbacks, including forced worker termination.
Pending operations, active listeners, and registered status handlers keep their
runner alive until completion, cancellation, handler removal, or explicit close.
Graceful persistence cannot be guaranteed if the process is killed.

The binding does not cap the callback queue, accumulated `getAll()` results, or
iterator buffers. Slow iterator consumers can accumulate values without
backpressure. Queued callbacks retain their native values until delivery or cleanup.

## Subscriptions and Async Iteration

`get()` and callback-free `listen()` start immediately and return iterators with
`next()`, `return()`, `throw()`, `cancel()`, and `Symbol.asyncDispose`.
Leaving a `for await` loop with `break` or an exception cancels the subscription.
Explicit cancellation discards buffered values and settles pending reads as done.
Closing the runner rejects active get iterators with `ERR_DHT_CLOSED`; listener
iterators end normally. Successful get completion preserves unread buffered values.

```js
for await (const value of runner.get(hash('example'))) {
  console.log(value.id, value.data);
}
const snapshots = await runner.getAll(hash('example'));

for await (const { value, expired } of runner.listen(hash('updates'))) {
  console.log(value.id, expired);
  if (expired) break;
}
```

## Using a Proxy

Pass the URL of an OpenDHT HTTP(S) proxy at startup:

```js
runner.run({
  proxyServer: 'https://proxy.example.com',
  proxyUserAgent: 'my-app/1.0',
});
```

This selects the native OpenDHT proxy client, not an HTTP CONNECT or SOCKS proxy.
The node does not bind UDP sockets in this mode; `port`, `ipv4`, `ipv6`, and
`bootstrap()` are for direct DHT nodes. Omit `proxyServer` or use an empty string
for direct networking. Proxy settings are startup-only; create a new runner to
change transports or servers. Packaged binaries include proxy-client support.

## Signed and Encrypted Values

`PrivateKey.generate(options)` and `Identity.generate(name, options)` return
promises and generate keys off the JS thread. Options accept `algorithm: 'ec'`
(the default, using OpenDHT's default EC curve) or `'rsa'`. RSA `bits` must be an
integer between 2048 and 8192; the default is 2048.

- `new PrivateKey(bytes, password?)` imports PEM/DER; `serialize(password?)`
  exports PKCS#8 PEM. A nonempty password encrypts the exported key.
- `privateKey.getPublicKey()` returns an owned public key. `sign(bytes)` and
  `decrypt(bytes)` return copied Buffers.
- `new PublicKey(bytes)` imports a public key; `serialize()` exports DER.
  `checkSignature(data, signature)` returns a boolean. `encrypt(bytes)` requires
  an RSA key; EC keys support signatures, not this encryption scheme.
- `new Certificate(bytes)` imports PEM/DER or a PEM certificate chain;
  `serialize()` exports a PEM chain. Import does not establish trust or validate
  the chain against a trust store. `getPublicKey()` returns its public key.
- Public keys and certificates expose `getId()` (40 hex characters) and
  `getLongId()` (64 hex characters).
- `new Identity(privateKey, certificate)` requires matching fingerprints.
  Its read-only `privateKey` and `certificate` properties return owned wrappers.
- Pass `identity` to `run()` to enable `putSigned(key, data, options?)` and
  `putEncrypted(key, data, recipientPublicKey, options?)`. Both return the same
  success boolean as `put()`. Encrypted values are signed by the sender and
  transparently decrypted for the recipient by OpenDHT's get/listen layer.

```js
import { Identity, DhtRunner } from 'opendht';

async function createAuthenticatedNode() {
    const identity = await Identity.generate('example', { algorithm: 'rsa', bits: 2048 });
    const runner = new DhtRunner();
    runner.run({ port: 0, identity });
    return { runner, identity };
}
```

Crypto methods take Uint8Array/Buffer inputs, not implicit text encodings. Imports
must be nonempty. Malformed handles/data,
mismatched identities, and unsupported encryption keys raise JS exceptions.
Only generation is asynchronous; import/export/sign/encrypt/decrypt are
synchronous and should be used from worker threads for large workloads.

Query filters, AbortSignal support, certificate trust-store validation, CA
issuance controls, and runtime proxy switching remain future work.

## Learn More

- [TypeScript API declarations](https://github.com/savoirfairelinux/opendht/blob/master/nodejs/index.d.ts)
- [Interactive node example](https://github.com/savoirfairelinux/opendht/blob/master/nodejs/examples/dhtnode.ts)
- [Build, test, and packaging guide](https://github.com/savoirfairelinux/opendht/blob/master/nodejs/BUILD.md)
- [OpenDHT project and issue tracker](https://github.com/savoirfairelinux/opendht)