# Stride testnet, Cosmos testnet (GAIA) relayer kurulumu.

![image](https://user-images.githubusercontent.com/3939786/181860016-606e6dc6-4942-4275-86c5-05029c90dbb4.png)


## Stride<>GAIA arasında relay kurulma işlemi.
İlgili mesaja ulaşmak için [link'e](https://discord.com/channels/988945059783278602/996875657512489130/1001943107593584651 "link'e") tıklayabilirsiniz.

Kurulum adımlarını sırası ile anlatacağım.

### Relay program kurulumu

*Aşağıdaki komutların her biri tek satırdır.*
```bash
cd $HOME
mkdir -p $HOME/.hermes/bin
sudo apt update && sudo apt upgrade -y
sudo apt install unzip -y
wget "https://github.com/informalsystems/ibc-rs/releases/download/v0.15.0/hermes-v0.15.0-x86_64-unknown-linux-gnu.tar.gz"
tar -C $HOME/.hermes/bin/ -vxzf hermes-v0.15.0-x86_64-unknown-linux-gnu.tar.gz	rm hermes-v0.15.0-x86_64-unknown-linux-gnu.tar.gz
cp $HOME/.hermes/bin/hermes /usr/local/bin/
hermes version
```
Hermes **sürüm bilgisini görebiliyorsanız** hermes programı düzgün kurulmuş demektir.

------------


### Hermes isimli sistem servisinin hazırlanması
*Aşağıdaki tüm komutlar tek satırdır. Komple kopyalanıp sunucuya yapıştırılmalı.*
```bash
sudo tee /etc/systemd/system/hermes.service > /dev/null <<EOF
[Unit]
Description=HERMES
After=network.target
[Service]
Type=simple
User=$USER
ExecStart=$(which hermes) start
Restart=on-failure
RestartSec=5
LimitNOFILE=6000
[Install]
WantedBy=multi-user.target
EOF
```
Ardından aşağıdaki komutları sırası ile yazacağız.
```bash
sudo systemctl daemon-reload
sudo systemctl enable hermes
```
Hermes kurulumunu yaptığımıza göre konfigurasyon dosyamızı oluşturabiliriz.

------------
### config.toml dosyası oluşturma.
Aşağıdaki komut ile config.toml dosyasını oluşturup nano ile açıyoruz.
```bash
nano $HOME/.hermes/config.toml
```
Aşağıdaki tüm satırları ilk karakterden son karaktere kadar kopyalıyoruz ve nano ile açtığımız config dosyamızın içine yapıştırıyoruz. Başını ve sonunu mutlaka kontrol edin.
**memo_prefix** kısmına kendi Discord ID'nizi yazın. *(örnek deneme#1234)*
```go
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
clear_interval = 100
clear_on_start = true
tx_confirmation = true
[rest]
enabled = true
host = '0.0.0.0'
port = 3000
[telemetry]
enabled = true
host = '0.0.0.0'
port = 3001
[[chains]]
id = 'STRIDE-TESTNET-2'
rpc_addr = 'http://stride.stake-take.com:26657'
grpc_addr = 'http://stride.stake-take.com:9090'
websocket_addr = 'ws://stride.stake-take.com:26657/websocket'
rpc_timeout = '10s'
account_prefix = 'stride'
key_name = 'hermes_stride'
store_prefix = 'ibc'
max_tx_size = 100000
max_gas = 20000000
gas_price = { price = 0.001, denom = 'ustrd' }
gas_adjustment = 0.1
max_msg_num = 15
clock_drift = '5s'
trusting_period = '8hours'
memo_prefix="DiscordID#1234"
trust_threshold = { numerator = '1', denominator = '3' }
[[chains]]
id = 'GAIA'
rpc_addr = 'http://stride.stake-take.com:46657'
grpc_addr = 'http://stride.stake-take.com:9490'
websocket_addr = 'ws://stride.stake-take.com:46657/websocket'
rpc_timeout = '10s'
account_prefix = 'cosmos'
key_name = 'hermes_gaia'
store_prefix = 'ibc'
max_tx_size = 100000
max_gas = 30000000
gas_price = { price = 0.001, denom = 'uatom' }
gas_adjustment = 0.1
max_msg_num = 15
clock_drift = '5s'
trusting_period = '8hours'
memo_prefix= "DiscordID#1234"
trust_threshold = { numerator = '1', denominator = '3' }
```
Dosyanızı oluşturduktan sonra. `hermes config validate` komutunu yazın ve aşağıdaki gibi bir çıktı alacaksınız.

![image](https://user-images.githubusercontent.com/3939786/181859892-1a0b5359-2efe-4c9d-a729-68aa014ec729.png)


Eğer yukarıdakinden farklı bir çıktı alıyorsanız hatanın hangi satır hangi harfte olduğu ile ilgili bir mesaj alacaksınız. O satırdaki yazım hatasını düzelterek komutu tekrar edin.

------------
###  Cüzdan Oluşturma/Yükleme
Eğer hali hazırda var olan stride ve cosmos-testnet (GAIA) cüzdanlarını kullanmak istemiyorsanız ve yeni bir cüzdan oluşturacaksanız, aşağıdaki adımları gerçekleştirin.
#### - Cüzdan oluşturma
Stride'ın (strided) ve cosmos testnet'in (gaiad) yüklü olduğu sunucunuzda şu komutu çalıştırın.
```bash
strided keys add hermes_stride
gaiad keys add hermes_gaia
```
> ##### --------NOT------- **Cüzdanları oluştururken lütfen mnemonic seed lerini (12 kelime) güvenli şekilde not edin.**

#### - Var olan cüzdanı kullanmak
Relayer yukarıdaki şekilde yeni bir cüzdan oluşturduysanız o cüzdanlarınızın içinde biraz coin/token bulunmalı. Stride cüzdanında ustrd, cosmos testnet (GAIA) cüzdanınızda uatom bulunmalı. Ne kadar çok coin/token bulunursa o kadar büyük siparişlerde sizin relayer'inizi kullanırlar.

Cüzdanlarınızı hermes uygulamamıza yüklememiz gerekiyor. bunu yapmak için şu komutları uygulamamız gerekli.

> ##### --------NOT------- **mnemonic ler uydurmadır, ve kendi kelimelerinizi girmeniz gerekmektedir.**

```bash
hermes keys restore GAIA -n hermes_gaia -m "economy deer banana melt remember outdoor moral pledge join link april guitar practice damage coin test luxury behave rotate vapor fashion analyst winter sand"
```
```bash
hermes keys restore STRIDE-TESTNET-2 -n hermes_stride -m "economy deer banana melt remember outdoor moral pledge join link april guitar practice damage coin test luxury behave rotate vapor fashion analyst winter sand"
```
Cüzdanların hermes'e yüklendiğini kontrol edin.
```bash
hermes keys list STRIDE-TESTNET-2
hermes keys list GAIA
```
Aağıdaki gibi bir çıktı almalısınız.

![image](https://user-images.githubusercontent.com/3939786/181859916-4e00b98c-85d6-422b-ad80-298aa1c3b1df.png)


Şimdi hermesi çalıştırma zamanı geldi. Şimdiye kadar herşeyi doğru yaptıysanız sorun çıkmadan çalıştırabilirsiniz. Size tavsiyem screen yada **tmux** gibi bir kabuk pencere yöneticisi kullanmanız yönünde. Bu sayede her seferinde komutlar yazarak kontrol etmenize gerek kalmaz. **Tmux hakkında bilgi için** [link'e](https://sudo.ubuntu-tr.net/tmux-kuvvetli-ucbirim-yonetimi "**link'e**") **tıklayın.**

Aşağıdaki komutu çalıştırarak hermes uygulamasının başlamasını ve logları ekrana yazmasını sağlayabilirsiniz.
```bash
sudo systemctl restart hermes && journalctl -u hermes -f
```
ctrl+c tuş kombinasyonu ile logların gözükmesini durdurup diğer işlerinizi yapabilirsiniz.
aşağıdaki komut ile tekrar hermes loglarının görünmesini sağlayabilrisiniz.
```bash
journalctl -u hermes -f
```

Relayer'imiz artık düzgün çalışmaya başlamış olması gerekiyor. Kendisi gerekli kanallara bağlanarak bağlantı beklemeli ve bağlantı kurmalı. Sistemde sizin cüzdanlarınızdaki bakiyelere uygun miktarda coin/token olan istekler gelince sizin relayer'iniz cevap verecek ve gerekli transferi gerçekleştirecektir. Hermes e tanımladığınız cüzdanlarda mutlaka bakiye olmalı.

thx to kj89 
