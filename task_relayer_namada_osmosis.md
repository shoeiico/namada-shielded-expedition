# Summary
To complete this subclass "Operating IBC/ Interoperability infrastructure", I demonstrate how to create IBC relayer channel on the fullnodes of Namada SE testnet and Osmosis testnet.   

1. Deploy Namada fullnode on remote server which chain-id is 'shielded-expedition.88f17d1d14', IP: 213.136.71.166  
2. Deploy Osmosis fullnode on this server which chain-id is 'osmo-test-5', IP: 127.0.0.1  
3. Prepare relayer accounts for Namada and Osmosis   
4. Prepare config.toml for Hermes  
5. Add relayer accounts to Hermes  
6. Create relayer channel  
7. IBC transfer Namada <-> Osmosis  
8. Check balances to confirm if channel is operational.  

_This docuement starts from step 3 because we mainly focus on IBC relayer operations._

**Achievement:**  
Namada <-> Osmosis  
channel-451 <-> channel-5901

Transactions:  
[create channel](https://testnet.mintscan.io/osmosis-testnet/txs/1647A64400C2F1137230AD5B98302C0DA5CE6A0F855175850B691B31B5C6A394?height=5632882)  
[transfer osmo](https://testnet.mintscan.io/osmosis-testnet/txs/52DCC3BDE099A0768EF01756EB19C7EE227B68F33914BE6185D7892209B9B4ED?height=5633197)  
[receive naan](https://testnet.mintscan.io/osmosis-testnet/txs/EAB0F905A560F23D8B573E7A9488DA1070FE912E9525248C6138E8D106D2827C?height=5633241)

# Prepare accounts for Osmosis and Namada
- Generate a new account for osmos
```
osmosisd keys add shoeiico_osmosis_relayer
Enter keyring passphrase (attempt 1/3):
- address: osmo1zpvstw8rrld3p3y4sd5azzdrmhsc08rde5mnnk
  name: shoeiico_osmosis_relayer
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"A33Dcyg+5189nb+w8p/iZZHazqvYmoFBKzwH3KNC88g/"}'
  type: local

**Important** write this mnemonic phrase in a safe place.
It is the only way to recover your account if you ever forget your password.

roast collect museum rib lobster custom beach myself trash slice turn kiss december tonight stone aerobic stomach slight web tattoo size lunar cancel width
```
Get fauect to shoeiico_osmosis_relayer  

- Install Namada CLI on this server
```
cd $HOME && git clone https://github.com/anoma/namada && cd namada && git checkout v0.31.6
make build-release
cd $HOME && sudo cp "$HOME/namada/target/release/namada" /usr/local/bin/namada && sudo cp "$HOME/namada/target/release/namadac" /usr/local/bin/namadac && sudo cp "$HOME/namada/target/release/namadan" /usr/local/bin/namadan && sudo cp "$HOME/namada/target/release/namadaw" /usr/local/bin/namadaw && sudo cp "$HOME/namada/target/release/namadar" /usr/local/bin/namadar
namada --version
Namada v0.31.6

namadac utils join-network --chain-id shielded-expedition.88f17d1d14
```
- Import previous namada wallet
```
namadaw derive --alias shoeiico_namada_relayer
```
# Prepare config.toml for Hermes
mkdir $HOME/.hermes  
export BASE_DIR=$HOME/.local/share/namada  
export HERMES_CONFIG=$HOME/.hermes/config.toml  
vi $HERMES_CONFIG
```
[global]
log_level = 'info'
 
[mode]

[mode.clients]
enabled = true
refresh = true
misbehaviour = true

[mode.connections]
enabled = false

[mode.channels]
enabled = false

[mode.packets]
enabled = true
clear_interval = 10
clear_on_start = false
tx_confirmation = true

[telemetry]
enabled = false
host = '127.0.0.1'
port = 3001

[[chains]]
id = 'shielded-expedition.88f17d1d14' 
type = 'Namada'
rpc_addr = 'http://213.136.71.166:26657'  # remote Namada fullnode rpc
grpc_addr = 'http://213.136.71.166:9090' 
event_source = { mode = 'push', url = 'ws://213.136.71.166:26657/websocket', batch_delay = '500ms' } 
account_prefix = ''
key_name = 'shoeiico_namada_relayer' 
store_prefix = 'ibc'
gas_price = { price = 0.0001, denom = 'tnam1qxvg64psvhwumv3mwrrjfcz0h3t3274hwggyzcee' } 
rpc_timeout = '30s'

[[chains]]
id = 'osmo-test-5'
type = 'CosmosSdk'
rpc_addr = 'http://127.0.0.1:26657'  # Local Osmos fullnode rpc
grpc_addr = 'http://127.0.0.1:9090'
event_source = { mode = 'push', url = 'ws://127.0.0.1:26657/websocket', batch_delay = '500ms' } 
account_prefix = 'osmo'
key_name = 'shoeiico_osmosis_relayer'
address_type = { derivation = 'cosmos' }
store_prefix = 'ibc'
default_gas = 400000
max_gas = 120000000
gas_price = { price = 0.0025, denom = 'uosmo' }
gas_multiplier = 1.2
max_msg_num = 30
max_tx_size = 1800000
clock_drift = '15s'
max_block_time = '30s'
trusting_period = '4days'
trust_threshold = { numerator = '1', denominator = '3' }
rpc_timeout = '30s'
```
# Add relayer accounts to Hermes
- Install Hermes
```
export TAG="v1.7.4-namada-beta7"
export ARCH="x86_64-unknown-linux-gnu" # or "aarch64-apple-darwin"
curl -Lo /tmp/hermes.tar.gz https://github.com/heliaxdev/hermes/releases/download/${TAG}/hermes-${TAG}-${ARCH}.tar.gz
tar -xvzf /tmp/hermes.tar.gz -C /usr/local/bin
```
- Add relayer keys to Hermes
```
echo "roast collect museum rib lobster custom beach myself trash slice turn kiss december tonight stone aerobic stomach slight web tattoo size lunar cancel width" > ./mnemonic_osmo
hermes --config $HERMES_CONFIG keys add --chain osmo-test-5 --mnemonic-file ./mnemonic_osmo
hermes --config $HERMES_CONFIG keys add --chain shielded-expedition.88f17d1d14 --key-file $HOME/.local/share/namada/shielded-expedition.88f17d1d14/wallet.toml  
```
#  Create Channel
```
hermes --config $HERMES_CONFIG \
  create channel \
  --a-chain shielded-expedition.88f17d1d14 \
  --b-chain osmo-test-5 \
  --a-port transfer \
  --b-port transfer \
  --new-client-connection --yes
```
```
2024-02-26T03:48:02.256900Z  INFO ThreadId(01) running Hermes v1.7.4+38f41c6
2024-02-26T03:48:02.592797Z  INFO ThreadId(01) Creating new clients, new connection, and a new channel with order ORDER_UNORDERED
2024-02-26T03:48:17.968223Z  INFO ThreadId(01) foreign_client.create{client=osmo-test-5->shielded-expedition.88f17d1d14:07-tendermint-0}: 🍭 client was created successfully id=07-tendermint-1570
2024-02-26T03:48:23.880685Z  INFO ThreadId(01) foreign_client.create{client=shielded-expedition.88f17d1d14->osmo-test-5:07-tendermint-0}: 🍭 client was created successfully id=07-tendermint-2382
2024-02-26T03:48:40.408401Z  INFO ThreadId(01) 🥂 shielded-expedition.88f17d1d14 => OpenInitConnection(OpenInit { Attributes { connection_id: connection-681, client_id: 07-tendermint-1570, counterparty_connection_id: None, counterparty_client_id: 07-tendermint-2382 } }) at height 0-68917
2024-02-26T03:49:19.483570Z  INFO ThreadId(01) 🥂 osmo-test-5 => OpenTryConnection(OpenTry { Attributes { connection_id: connection-2222, client_id: 07-tendermint-2382, counterparty_connection_id: connection-681, counterparty_client_id: 07-tendermint-1570 } }) at height 5-5632895
2024-02-26T03:49:48.292606Z  INFO ThreadId(01) 🥂 shielded-expedition.88f17d1d14 => OpenAckConnection(OpenAck { Attributes { connection_id: connection-681, client_id: 07-tendermint-1570, counterparty_connection_id: connection-2222, counterparty_client_id: 07-tendermint-2382 } }) at height 0-68924
2024-02-26T03:50:08.415902Z  INFO ThreadId(01) 🥂 osmo-test-5 => OpenConfirmConnection(OpenConfirm { Attributes { connection_id: connection-2222, client_id: 07-tendermint-2382, counterparty_connection_id: connection-681, counterparty_client_id: 07-tendermint-1570 } }) at height 5-5632908
2024-02-26T03:50:11.497825Z  INFO ThreadId(01) connection handshake already finished for Connection { delay_period: 0ns, a_side: ConnectionSide { chain: BaseChainHandle { chain_id: shielded-expedition.88f17d1d14 }, client_id: 07-tendermint-1570, connection_id: connection-681 }, b_side: ConnectionSide { chain: BaseChainHandle { chain_id: osmo-test-5 }, client_id: 07-tendermint-2382, connection_id: connection-2222 } }
2024-02-26T03:50:25.601446Z  INFO ThreadId(01) 🎊  shielded-expedition.88f17d1d14 => OpenInitChannel(OpenInit { port_id: transfer, channel_id: channel-451, connection_id: None, counterparty_port_id: transfer, counterparty_channel_id: None }) at height 0-68927
2024-02-26T03:50:44.203622Z  INFO ThreadId(01) 🎊  osmo-test-5 => OpenTryChannel(OpenTry { port_id: transfer, channel_id: channel-5901, connection_id: connection-2222, counterparty_port_id: transfer, counterparty_channel_id: channel-451 }) at height 5-5632917
2024-02-26T03:51:09.222117Z  INFO ThreadId(01) 🎊  shielded-expedition.88f17d1d14 => OpenAckChannel(OpenAck { port_id: transfer, channel_id: channel-451, connection_id: connection-681, counterparty_port_id: transfer, counterparty_channel_id: channel-5901 }) at height 0-68931
2024-02-26T03:51:25.162830Z  INFO ThreadId(01) 🎊  osmo-test-5 => OpenConfirmChannel(OpenConfirm { port_id: transfer, channel_id: channel-5901, connection_id: connection-2222, counterparty_port_id: transfer, counterparty_channel_id: channel-451 }) at height 5-5632927
2024-02-26T03:51:28.292791Z  INFO ThreadId(01) channel handshake already finished for Channel { ordering: ORDER_UNORDERED, a_side: ChannelSide { chain: BaseChainHandle { chain_id: shielded-expedition.88f17d1d14 }, client_id: 07-tendermint-1570, connection_id: connection-681, port_id: transfer, channel_id: channel-451, version: None }, b_side: ChannelSide { chain: BaseChainHandle { chain_id: osmo-test-5 }, client_id: 07-tendermint-2382, connection_id: connection-2222, port_id: transfer, channel_id: channel-5901, version: None }, connection_delay: 0ns }
SUCCESS Channel {
    ordering: Unordered,
    a_side: ChannelSide {
        chain: BaseChainHandle {
            chain_id: ChainId {
                id: "shielded-expedition.88f17d1d14",
                version: 0,
            },
            runtime_sender: Sender { .. },
        },
        client_id: ClientId(
            "07-tendermint-1570",
        ),
        connection_id: ConnectionId(
            "connection-681",
        ),
        port_id: PortId(
            "transfer",
        ),
        channel_id: Some(
            ChannelId(
                "channel-451",
            ),
        ),
        version: None,
    },
    b_side: ChannelSide {
        chain: BaseChainHandle {
            chain_id: ChainId {
                id: "osmo-test-5",
                version: 5,
            },
            runtime_sender: Sender { .. },
        },
        client_id: ClientId(
            "07-tendermint-2382",
        ),
        connection_id: ConnectionId(
            "connection-2222",
        ),
        port_id: PortId(
            "transfer",
        ),
        channel_id: Some(
            ChannelId(
                "channel-5901",
            ),
        ),
        version: None,
    },
    connection_delay: 0ns,
}
```
Now, channels are ready:  
Namada: 'channel-451'   
Osmosis: 'channel-5901'   

- We check balances before IBC transfer.
```
namadac balance --owner shoeiico_namada_relayer --node http://213.136.71.166:26657
naan: 259.672897

osmosisd query bank balances osmo1zpvstw8rrld3p3y4sd5azzdrmhsc08rde5mnnk
balances:
- amount: "249983700"
  denom: uosmo
pagination:
  next_key: null
  total: "0"
```

# IBC transfer Namada <-> Osmosis 
- Start Hermes
```
hermes --config $HOME/.hermes/config.toml start
```
- We tranfer 1 naan from namada to osmosis via channel-451
```
namadac --base-dir $BASE_DIR \
    ibc-transfer \
    --amount 1 \
    --source shoeiico_namada_relayer \
    --receiver osmo1zpvstw8rrld3p3y4sd5azzdrmhsc08rde5mnnk \
    --token naan \
    --channel-id channel-451 \
    --node http://213.136.71.166:26657 \
    --memo tpknam1qp0cyvrh4r5anmslnyda88nrem4easja386jlug369mcw3l3fjqkvueqwqs
```
```
Enter your decryption password: 
Transaction added to mempool.
Wrapper transaction hash: E4D73493D344116F9E8ECF191EC9EB9CFCA69AD68F69B5823751360F3E495E8A
Inner transaction hash: 9DE7FD3F4AFC34897F0070CF7D1BAFF692B085275D31F6CE96E0E9D0B8A38749
Wrapper transaction accepted at height 69044. Used 26 gas.
Waiting for inner transaction result...
Transaction was successfully applied at height 69045. Used 6193 gas.
```
- We tranfer 1000000uosmo from osmosis to namada via channel-5901
```
osmosisd tx ibc-transfer transfer \
  transfer \
  channel-5901 \
  tnam1qzanc7wtv7qufuw8rzg5tteua2hccks4vyxcnmd7 \
  1000000uosmo \
  --from shoeiico_osmosis_relayer \
  --gas auto \
  --gas-prices 0.035uosmo \
  --gas-adjustment 1.2 \
  --node "http://127.0.0.1:26657" \
  --home "$HOME/.osmosisd" \
  --chain-id osmo-test-5 \
  --yes
```
```
Enter keyring passphrase (attempt 1/3):
gas estimate: 131186
code: 0
codespace: ""
data: ""
events: []
gas_used: "0"
gas_wanted: "0"
height: "0"
info: ""
logs: []
raw_log: '[]'
timestamp: ""
tx: null
txhash: 52DCC3BDE099A0768EF01756EB19C7EE227B68F33914BE6185D7892209B9B4ED
```

- Check balances on both sides.
```
namadac balance --owner shoeiico_namada_relayer --node http://213.136.71.166:26657
naan: 253.672897
transfer/channel-451/uosmo: 1000000

osmosisd query bank balances osmo1zpvstw8rrld3p3y4sd5azzdrmhsc08rde5mnnk
balances:
- amount: "1"
  denom: ibc/D7C4C279DBE2F7B4140C8AFBB00D16EBBC71177FA65ABB135610F6B497108545
- amount: "248979108"
  denom: uosmo
pagination:
  next_key: null
  total: "0"
```
We confirm that the account shoeiico_namada_relayer on Namada has received _'transfer/channel-451/uosmo: 1000000'_ and the account shoeiico_osmosis_relayer has received 1 _'denom: ibc/D7C4C279DBE2F7B4140C8AFBB00D16EBBC71177FA65ABB135610F6B497108545'_   

Now, We've created channels successfully!
