# Exercise 6

In exercise 6, I plan to do several things:

- Runtime benchmarking
- Use runtime benchmarking to provide the weights for extrinsic calls.
- Create a custom Chainspec and change to raw chainspec, and deploy a test net.

Also, I plan to use babe and some other related pallets to construct the POS chain step by step in the future.

## 1. Benchmarking

### 1.1 Build

```sh
cargo build --release --features runtime-benchmarks
```

### 1.2 Test

```sh
cargo test -p pallet-template --all-features
```

The results will be:
![Test Result Picture](https://github.com/IanGYan/Exercise6-Benchmarking-Weights-and-Chainspec/blob/main/doc/img/test_results.jpeg)

### 1.3 Create the Weights trait

This step is run to generate the pallet’s Weights trait, for use in the pallet’s #[pallet::weight] tags. It should also be re-run to incorporate changes to the complexities of functions or new dispatchables.

```sh
./target/release/node-template benchmark --chain dev --execution=wasm --wasm-execution=compiled --pallet pallet_template --extrinsic do_something --steps 50 --repeat 20 --output ./pallets/template/src/weights.rs --template ./.maintain/frame-weight-template.hbs
```

It will create or update the source file [weights.rs](./pallets/template/src/weights.rs) in the specific folder. the Weights trait will be:

```rust
#![allow(unused_parens)]
#![allow(unused_imports)]

use frame_support::{traits::Get, weights::{Weight, constants::RocksDbWeight}};
use sp_std::marker::PhantomData;

/// Weight functions needed for pallet_template.
pub trait WeightInfo {
    fn do_something(s: u32, ) -> Weight;
}

/// Weights for pallet_template using the Substrate node and recommended hardware.
pub struct SubstrateWeight<T>(PhantomData<T>);
impl<T: frame_system::Config> WeightInfo for SubstrateWeight<T> {
    fn do_something(s: u32, ) -> Weight {
        (22_320_000 as Weight)
            // Standard Error: 0
            .saturating_add((1_000 as Weight).saturating_mul(s as Weight))
            .saturating_add(T::DbWeight::get().writes(1 as Weight))
    }
}

// For backwards compatibility and tests
impl WeightInfo for () {
    fn do_something(s: u32, ) -> Weight {
        (22_320_000 as Weight)
            // Standard Error: 0
            .saturating_add((1_000 as Weight).saturating_mul(s as Weight))
            .saturating_add(RocksDbWeight::get().writes(1 as Weight))
    }
}
```

Or, just run the command below to review the benchmarking result as below:

```sh
./target/release/node-template benchmark --chain dev --execution=wasm --wasm-execution=compiled --pallet pallet_template --extrinsic do_something --steps 50 --repeat 20
```

The results will be:
![Benchmarking Picture 1](https://github.com/IanGYan/Exercise6-Benchmarking-Weights-and-Chainspec/blob/main/doc/img/benchmarking_1.jpeg)
![Benchmarking Picture 2](https://github.com/IanGYan/Exercise6-Benchmarking-Weights-and-Chainspec/blob/main/doc/img/benchmarking_2.jpeg)

## 2. The Weights

In the [pallet-template](./pallets/template/src/lib.rs), we use the benchmarking weights for the do_something extrinsic.

```rust
#![cfg_attr(not(feature = "std"), no_std)]
--- snip ---
#[cfg(feature = "runtime-benchmarks")]
mod benchmarking;
pub mod weights;

#[frame_support::pallet]
pub mod pallet {
    pub use crate::weights::WeightInfo;
// --- snip ---
    #[pallet::config]
    pub trait Config: frame_system::Config {
        type Event: From<Event<Self>> + IsType<<Self as frame_system::Config>::Event>;
        ype WeightInfo: WeightInfo;
    }
// --- snip ---
    #[pallet::call]
    impl<T: Config> Pallet<T> {
        #[pallet::weight(T::WeightInfo::do_something(*something))]
        pub fn do_something(origin: OriginFor<T>, something: u32) -> DispatchResult {
// --- snip ---
            Ok(())
        }
// --- snip ---
    }
}
```

## 3. Create custom chain specification

### 3.1 Create key pair with Subkey

Generate sr25519 for Aura:

```sh
# subkey command
subkey generate --scheme sr25519
```

```sh
# subkey output - it is not my results, just for instance.
Secret phrase `{The Mnemonic Phase}` is account:
  Secret seed:      0xa2b0200f9666b743402289ca4f7e79c9a4a52ce129365578521b0b75396bd242
  Public key (hex): 0x0a11c9bcc81f8bd314e80bc51cbfacf30eaeb57e863196a79cccdc8bf4750d21
  Account ID:       0x0a11c9bcc81f8bd314e80bc51cbfacf30eaeb57e863196a79cccdc8bf4750d21
  SS58 Address:     5CHucvTwrPg8L2tjneVoemApqXcUaEdUDsCEPyE7aDwrtR8D
```

Generate ed25519 for Grandpa:

```sh
# subkey command
subkey inspect --scheme ed25519 "{The Mnemonic Phase FROM PREVIOUS STEP}"
```

I prepared 2 key pairs for next steps.

### 3.2 Customize my own Chainspec

Firstly, export the chainspec to a file:

```sh
# Export the local chain spec to json
./target/release/node-template build-spec --disable-default-bootnode --chain local > my-staging-spec.json
```

Use the SS58 address generated with Subkey to modify the exported chainspec.

```json
{
 //-- snip --
 "genesis": {
    "runtime": {
      "frameSystem": {
        //-- snip --
      },
      "palletAura": {
        "authorities": [
            //change the 2 addresses below with generated sr25519 addresses.
            "5FfBQ3kwXrbdyoqLPvcXRp7ikWydXawpNs2Ceu3WwFdhZ8W4",
            "5EhrCtDaQRYjVbLi7BafbGpFqcMhjZJdu8eW8gy6VRXh6HDp"
        ]
      },
      "palletGrandpa": {
        "authorities": [
            //change the 2 addresses below with generated sr25519 addresses.
            ["5G9NWJ5P9uk7am24yCKeLZJqXWW6hjuMyRJDmw4ofqxG8Js2", 1],
            ["5CRZoFgJs4zLzCCAGoCUUs2MRmuD5BKAh17pWtb62LMoCi9h", 1]
        ]
      },
    //-- snip --
    }
 }
}
```

The final chain specification will be [my-staging-spec.json](./my-staging-spec.json).

Then, convert it to raw specification:

```sh
./target/release/node-template build-spec --chain=my-staging-spec.json --raw --disable-default-bootnode > my-staging-spec-Raw.json
```

The final raw chain specification will be [my-staging-spec-Raw.json](./my-staging-spec-Raw.json).

## 4. Launch staging chain with the custom spec

### 4.1 First Participant Starts a Bootnode

```sh
# purge chain (only required for new/modified dev chain spec)
./target/release/node-template purge-chain --base-path /tmp/node01 --chain local -y
```

```sh
# start node01
./target/release/node-template \
  --base-path /tmp/node01 \
  --chain ./my-staging-spec-Raw.json \
  --port 30333 \
  --ws-port 9945 \
  --rpc-port 9933 \
  --telemetry-url 'wss://telemetry.polkadot.io/submit/ 0' \
  --validator \
  --rpc-methods Unsafe \
  --name MyNode01
```

### 4.2 Add keys to Keystore

I use Polkdot-JS API UI to add the keys to the keystore.

### 4.3 Subsequent Participants Join

```sh
# purge chain (only required for new/modified dev chain spec)
./target/release/node-template purge-chain --base-path /tmp/node02 --chain local -y
```

```sh
# start node02
./target/release/node-template \
  --base-path /tmp/node02 \
  --chain ./my-staging-spec-Raw.json \
  --port 30334 \
  --ws-port 9946 \
  --rpc-port 9934 \
  --telemetry-url 'wss://telemetry.polkadot.io/submit/ 0' \
  --validator \
  --rpc-methods Unsafe \
  --name MyNode02 \
  --bootnodes /ip4/127.0.0.1/tcp/30333/p2p/12D3KooWS1Veq29c5xTFDgYS4GbdV9c7MVBP5nWzaA9rR8aLYapx
  # you MUST fill the correct info in the line above:
  # --bootnodes /ip4/<IP Address>/tcp/<p2p Port>/p2p/<Peer ID>
```

Also, I add the key to Keystore as the step 4.2.

Now, the chain works! The pictures of the chain are showed as below:

Node01 -

![node01](https://github.com/IanGYan/Exercise6-Benchmarking-Weights-and-Chainspec/blob/main/doc/img/node01.jpeg)

Node02 -

![node02](https://github.com/IanGYan/Exercise6-Benchmarking-Weights-and-Chainspec/blob/main/doc/img/node02.jpeg)

Polkdot-JS -

![polkdot-js](https://github.com/IanGYan/Exercise6-Benchmarking-Weights-and-Chainspec/blob/main/doc/img/polkdot-js.jpeg)

## 5. TODO: Change PoA to POS

TODO...
