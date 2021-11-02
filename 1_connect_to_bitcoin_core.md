# Breaking LDK sample down

- [0. Introduction](#1-create-bicoind-client)
- [1. Create bicoind client](#1-create-bicoind-client)
- [2. Implement `BlockSource` trait](#)
- [3. Implement `BitcoindClient` methods](#)
- [4. Get Fee Estimates](#)
- [5. Implement other RPC methods](#)
- [6. Implement `BroadcasterInterface` trait](#)

## 0. Introduction

This tutorial focuses on `src/bitcoind_client.rs` file from LDK sample, as this implements the first component needed to build an LDK node.

This component is `BitcoindClient`, whose main purpose is to communicate with bitcoind.

This is a suggestion, as there are other ways to implement the interface with the bitcoin node, as both LDK and BDK allow a good level of flexibility. The concepts presented here can be used in other approaches.

## 1. Create bicoind client

The following struct can be used to store the information to connect and interacts with bitcoind.

```rust
pub struct BitcoindClient {
	bitcoind_rpc_client: Arc<Mutex<RpcClient>>,
	host: String,
	port: u16,
	rpc_user: String,
	rpc_password: String,
	fees: Arc<HashMap<Target, AtomicU32>>,
	handle: tokio::runtime::Handle,
}
```

The `host`, `port`, `rpc_user` and `rpc_password` fields are self explanatory. These parameter will be provided by the user.

`bitcoind_rpc_client` is `RpcClient` is a simple RPC client for calling methods using HTTP `POST`. It is implemented in [`rust-lightning/lightning-block-sync/rpc.rs`](https://github.com/rust-bitcoin/rust-lightning/blob/61341df39e90de9d650851a624c0644f5c9dd055/lightning-block-sync/src/rpc.rs).

The purpose `RpcClient` is to creates a new RPC client connected to the given endpoint with the provided credentials. The credentials should be a base64 encoding of a user name and password joined by a colon, as is required for HTTP basic access authentication.

It implements [`BlockSource`](https://github.com/rust-bitcoin/rust-lightning/blob/61341df39e90de9d650851a624c0644f5c9dd055/lightning-block-sync/src/lib.rs#L55) against a Bitcoin Core RPC. It is an asynchronous interface for retrieving block headers and data.

This structure is wrapped in `Arc<Mutex<...>>`. It provides non-async methods for performing operations on the data within, and only lock the mutex inside these methods. This is provided by [Tokio stack](https://github.com/tokio-rs/tokio).


The `fees` field is a `HashMap` defined by a target and the value expected for this target (which is an AtomicU32 type, an integer type which can be safely shared between threads). The target, in this case, is a new `enum`, defined by three priority levels. Note that this `enum` derives some traits (such as `Clone`, `Hash` and `Eq`) via the `#[derive]` attribute.

This structure is wrapped in `std::sync::Arc` to allow safe sharing data between threads. `Arc` is a single-threaded reference-counting pointer that uses an atomic type for the reference count. Hence it can be used by multiple threads without the count getting out of sync.

```rust
#[derive(Clone, Eq, Hash, PartialEq)]
pub enum Target {
	Background,
	Normal,
	HighPriority,
}
```

The `handle` field is a `tokio::runtime::Handle` instance and is used to spawn an asynchronous computation (future) onto the runtime's executor, usually a thread pool. The thread pool is then responsible for polling the future until it completes.

## 2. Implement `BlockSource` trait

`BlockSource`, as mentioned before, is a trait implemented in `lightning-block-sync` and extends the `Sync` and `Send` traits.

```rust
pub trait BlockSource : Sync + Send {
	fn get_header<'a>(&'a mut self, header_hash: &'a BlockHash, height_hint: Option<u32>) -> AsyncBlockSourceResult<'a, BlockHeaderData>;

	fn get_block<'a>(&'a mut self, header_hash: &'a BlockHash) -> AsyncBlockSourceResult<'a, Block>;

	fn get_best_block<'a>(&'a mut self) -> AsyncBlockSourceResult<(BlockHash, Option<u32>)>;
}
```

`Sync` allows an object to to be used by two threads at the _same_ time. This is trivial for non-mutable objects, but mutations need to be synchronized (performed in sequence with the same order being seen by all threads).

`Send` allows an object to be used by two threads A and B at _different_ times. Thread A can create and use an object, then send it to thread B, so thread B can use the object while thread A cannot.

`RpcClient` implements `BlockSource` and the `get_header`, `get_block` and `get_best_block` functions. They call the equivalent RPC methods.

```rust
impl BlockSource for RpcClient {
	fn get_header<'a>(&'a mut self, header_hash: &'a BlockHash, _height: Option<u32>) -> AsyncBlockSourceResult<'a, BlockHeaderData> {
		Box::pin(async move {
			let header_hash = serde_json::json!(header_hash.to_hex());
			Ok(self.call_method("getblockheader", &[header_hash]).await?)
		})
	}

	fn get_block<'a>(&'a mut self, header_hash: &'a BlockHash) -> AsyncBlockSourceResult<'a, Block> {
		Box::pin(async move {
			let header_hash = serde_json::json!(header_hash.to_hex());
			let verbosity = serde_json::json!(0);
			Ok(self.call_method("getblock", &[header_hash, verbosity]).await?)
		})
	}

	fn get_best_block<'a>(&'a mut self) -> AsyncBlockSourceResult<'a, (BlockHash, Option<u32>)> {
		Box::pin(async move {
			Ok(self.call_method("getblockchaininfo", &[]).await?)
		})
	}
}
```

So, for struct `BitcoindClient` to implement `BlockSource` trait, just call those same methods of `bitcoind_rpc_client`. It allows this struct be directly used in synchronization and validation functions.

```rust
impl BlockSource for &BitcoindClient {
	fn get_header<'a>(
		&'a mut self, header_hash: &'a BlockHash, height_hint: Option<u32>,
	) -> AsyncBlockSourceResult<'a, BlockHeaderData> {
		Box::pin(async move {
			let mut rpc = self.bitcoind_rpc_client.lock().await;
			rpc.get_header(header_hash, height_hint).await
		})
	}

	fn get_block<'a>(
		&'a mut self, header_hash: &'a BlockHash,
	) -> AsyncBlockSourceResult<'a, Block> {
		Box::pin(async move {
			let mut rpc = self.bitcoind_rpc_client.lock().await;
			rpc.get_block(header_hash).await
		})
	}

	fn get_best_block<'a>(&'a mut self) -> AsyncBlockSourceResult<(BlockHash, Option<u32>)> {
		Box::pin(async move {
			let mut rpc = self.bitcoind_rpc_client.lock().await;
			rpc.get_best_block().await
		})
	}
}

```

## 3. Implement `BitcoindClient` methods

The first step is to to declare minimum fee rate the node are allowed to send. This value is important because it will be used as fallback when calculating the most economical fee.

The minimum feerate allowed, as specify by LDK is 253. The reason for this value is that the LDK fee is expressed in sats/KW (kilo weight units) and this value correspond approximately to 1.0 sats/vB.

```
1,000 sat/kvB = 1 sat/vB
1 sat/vB = 0.25 sat/wu
0.25 sat/wu = 250 sat/kwu
```

The primary reason for the minimum feerate (and related things like the discard feerate) is preventing the use of the P2P network as a cheap global broadcast system. If it's easy to produce transaction that will never confirm yet will relay across the network, it'd open the network to abuse. And 1 sat/vbyte seems enough to avoid it.

```rust
const MIN_FEERATE: u32 = 253;

impl BitcoindClient {
	pub async fn new(
		host: String, port: u16, rpc_user: String, rpc_password: String,
		handle: tokio::runtime::Handle,
	) -> std::io::Result<Self> {
		let http_endpoint = HttpEndpoint::for_host(host.clone()).with_port(port);
		let rpc_credentials =
			base64::encode(format!("{}:{}", rpc_user.clone(), rpc_password.clone()));
		let mut bitcoind_rpc_client = RpcClient::new(&rpc_credentials, http_endpoint)?;
		let _dummy = bitcoind_rpc_client
			.call_method::<BlockchainInfo>("getblockchaininfo", &vec![])
			.await
			.map_err(|_| {
				std::io::Error::new(std::io::ErrorKind::PermissionDenied,
				"Failed to make initial call to bitcoind - please check your RPC user/password and access settings")
			})?;
		let mut fees: HashMap<Target, AtomicU32> = HashMap::new();
		fees.insert(Target::Background, AtomicU32::new(MIN_FEERATE));
		fees.insert(Target::Normal, AtomicU32::new(2000));
		fees.insert(Target::HighPriority, AtomicU32::new(5000));
		let client = Self {
			bitcoind_rpc_client: Arc::new(Mutex::new(bitcoind_rpc_client)),
			host,
			port,
			rpc_user,
			rpc_password,
			fees: Arc::new(fees),
			handle: handle.clone(),
		};
		BitcoindClient::poll_for_fee_estimates(
			client.fees.clone(),
			client.bitcoind_rpc_client.clone(),
			handle,
		);
		Ok(client)
	}
    // ...
}
```

The code above declares the `MIN_FEERATE` constant and implements a static `new` method to create a `BitcoindClient` object.

Firstly, an `HttpEndpoint` object is created to connect to Bitcoin Core daemon. It also uses the credentials passed by the user (`rpc_credentials`). Both are parameters to a new `RpcClient` object.

After a `getblockchaininfo` request is done. The purpose of this call is just to test the connection. Note that the response is converted to `BlockchainInfo`. This struct is declared `convert.rs` file and it implements the `TryInto<T>` trait for `for JsonResponse`. This pattern is used for other calls to bitcoind.

```rust
pub struct BlockchainInfo {
	pub latest_height: usize,
	pub latest_blockhash: BlockHash,
	pub chain: String,
}

impl TryInto<BlockchainInfo> for JsonResponse {
	type Error = std::io::Error;
	fn try_into(self) -> std::io::Result<BlockchainInfo> {
		Ok(BlockchainInfo {
			latest_height: self.0["blocks"].as_u64().unwrap() as usize,
			latest_blockhash: BlockHash::from_hex(self.0["bestblockhash"].as_str().unwrap())
				.unwrap(),
			chain: self.0["chain"].as_str().unwrap().to_string(),
		})
	}
}
```

`JsonResponse` is a simple struct declared in `rust-lightning/lightning-block-sync` and the `TryInto` implementation specifies the conversion logic form json response to BlockchainInfo object.

```rust
pub struct JsonResponse(pub serde_json::Value);
```

After the initial call to bitcoind, the default values for fees are set in `fees` hashmap. Then it creates the `client` object to return the new instance and finally, the method calls `BitcoindClient::poll_for_fee_estimates(...)` to get the fee estimates.

## 4. Get Fee Estimates

The `poll_for_fee_estimates()` function has 3 parameters: `fees`, the hashmap which the values will be stored, `rpc_client`, the client that is used to communicate with bitcoind and `handle` to spawn the asynchronous processing.

This function calls `estimatesmartfee` 3 times, with following parameters:

* `economical` mode and 144 blocks target. The result will be attributed to `background_estimate` (and `Target::Background` in the `fees` hashmap.)

* `economical` mode and 18 blocks target. The result will be attributed to `normal_estimate` (and `Target::Normal` in the `fees` hashmap.)

* `conservative` mode and 6 blocks target. The result will be attributed to `high_prio_estimate` (and `Target::HighPriority` in the `fees` hashmap.)

```rust
fn poll_for_fee_estimates(
		fees: Arc<HashMap<Target, AtomicU32>>, rpc_client: Arc<Mutex<RpcClient>>,
		handle: tokio::runtime::Handle,
	) {
		handle.spawn(async move {
			loop {
				let background_estimate = {
					let mut rpc = rpc_client.lock().await;
					let background_conf_target = serde_json::json!(144);
					let background_estimate_mode = serde_json::json!("ECONOMICAL");
					let resp = rpc
						.call_method::<FeeResponse>(
							"estimatesmartfee",
							&vec![background_conf_target, background_estimate_mode],
						)
						.await
						.unwrap();
					match resp.feerate_sat_per_kw {
						Some(feerate) => std::cmp::max(feerate, MIN_FEERATE),
						None => MIN_FEERATE,
					}
				};

				// ...

				fees.get(&Target::Background)
					.unwrap()
					.store(background_estimate, Ordering::Release);
				fees.get(&Target::Normal).unwrap().store(normal_estimate, Ordering::Release);
				fees.get(&Target::HighPriority)
					.unwrap()
					.store(high_prio_estimate, Ordering::Release);
				tokio::time::sleep(Duration::from_secs(60)).await;
			}
		});
	}
```

Note that the RPC response is converted to `FeeResponse`. Here is used the exact same pattern previously mentioned for `BlockchainInfo`.

`FeeResponse` is a struct that implements `TryInto` trait which converts the JSON bitcoind response to this struct.

```rust
pub struct FeeResponse {
	pub feerate_sat_per_kw: Option<u32>,
	pub errored: bool,
}

impl TryInto<FeeResponse> for JsonResponse {
	type Error = std::io::Error;
	fn try_into(self) -> std::io::Result<FeeResponse> {
		let errored = !self.0["errors"].is_null();
		Ok(FeeResponse {
			errored,
			feerate_sat_per_kw: match self.0["feerate"].as_f64() {
				Some(feerate_btc_per_kvbyte) => {
					Some((feerate_btc_per_kvbyte * 100_000_000.0 / 4.0).round() as u32)
				}
				None => None,
			},
		})
	}
}
```

## 5. Implement other RPC methods

The other methods that `RpcClient` implement follow the same scheme already presented. Makes a call through `rpc.call_method` and returns a structure that implements `TryInto<T> for JsonResponse`.

In case of `RawTx` or `SignedTx`, the definition are implemented in `convert.rs`, but `Address`, for example, is already implemented in `rust-bitcoin/src/util/address.rs`.

```rust
impl BitcoindClient {
	// ...
	pub async fn send_raw_transaction(&self, raw_tx: RawTx) {
		let mut rpc = self.bitcoind_rpc_client.lock().await;

		let raw_tx_json = serde_json::json!(raw_tx.0);
		rpc.call_method::<Txid>("sendrawtransaction", &[raw_tx_json]).await.unwrap();
	}

	pub async fn sign_raw_transaction_with_wallet(&self, tx_hex: String) -> SignedTx {
		let mut rpc = self.bitcoind_rpc_client.lock().await;

		let tx_hex_json = serde_json::json!(tx_hex);
		rpc.call_method("signrawtransactionwithwallet", &vec![tx_hex_json]).await.unwrap()
	}

	pub async fn get_new_address(&self) -> Address {
		let mut rpc = self.bitcoind_rpc_client.lock().await;

		let addr_args = vec![serde_json::json!("LDK output address")];
		let addr = rpc.call_method::<NewAddress>("getnewaddress", &addr_args).await.unwrap();
		Address::from_str(addr.0.as_str()).unwrap()
	}
	// ...
}
```

The method `fund_raw_transaction()` calls `get_est_sat_per_1000_weight` function to get the fees. This is implemented in `FeeEstimator` trait. Since the fee estimates are already allocated in the `fees` hashmap, this function just need to get the value to the corresponding target. Note that after calling the function, the value is converted back to sats/vB, which is expected by Bitcoin Core.

```rust
impl BitcoindClient {
	// ...
	pub async fn fund_raw_transaction(&self, raw_tx: RawTx) -> FundedTx {
		let mut rpc = self.bitcoind_rpc_client.lock().await;

		let raw_tx_json = serde_json::json!(raw_tx.0);
		let options = serde_json::json!({
			"fee_rate": self.get_est_sat_per_1000_weight(ConfirmationTarget::Normal) as f64 / 250.0,
			"replaceable": false,
		});
		rpc.call_method("fundrawtransaction", &[raw_tx_json, options]).await.unwrap()
	}
}

// ...

impl FeeEstimator for BitcoindClient {
	fn get_est_sat_per_1000_weight(&self, confirmation_target: ConfirmationTarget) -> u32 {
		match confirmation_target {
			ConfirmationTarget::Background => {
				self.fees.get(&Target::Background).unwrap().load(Ordering::Acquire)
			}
			ConfirmationTarget::Normal => {
				self.fees.get(&Target::Normal).unwrap().load(Ordering::Acquire)
			}
			ConfirmationTarget::HighPriority => {
				self.fees.get(&Target::HighPriority).unwrap().load(Ordering::Acquire)
			}
		}
	}
}

```

## 6. Implement `BroadcasterInterface` trait

`BroadcasterInterface` is a trait defined in `lightning/src/chain/chaininterface.rs`. It is an interface to send a transaction to the Bitcoin network. In this LDK node, it will be used when it receives the `Event::SpendableOutputs`, which is used to indicate that an output which the node should know how to spend was confirmed on chainS and is now spendable.

To broadcast the transaction, `sendrawtransaction` can be used.

```rust
impl BroadcasterInterface for BitcoindClient {
	fn broadcast_transaction(&self, tx: &Transaction) {
		let bitcoind_rpc_client = self.bitcoind_rpc_client.clone();
		let tx_serialized = serde_json::json!(encode::serialize_hex(tx));
		self.handle.spawn(async move {
			let mut rpc = bitcoind_rpc_client.lock().await;
			match rpc.call_method::<Txid>("sendrawtransaction", &vec![tx_serialized]).await {
				Ok(_) => {}
				Err(e) => {
					let err_str = e.get_ref().unwrap().to_string();
					if !err_str.contains("Transaction already in block chain")
						&& !err_str.contains("Inputs missing or spent")
						&& !err_str.contains("bad-txns-inputs-missingorspent")
						&& !err_str.contains("non-BIP68-final")
						&& !err_str.contains("insufficient fee, rejecting replacement ")
					{
						panic!("{}", e);
					}
				}
			}
		});
	}
}
```

The `encode::serialize_hex` function is implemented in `rust-bitcoin/src/consensus/encode.rs` and it serializes the transaction according to the consensus rules, so other nodes can understand the format.

`Txid` is also already implemented in `rust-bitcoin/src/hash_types.rs`, basically a 32 bytes hash.

```rust
hash_newtype!(Txid, sha256d::Hash, 32, doc="A bitcoin transaction hash/transaction ID.");
```

## Sources:

* [Why can't default minimum fee rate be changed to 0.1-0.2 sat/vByte?](https://bitcoin.stackexchange.com/questions/100861/why-cant-default-minimum-fee-rate-be-changed-to-0-1-0-2-sat-vbyte)

* [Different fee rate units - sat/vB, sat perkw, sat perkb](https://bitcoin.stackexchange.com/questions/106333/different-fee-rate-units-sat-vb-sat-perkw-sat-perkb)