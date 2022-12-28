# Nibiru-

# ОС Ubuntu 20.04

### Минимальные системные требования 

4x процессора; чем выше тактовая частота, тем лучше
16 ГБ ОЗУ
250 ГБ памяти (SSD или NVME)
### Официально рекомендуемые системные требования
8 процессоров
64 ГБ ОЗУ
1 ТБ памяти (SSD или NVME)

## 1. Обновление пакетов

sudo apt update && sudo apt upgrade -y

## 2. Устанавливаем необходимые утилиты

apt install curl iptables build-essential git wget jq make gcc nano tmux htop nvme-cli pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev -y

##  3. Устанавливаем GO

ver="1.19.1" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version

## 4. Устанавливаем бинарные файлы

git clone https://github.com/NibiruChain/nibiru && cd nibiru
git checkout v0.16.2
make install

nibid version --long | grep -e version -e commit

## 5. Инициализируем ноду, задаем СВОЕ имя

nibid init ИМЯ_НОДЫ --chain-id nibiru-testnet-2

## 6.Скачиваем и проверяем Genesis

curl -s https://rpc.testnet-2.nibiru.fi/genesis | jq -r .result.genesis > $HOME/.nibid/config/genesis.json

## --Проверим генезис
sha256sum ~/.nibid/config/genesis.json

## 7. Скачиваем Addr book

wget -O $HOME/.nibid/config/addrbook.json "https://share.utsa.tech/nibiru/addrbook.json"

## 8. Настраиваем конфигурацию ноды

nibid config chain-id nibiru-testnet-2

nibid config keyring-backend os

sed -i.bak -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.025unibi\"/;" ~/.nibid/config/app.toml

external_address=$(wget -qO- eth0.me)
sed -i.bak -e "s/^external_address *=.*/external_address = \"$external_address:26656\"/" $HOME/.nibid/config/config.toml

peers=""
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.nibid/config/config.toml

seeds="dabcc13d6274f4dd86fd757c5c4a632f5062f817@seed-2.nibiru-testnet-2.nibiru.fi:26656,a5383b33a6086083a179f6de3c51434c5d81c69d@seed-1.nibiru-testnet-2.nibiru.fi:26656"
sed -i.bak -e "s/^seeds =.*/seeds = \"$seeds\"/" $HOME/.nibid/config/config.toml

sed -i 's/max_num_inbound_peers =.*/max_num_inbound_peers = 50/g' $HOME/.nibid/config/config.toml
sed -i 's/max_num_outbound_peers =.*/max_num_outbound_peers = 25/g' $HOME/.nibid/config/config.toml

sed -i -e "s/^filter_peers *=.*/filter_peers = \"true\"/" $HOME/.nibid/config/config.toml

## 9. Настраиваем прунинг (одной командой)

pruning="custom"
pruning_keep_recent="1000"
pruning_interval="10"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.nibid/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.nibid/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.nibid/config/app.toml

## 10. Создаем свой сервисный файл

tee /etc/systemd/system/nibid.service > /dev/null <<EOF
[Unit]
Description=nibid
After=network-online.target

[Service]
User=$USER
ExecStart=$(which nibid) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

## 11. Запускаем узел

systemctl daemon-reload
systemctl enable nibid
systemctl restart nibid && journalctl -u nibid -f -o cat


## 12. Ждем синхронизацию Ноды (false)

проверяем статус синхронизации

nibid status 2>&1 | jq .SyncInfo


## 13. Cоздаем или восстанавливаем кошелек и СОХРАНЯЕМ мнемонику. Задаем СВОЕ имя кошелька

nibid keys add ИМЯ_КОШЕЛЬКА --keyring-backend os

--восстановить кошелек

nibid keys add ИМЯ_КОШЕЛЬКА --recover --keyring-backend os

##14. Пополняем свой кошелек командой или запросом 

$request АДРЕС_КОШЕЛЬКА

через кран в дискорде (кран должен быть в рабочем состоянии, когда вы будете запрашивать монеты)

--проверяем баланс кошелька

nibid query bank balances АДРЕС_КОШЕЛЬКА

## 15. Создаем валидатора, задаем СВОЕ имя ноды и вписываем свой кошелек

nibid tx staking create-validator \
--chain-id nibiru-testnet-2 \
--commission-rate 0.05 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.1 \
--min-self-delegation "1000000" \
--amount 1000000unibi \
--pubkey $(nibid tendermint show-validator) \
--moniker "ИМЯ_НОДЫ" \
--from ИМЯ_КОШЕЛЬКА \
--fees 5000unibi


## 16. Отправляем ссылку на валидатора в канал #validator-role-request.


### Полезные команды

--проверить блоки
nibid status 2>&1 | jq ."SyncInfo"."latest_block_height"

--проверить логи
journalctl -u nibid -f -o cat

--проверить статус
curl localhost:26657/status

--проверить баланс
nibid query bank balances АДРЕС_КОШЕЛЬКА
--собрать реварды со всех валидаторов, которым делегировали
nibid tx distribution withdraw-all-rewards --from ИМЯ_КОШЕЛЬКА --fees 5000unibi -y

--заделегировать себе в стейк (так отправляется 1 монетa)
nibid tx staking delegate АДРЕС_ВАЛИДАТОРА 1000000unibi --from ИМЯ_КОШЕЛЬКА --fees 5000unibi -y

--отправить монеты на другой адрес
nibid tx bank send ИМЯ_СВОЕГО_КОШЕЛЬКА АДРЕС_КОШЕЛЬКА_ПОЛУЧАТЕЛЯ 1000000unibi --fees 5000unibi -y

--выбраться из тюрьмы
nibid tx slashing unjail --from ИМЯ_КОШЕЛЬКА --fees 5000unibi -y
--вывести список кошельков
nibid keys list

--удалить кошелек
nibid keys delete ИМЯ_КОШЕЛЬКА
Удалить ноду

systemctl stop nibid && \
systemctl disable nibid && \
rm /etc/systemd/system/nibid.service && \
systemctl daemon-reload && \
cd $HOME && \
rm -rf .nibid nibiru && \
rm -rf $(which nibid)
