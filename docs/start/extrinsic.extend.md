# Extending extrinsics

On some chains, the need may arise to customize the extrinsic format. In this section we will explain what extrinsics and their payloads contain, explain how signed extensions work and provide a walk through of an advanced use-case where extrinsics are customized for a specific Substrate chain.

## Extensions

In Substrate (which forms the base of Polkadot and a number of custom chains), an extrinsic has a specific version, flag to indicating if it has been signed, the address, signature, extra data as well as the actual method with params. In addition, the signature is generated over the call and may include values that are not encoded into the final extrinsic.

For instance, with version 3 extrinsics, the signature payload contains the runtime spec version as well as the `genesisHash` and `blockHash` (the latter being equivalent to the `genesisHash` in case of immortal transactions), however while these 3 fields are signed together with the method data, they do not actually appear in the extrinsic itself. Rather the chain executing the transaction will retrieve this information, add it to the method data and compare the signatures thereof.

So in both the cases of the `genesisHash` and `specVersion`, if these do not match the signer version, the transaction won't be accepted - however the extrinsic doesn't explicitly carry this information in the data being transmitted, rather only implicitly as part of the signature. And it always forms part of the validation.

## Default extrinsics

With the above in-mind, the extrinsic format is explicitly defined as having the following structure in Substrate -

```rust
pub struct UncheckedExtrinsic<Address, Call, Signature, Extra>
where Extra: SignedExtension
{
	pub signature: Option<(Address, Signature, Extra)>,
	pub function: Call,
}
```

The `Option` is here encoded such that is conveys both the extrinsic version `0x03` for Substrate 2.x chains (`0x01` for Substrate 1.x chains) as well as a high-bit that indicates if the transaction is signed or unsigned. (For unsigned the `signature` details therefore does not appear). The `SignedExtension` part defined both the data in the actual extrinsic as well as data, i.e. `AdditionalSigned`, that appears in the payload for the signature, but is not explicitly contained in the extrinsic.

The default `SignedExtension` for Substrate 2.x with extrinsic version 3 is defined as follow -

```rust
pub type SignedExtra = (
	system::CheckVersion<Runtime>,
	system::CheckGenesis<Runtime>,
	system::CheckEra<Runtime>,
	system::CheckNonce<Runtime>,
	system::CheckWeight<Runtime>,
	balances::TakeFees<Runtime>,
	contracts::CheckBlockGasLimit<Runtime>,
);
```

