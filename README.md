# noble-ed25519 ![Node CI](https://github.com/paulmillr/noble-ed25519/workflows/Node%20CI/badge.svg) [![code style: prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square)](https://github.com/prettier/prettier)

Fastest JS implementation of [ed25519](https://en.wikipedia.org/wiki/EdDSA), an elliptic curve that could be used for asymmetric encryption and EDDSA signature scheme. Algorithmically resistant to timing attacks, conforms to [RFC8032](https://tools.ietf.org/html/rfc8032).

Includes [ristretto255](https://ristretto.group) support. Ristretto is a technique for constructing prime order elliptic curve groups with non-malleable encodings.

### This library belongs to *noble* crypto

> **noble-crypto** — high-security, easily auditable set of contained cryptographic libraries and tools.

- No dependencies, one small file
- Easily auditable TypeScript/JS code
- Uses es2020 bigint. Supported in Chrome, Firefox, Safari, node 10+
- All releases are signed and trusted
- Check out all libraries:
  [secp256k1](https://github.com/paulmillr/noble-secp256k1),
  [ed25519](https://github.com/paulmillr/noble-ed25519),
  [bls12-381](https://github.com/paulmillr/noble-bls12-381),
  [ripemd160](https://github.com/paulmillr/noble-ripemd160)

## Usage

Node:

> npm install noble-ed25519

```js
import * as ed from 'noble-ed25519';

const privateKey = ed.utils.randomPrivateKey(); // 32-byte Uint8Array or string.
const msgHash = 'deadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef';
(async () => {
  const publicKey = await ed.getPublicKey(privateKey);
  const signature = await ed.sign(msgHash, privateKey);
  const isSigned = await ed.verify(signature, msgHash, publicKey);
})();
```

Deno:

```typescript
import * as ed from 'https://deno.land/x/ed25519/mod.ts';
const privateKey = ed.utils.randomPrivateKey();
const publicKey = await ed.getPublicKey(privateKey);
```

## API

- [`getPublicKey(privateKey)`](#getpublickeyprivatekey)
- [`sign(hash, privateKey)`](#signhash-privatekey)
- [`verify(signature, hash, publicKey)`](#verifysignature-hash-publickey)
- [Helpers & Point](#helpers--point)

##### `getPublicKey(privateKey)`
```typescript
function getPublicKey(privateKey: Uint8Array): Promise<Uint8Array>;
function getPublicKey(privateKey: string): Promise<string>;
function getPublicKey(privateKey: bigint): Promise<Uint8Array>;
```
- `privateKey: Uint8Array | string | bigint` will be used to generate public key.
  Public key is generated by executing scalar multiplication of a base Point(x, y) by a fixed
  integer. The result is another `Point(x, y)` which we will by default encode to hex Uint8Array.
- Returns:
    * `Promise<Uint8Array>` if `Uint8Array` was passed
    * `Promise<string>` if hex `string` was passed
    * Uses **promises**, because ed25519 uses SHA internally; and we're using built-in browser `window.crypto`, which returns `Promise`.

##### `sign(hash, privateKey)`
```typescript
function sign(hash: Uint8Array, privateKey: Uint8Array): Promise<Uint8Array>;
function sign(hash: string, privateKey: string): Promise<string>;
```
- `hash: Uint8Array | string` - message hash which would be signed
- `privateKey: Uint8Array | string` - private key which will sign the hash
- Returns EdDSA signature. You can consume it with `SignResult.fromHex()` method:
    - `SignResult.fromHex(ed25519.sign(hash, privateKey))`

##### `verify(signature, hash, publicKey)`
```typescript
function verify(
  signature: Uint8Array | string | SignResult,
  hash: Uint8Array | string,
  publicKey: Uint8Array | string | Point
): Promise<boolean>
```
- `signature: Uint8Array | string | SignResult` - returned by the `sign` function
- `hash: Uint8Array | string` - message hash that needs to be verified
- `publicKey: Uint8Array | string | Point` - e.g. that was generated from `privateKey` by `getPublicKey`
- Returns `Promise<boolean>`: `Promise<true>` if `signature == hash`; otherwise `Promise<false>`

##### Ristretto255

To use Ristretto, simply use `fromRistrettoHash()` and `toRistrettoBytes()` methods.

```typescript
// The hash-to-group operation applies Elligator twice and adds the results.
ExtendedPoint.fromRistrettoHash(hash: Uint8Array): ExtendedPoint;

// Decode a byte-string s_bytes representing a compressed Ristretto point into extended coordinates.
ExtendedPoint.fromRistrettoBytes(bytes: Uint8Array)

// Encode a Ristretto point represented by the point (X:Y:Z:T) in extended coordinates to Uint8Array.
ExtendedPoint.toRistrettoBytes(): Uint8Array
```

It extends Mike Hamburg's Decaf approach to cofactor elimination to support cofactor-8 curves such as Curve25519.

In particular, this allows an existing Curve25519 library to implement a prime-order group with only a thin abstraction layer, and makes it possible for systems using Ed25519 signatures to be safely extended with zero-knowledge protocols, with no additional cryptographic assumptions and minimal code changes.

##### Helpers & Point

`utils.randomPrivateKey()`

Returns cryptographically random `Uint8Array` that could be used as Private Key.

`utils.precompute(W = 8, point = Point.BASE)`

Returns cached point which you can use to `#multiply` by it.

This is done by default, no need to run it unless you want to
disable precomputation or change window size.

We're doing scalar multiplication (used in getPublicKey etc) with
precomputed BASE_POINT values.

This slows down first getPublicKey() by milliseconds (see Speed section),
but allows to speed-up subsequent getPublicKey() calls up to 20x.

You may want to precompute values for your own point.

`utils.TORSION_SUBGROUP`

The 8-torsion subgroup ℰ8. Those are "buggy" points, if you multiply them by 8, you'll receive Point.ZERO.

Useful to check implementations for signature malleability. See [the link](https://moderncrypto.org/mail-archive/curves/2017/000866.html)

`Point#toX25519`

You can use the method to use ed25519 keys for curve25519 encryption.

https://blog.filippo.io/using-ed25519-keys-for-encryption

```typescript
ed25519.CURVE.P // 2 ** 255 - 19
ed25519.CURVE.n // 2 ** 252 - 27742317777372353535851937790883648493
ed25519.Point.BASE // new ed25519.Point(Gx, Gy) where
// Gx = 15112221349535400772501151409588531511454012693041857206046113283949847762202n
// Gy = 46316835694926478169428394003475163141307993866256225615783033603165251855960n;


// Elliptic curve point in Affine (x, y) coordinates.
ed25519.Point {
  constructor(x: bigint, y: bigint);
  static fromY(y: bigint);
  static fromHex(hash: string);
  toX25519(): bigint; // Converts to Curve25519
  toRawBytes(): Uint8Array;
  toHex(): string; // Compact representation of a Point
  equals(other: Point): boolean;
  negate(): Point;
  add(other: Point): Point;
  subtract(other: Point): Point;
  multiply(scalar: bigint): Point;
}
ed25519.SignResult {
  constructor(r: bigint, s: bigint);
  toHex(): string;
}

// Precomputation helper
utils.precompute(W, point);
```

## Security

Noble is production-ready & secure. Our goal is to have it audited by a good security expert.

We're using built-in JS `BigInt`, which is "unsuitable for use in cryptography" as [per official spec](https://github.com/tc39/proposal-bigint#cryptography). This means that the lib is potentially vulnerable to [timing attacks](https://en.wikipedia.org/wiki/Timing_attack). But:

1. JIT-compiler and Garbage Collector make "constant time" extremely hard to achieve in a scripting language.
2. Which means *any other JS library doesn't use constant-time bigints*. Including bn.js or anything else. Even statically typed Rust, a language without GC, [makes it harder to achieve constant-time](https://www.chosenplaintext.ca/open-source/rust-timing-shield/security) for some cases.
3. If your goal is absolute security, don't use any JS lib — including bindings to native ones. Use low-level libraries & languages.
4. We however consider infrastructure attacks like rogue NPM modules very important; that's why it's crucial to minimize the amount of 3rd-party dependencies & native bindings. If your app uses 500 dependencies, any dep could get hacked and you'll be downloading rootkits with every `npm install`. Our goal is to minimize this attack vector.

## Speed

Benchmarks done with Apple M1.

```
getPublicKey(utils.randomPrivateKey()) x 6,562 ops/sec @ 152μs/op
sign x 3,017 ops/sec @ 331μs/op
verify x 599 ops/sec @ 1ms/op
verifyBatch x 825 ops/sec @ 1ms/op
ristretto255#fromHash x 4,966 ops/sec @ 201μs/op
ristretto255 round x 2,251 ops/sec @ 444μs/op
```

Compare to alternative implementations:

```
# tweetnacl@1.0.3 (fast)
getPublicKey x 920 ops/sec @ 1ms/op # aka scalarMultBase
sign x 519 ops/sec @ 2ms/op
# ristretto255@0.1.1
getPublicKey x 877 ops/sec @ 1ms/op # aka scalarMultBase
```

## Contributing

1. Clone the repository.
2. `npm install` to install build dependencies like TypeScript
3. `npm run compile` to compile TypeScript code
4. `npm run test` to run jest on `test/index.ts`

## License

MIT (c) Paul Miller [(https://paulmillr.com)](https://paulmillr.com), see LICENSE file.
