##  _Kichain ibc relayer guide_

Special thanks to [Confuzerpy](https://github.com/Confuzerpy) and [goooodnes](https://github.com/goooodnes) from [Let's encrypt community](https://t.me/kichain_ru)

in this guide we will using ibc between [kichain-t-4](https://github.com/KiFoundation/ki-testnet-challenge) and [testnet-croeseid-4](https://crypto.org/docs/getting-started/croeseid-testnet.html#step-1-get-the-crypto-org-chain-testnet-binary)

#### 1. Install the latest ibc relayer from the [official repo](https://github.com/cosmos/relayer)
Once relayer installed check the relayer version. In my case it was 0.9.3 version

```sh 
rly version
version: v0.9.3
commit: 4b81fa59055e3e94520bdfae1debe2fe0b747dc1
cosmos-sdk: v0.42.4
go: go1.15.11 linux/amd64
```

#### 2. Intialize relayer
```
rly config init
```
#### 3. Once initialized lets configure it:
```
cd && mkdir relayer_config && cd relayer_config
```
Create config for the kichain-t-4 network
```
nano ki_config.json
{
"chain-id": "kichain-t-4",
  "rpc-addr": "http://127.0.0.1:26657",
  "account-prefix": "tki",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025utki",
  "trusting-period": "48h"
}
```
Create config for the croeseid testnet network
- _note in my case i configured my croeseid testnet to operate with port 26552. Your port might be different!_
```
nano cro_config.json

{
  "chain-id": "testnet-croeseid-4",
  "rpc-addr": "http://127.0.0.1:26652",
  "account-prefix": "tcro",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025basetcro",
  "trusting-period": "48h"
}
```

#### 4. Next step we should add our configs to the relayer

```rly chains add -f ki_config.json```

```rly chains add -f cro_config.```
#### 5. Well now we have to generate or import wallets.
- _Notice in my case im using key name kichain_key and cro_key.
You can specify watever you want name._

###### Generating new keys
```rly keys add kichain-t-4 kichain_key```

```rly keys add testnet-croeseid-4 --coin-type 1 cro_key```
	
#### WARNING! 
- We using coin type 1 when generating croeseid wallet
  because croesseid using DIFFERENT hd path!

###### Import existing keys
```rly keys restore kichain-t-4 kichain_key "MNEMONIC"```

```rly keys restore "testnet-croeseid-4" cro_key --coin-type 1 "MNEMONIC"```



#### 6. Check that everything is OK

```sh
rly keys list kichain-t-4

key(0): kichain_key -> tki1__YOUR_WALLET
```
```sh
rly keys list testnet-croeseid-4

key(0): cro_key -> tcro__YOUR_WALLET
```
#### 7. Add the newly created keys to the config of the relayer:

```rly chains edit kichain-t-4 key kichain_key```

```rly chains edit testnet-croeseid-4 key cro_key```

#### 8. Change timeout in the relayer settings

```nano ~/.relayer/config/config.yaml```

Find the line ```timeout: 10s``` and replace to ```timeout: 10m```

#### 9. Make sure that you have enough funds in you wallets.
- If you dont have funds. Claim some through faucet.

######  [Kichain faucet](https://discord.gg/D3vvEeBpE5). 
Under kichain testnet-challenge channel you will find faucet thread.

######  [Croeseid faucet](https://crypto.org/faucet).

#### 10. Check the wallet balance of the relayer.

```
rly q balance kichain-t-4
10000000utki
```

```
rly q balance testnet-croeseid-4
100000000basetcro
```


#### 11. Initialize clients for both networks:
```rly light init kichain-t-4 -f```

```rly light init testnet-croeseid-4 -f```

#### 12. Create a channel between the two networks:
- ###### rly paths generate [src-chain-id] [dst-chain-id] [name] [flags]

```rly paths generate kichain-t-4 testnet-croeseid-4 transfer --port=transfer```

#### 13. Next step lets create channel from croeseid to kichain
Note this command might take some time. If you have some errors try to repeat several times.
Creating channel from croeseid to kichain

```rly tx link transfer```

If operation completes successfull the otput of the last line should be like:

```★ Channel created: [kichain-t-4]chan{channel-45}port{transfer} -> [testnet-croeseid-4]chan{channel-18}port{transfer}```

#### 14. Checking the path by typing: 

```
rly paths list -d
0: transfer          -> chns(✔) clnts(✔) conn(✔) chan(✔) (kichain-t-4:transfer<>testnet-croeseid-4:transfer)
```

#### 15. Start relayer as a service
In our service file we are using ```rly tx link-then-start transfer --timeout 30s``` command.
First it will link our channel and then relayer will immediately started with timeout 30 seconds.

```
sudo tee /etc/systemd/system/rlyd.service > /dev/null <<EOF
[Unit]
Description=relayer service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which rly) tx link-then-start transfer --timeout 30s
Restart=always
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```
Start rlyd daemon
```
sudo systemctl daemon-reload
sudo systemctl enable rlyd
sudo systemctl start rlyd
```

#### 16. Send some coins

From croeseid to kichain

```rly tx transfer testnet-croeseid-4 kichain-t-4 1000000basetcro $(rly chains address kichain-t-4) --path transfer```

The output should be:
###### ✔ [testnet-croeseid-4]@{289697} - msg(0:transfer) hash(1C1....your_hash....07)


From kichain to croeseid

```rly tx transfer kichain-t-4 testnet-croeseid-4 666000utki $(rly chains address testnet-croeseid-4) --path transfer```

The output should be:
###### ✔ [kichain-t-4]@{221552} - msg(0:transfer) hash(E1....your_hash....6)



Example of txs from kichain to croeseid:

[E929A4C1FB860F7850B18E5C0F43E60C5D4434A066428B06C872D0640214F1DA](https://ki.thecodes.dev/tx/E929A4C1FB860F7850B18E5C0F43E60C5D4434A066428B06C872D0640214F1DA)

[6A4E115EF029501FEA54CABA2A0A05D306216422D8D62A0A5BCB667953FA5D3F](https://ki.thecodes.dev/tx/6A4E115EF029501FEA54CABA2A0A05D306216422D8D62A0A5BCB667953FA5D3F)

[E1B62CC37BF895CFB7B74944D1B8BB9C9AF403158F65EB6F55318AA25BD93BF6](https://ki.thecodes.dev/tx/E1B62CC37BF895CFB7B74944D1B8BB9C9AF403158F65EB6F55318AA25BD93BF6)

[B52FBC271B4D7D070ED30B81614A74DABA350D6C7CF2B32BF022EA751EF28479](https://ki.thecodes.dev/tx/B52FBC271B4D7D070ED30B81614A74DABA350D6C7CF2B32BF022EA751EF28479)

[05803A35AE80361ABBC76C239CA8F9DB3B118138CB7B13492312D37AFD48D1C3](https://ki.thecodes.dev/tx/05803A35AE80361ABBC76C239CA8F9DB3B118138CB7B13492312D37AFD48D1C3)

Example of txs from croeseid to kichain:

[6E76A5BC2C14113AA231E4BE27C64252F2C00B19E7BDA3C8606F68D80F72AD0D](https://ki.thecodes.dev/tx/6E76A5BC2C14113AA231E4BE27C64252F2C00B19E7BDA3C8606F68D80F72AD0D)

## Helpful links:
[Kichain testnet explorer 1](https://kichain-t-3.blockchain.ki/)

[Kichain testnet explorer 2](https://ki.thecodes.dev/)

[Croeseid testnet explorer](https://crypto.org/explorer/croeseid4/)

[KI Foundation discord](https://discord.gg/DSSUC7Tt)