Some of these are only checks, requiring no additional data in the payload or extrinsic itself, i.e. the contracts checks do exactly that. It only applies checks and invalidates when those checks are invalid. This is a powerful concept, for instance on the [initial Kusama chain this was used to limit the types of calls allowed](https://github.com/paritytech/polkadot/blob/f52c714ec3411eea58647d0f5176f4eb81660188/runtime/src/lib.rs#L117-L139).

## Extension deep-dive

For each of the default extensions, we will take a look through to understand the implications of the specific extension and how it relates to both the data contained in the extrinsic as well as the signature payload.

- `system::CheckVersion` - This checks that the spec version matches between the extrinsic and the chain. It takes no parameters which means that these is no explicit data in the extrinsic format for this field, however it has `type AdditionalSigned = u32` which means that a `u32` containing the runtime spec version is part of the signature payload.

- `system::CheckGenesis` - This checks that the `genesisHash` matches between extrinsic and chain. Like the previous check, no additional data is added to the extrinsic, however with `type AdditionalSigned = T::Hash`, the `genesisHash` is part of the signature payload.

- `system::CheckEra` - This checks the era (mortal or immortal) for the transaction being sent. It checks both the `era: Era` as part of the actual extrinsic and the `blockHash` via the `type AdditionalSigned = T::Hash`. This means that the extrinsic era is both in the data being signed and the extrinsic itself, while the `blockHash` the era applies to is only available in the signature payload.

- `system::CheckNonce` - This checks the nonce for the sending account. Unlike the preceding checks, it has no payload-specific data, however the `nonce: Compact<T::Index>` (`Index` is default `u32`) is applicable to both the extrinsic and, therefore, the actual signature payload as well.

- `system::CheckWeight` - This checks the weight and length of the block and ensure that it does not exceed the limits. It does not have any specific data attached to either the extrinsic nor payload, but rather just does calculations based on the weights and type of transaction received.

- `balances::TakeFees` - Consumes fees proportional to the length and weight of the transaction. It operates on the `fee: Compact<T::Balance>`, which means this value is included in both the extrinsic and subsequent payload being signed.

- `contracts::CheckBlockGasLimit` - As explained briefly above, this extension does not add data to the extrinsic, or the signature payload, however it ensures that the transaction does not exceeds the block gas limit.

## Extrinsic and signature payloads

With the above extension, the following formats for the extrinsic and payloads are the outcome of the application of the signed extension. For the extrinsic the following data is [always encoded for v3 extrinsics](https://github.com/polkadot-js/api/blob/8b0ef159c05bcb5d9b664546d0e7289e79b5c9d5/packages/types/src/primitive/Extrinsic/v3/Extrinsic.ts#L27) -

```js
class ExtrinsicV3 extends Struct {
 constructor (value) {
  super({
   signer: 'Address',
   signature: 'Signature',
   era: 'ExtrinsicEra', // extra via system::CheckEra
   nonce: 'Compact<Index>', // extra via system::CheckNonce
   tip: 'Compact<Balance>', // extra via balances::TakeFees
   method: 'Call'
  }, value);
 }
 ...
```

The signature payload will contain the same information as the extrinsic, with the following [additional information](https://github.com/polkadot-js/api/blob/8b0ef159c05bcb5d9b664546d0e7289e79b5c9d5/packages/types/src/primitive/Extrinsic/v3/ExtrinsicPayload.ts#L33) as expected by the `AdditionalSigned` portions of the extensions -

```js
class ExtrinsicPayloadV3 extends Struct {
 constructor (value) {
  super({
   method: 'Bytes',
   era: 'ExtrinsicEra', // extra via system::CheckEra
   nonce: 'Compact<Index>', // extra via system::CheckNonce
   tip: 'Compact<Balance>', // extra via balances::TakeFees
   specVersion: 'u32', // additional via system::CheckVersion
   genesisHash: 'Hash', // additional via system::CheckGenesis
   blockHash: 'Hash' // additional via system::CheckEra
  }, value);
 }
 ...
```

As per the above structures, it means that both the extrinsic sent on-chain as well as the data being signed to generate the signature is tied by the hip based on the logic the chain expects via `SignedExtension`. The API is only aware of the version of the extrinsic being used on-chain (it determines this on connection) and therefore only knows about the specific logic that has been coded for the extrinsic version.

## Extending existing or implementing new

When the API encodes or decodes an extrinsic, it uses the first `Option` byte to determine the version. Once it has this value, it will create a specific extrinsic via `createType('ExtrinsicV3', value)`. This means that at any point, you can supply your own version of either the `Extrinsic` or `ExtrinsicPayload` and you can do so via 2 avenues -

- If you are extending/replacing the existing version, you can inject your own types for both `ExtrinsicV3` and `ExtrinsicV3Payload` (assuming you are replacing v3)

- If you are adding a new version, you can add a handler for both `ExtrinsicUnknown` and `ExtrinsicPayloadUnknown`. These will be constructed when the version the API is aware of does not match with the on-chain version.

While we will not provide a full example of all the code here, the above links will show the existing implementations. However, assuming we have a chain where neither the nonce or tip is applicable (or we just don't care) and we are ignoring the check to the runtime versioning.

Additionally assuming that we have made the required `SignedExtension` updates by removing `system::CheckVersion`, `system::CheckNonce` and `balances::TakeFees`, we can do the following -

```js
...
class OwnExtrinsic extends Struct {
 constructor (value) {
  super({
   signer: 'Address',
   signature: 'Signature',
   era: 'ExtrinsicEra', // extra via system::CheckEra
   method: 'Call'
  }, value);
 }
 ...
}

class OwnExtrinsicPayload extends Struct {
 constructor (value) {
  super({
   method: 'Bytes',
   era: 'ExtrinsicEra', // extra via system::CheckEra
   genesisHash: 'Hash', // additional via system::CheckGenesis
   blockHash: 'Hash' // additional via system::CheckEra
  }, value);
 }
 ...
 // signing logic needs to be included, as per existing
}
...

// inject our types at API construction
const api = ApiPromise.create({
 types: {
  'ExtrinsicV3': OwnExtrinsic,
  'ExtrinsicV3Payload': OwnExtrinsicPayload,
 }
})
```

The above example is certainly an advanced example, but it shows that all data types in the API can be adjusted and these adjustment can be provided to the API. In all cases, if you made updates to the formats and types of the actual runtime, you need to ensure that the API is aware of these changes.

In the above example, should these updates only be made on the node side, without the required API adjustments, the API will generate invalid transactions for the node since it is unaware of the changes and adjusted formats. Making the adjustments on only one side will mean that the signature verification can fail and that the format will not be decodable via the node.

(These extensions are not exposed via metadata at all, and would be quite difficult to do as well - since each of these have specific logic as well as data types assigned.)

## Using with TypeScript

The API is built with TypeScript (as are all projects in the [polkadot-js organization](https://github.com/polkadot-js/)) and as such allows developers using TS to have access to all the type interfaces defined on the chain, as well as having access to typings on interacting with the `api.*` namespaces. In the next section we will provide an overview of [what is available in terms of types and TypeScript](typescript.md).
