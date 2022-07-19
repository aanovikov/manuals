# Forta Mainnet

**Официальные требования к серверу** :

- CPU with 4+ cores

- 16GB RAM

- 100GB SSD

>На данный момент хватает VPS 4 cpu 4GB 50SSD

**Предварительно потребуется выполнить данные операции:**

>На момент написания мануала для запуска ноды нужно 500 FORT, но с 30.09.2022 минимальный стейк увеличится до 2500 FORT.
[пост в дискорде](https://discord.com/channels/869983523371642921/869983523816226898/997518693749755985)

Подготовка:
1) Понадобится адрес владельца ноды, можно создать новый или использовать имеющийся - ваш личный (не биржевой) кошелёк в сети эфира (подойдёт metamask). На него перевести ETH эквивалентно ~$10 для оплаты комиссии.
2) Купить 500 FORT, например на Kucoin и перевести их на адрес владельца ноды (адрес из пункта 1). Дополнительно понадобится 20 FORT для оплаты комиссии за вывод с биржи, если покупаете на Kucoin (комиссия может меняться).
3) Купить 0.1 MATIC на любой бирже. Позже токены нужно будет перевести на адрес сканера для регистрации.
4) Перевести ещё 0.1 MATIC на адрес владельца ноды.
5) Перевести токены из сети Ethereum в сеть Polygon, например по [этой](https://docs.forta.network/en/latest/bridging-fort/) инструкции.

# Получаем ссылку для взаимодействия с RPC

Зарегистрировать аккаунт в [Alchemy](https://auth.alchemyapi.io/signup). 

Логинимся на Alchemy и переходим в [дашборд](https://dashboard.alchemyapi.io/). Справа нажать `Create app`

![image](https://user-images.githubusercontent.com/23663132/179503142-05c0419a-1aea-4bc9-ac25-2ee2bbe98649.png)

Ввести название проекта, выбрать `chain: Polygon` и `Network: Polygon Mainnet` и нажать `Create app`

![image](https://user-images.githubusercontent.com/23663132/179503596-80755c63-6dfd-4bb3-a35a-154956209a84.png)

Нажать `View key` и скопировать ссылку

![image](https://user-images.githubusercontent.com/23663132/179506265-e43599ca-3c7e-4601-a1e4-02698acbd281.png)


# Установка ноды

**Назначаем переменные**
```bash
echo "export FORTA_PASSPHRASE=<YOUR_PASSPHRASE>" >> ~/.bash_profile # <YOUR_PASSPHRASE> заменить ваш пароль
echo "export FORTA_OWNER_ADDRESS=<YOUR_FORTA_OWNER_ADDRESS>" >> ~/.bash_profile # <YOUR_FORTA_OWNER_ADDRESS> заменить на адрес владельца ноды (пункт 1)
echo "export FORTA_RPC_URL=<YOUR_FORTA_RPC_URL>" >> ~/.bash_profile # <YOUR_FORTA_RPC_URL> заменить на ссылку полученную в пункте 5
echo "export FORTA_PROXY_RPC_URL=<YOUR_FORTA_PROXY_RPC_URL>" >> ~/.bash_profile # <YOUR_FORTA_PROXY_RPC_URL> заменить на https://polygon-rpc.com или другого провайдера
echo "export FORTA_DIR=~/.forta" >> ~/.bash_profile
source ~/.bash_profile
```
**Обновляем систему и устанавливаем зависимости**

```bash
sudo apt-get update && apt-get upgrade -y && sudo apt-get install jq ncdu tmux -y
```


**Устанавливаем Docker**

```bash

sudo apt-get install ca-certificates curl gnupg lsb-release -y

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
```

**Создаём файл daemon.json в папке /etc/docker**

```bash
tee /etc/docker/daemon.json > /dev/null <<EOF
{
   "default-address-pools": [
        {
            "base":"172.17.0.0/12",
            "size":16
        },
        {
            "base":"192.168.0.0/16",
            "size":20
        },
        {
            "base":"10.99.0.0/16",
            "size":24
        }
    ]
}
EOF

systemctl restart docker
```
**Устанавливаем Forta Scanner**
```bash
sudo curl https://dist.forta.network/pgp.public -o /usr/share/keyrings/forta-keyring.asc -s
echo 'deb [signed-by=/usr/share/keyrings/forta-keyring.asc] https://dist.forta.network/repositories/apt stable main' | sudo tee -a /etc/apt/sources.list.d/forta.list
sudo apt-get update
sudo apt-get install forta
```

**Создаём сервисный файл для запуска ноды**

```bash
sudo tee /etc/systemd/system/forta.service > /dev/null <<EOF
[Unit]
Description=Forta
After=network-online.target
Wants=network-online.target systemd-networkd-wait-online.service
StartLimitIntervalSec=500
StartLimitBurst=5
[Service]
Restart=on-failure
RestartSec=15s
Environment="FORTA_DIR=$FORTA_DIR"
Environment="FORTA_PASSPHRASE=$FORTA_PASSPHRASE"
ExecStart=/usr/bin/forta run
[Install]
WantedBy=multi-user.target
EOF
```

**Инициализируем ноду**

>*После выполнения следующей команды на экран будет выведен адрес сканера, который можно посмотреть командой `forta account address`*
```bash
forta init --passphrase $FORTA_PASSPHRASE
```
>**`Обязательно сохраните в надёжное место содержимое папки ~/.forta/.keys`**

**Отправляем 0.1 MATIC на полученный выше адрес сканера `Scanner address`**

**Создание файла конфигурации сканера**

```bash
mv ~/.forta/config.yml ~/.forta/config.yml.backup

tee ~/.forta/config.yml > /dev/null <<EOF
chainId: 137

scan:
  jsonRpc:
    url: $FORTA_RPC_URL
	
trace:
  enabled: false
  
jsonRpcProxy:
  jsonRpc:
    url: $FORTA_PROXY_RPC_URL
EOF
```

**Теперь нужно зарегистрировать сканер**

> Чтоб регистрация прошла успешно, монеты уже должны быть на адресе сканера. Проверить можно [здесь](https://polygonscan.com/) введя в поле поиска адрес сканера.

```bash
forta register --owner-address $FORTA_OWNER_ADDRESS
```
**Запускаем ноду**
```bash
systemctl daemon-reload
systemctl enable forta.service
systemctl start forta
```

# Делаем стейк на адрес сканера

1. Переходим на страницу контракта токена в секцию [Write as Proxy](https://polygonscan.com/address/0x9ff62d1FC52A907B6DCbA8077c2DDCA6E6a9d3e1#writeProxyContract)

2. [Добавляем](https://docs.polygon.technology/docs/develop/metamask/config-polygon-on-metamask#polygon-scan) в Metamask сеть Polygon и переключаем сеть

![image](https://user-images.githubusercontent.com/23663132/179515526-4672bf37-247c-4853-bc2b-f3967d9223f4.png)

3. Нажимаем `Connect to web3` и подключаем Metamask

![image](https://user-images.githubusercontent.com/23663132/179515212-339acca5-3e91-4651-914a-ef4bca2ce979.png)

4. Утверждаем количество монет, которые хотим застейкать, в полях вводим следующие значения:

spender: 0xd2863157539b1D11F39ce23fC4834B62082F6874

amount: количество которое хотите застейкать, если 500 FORT то значение будет 500000000000000000000

> *такое число, т.к. токен FORT имеет 18 знаков после запятой, например для стейкинга 2000 FORT нужно указать 2000000000000000000000*

5. Жмём Write

6. Переходим на страницу стейкинг контракта в секцию [Write as Proxy](https://polygonscan.com/address/0xd2863157539b1D11F39ce23fC4834B62082F6874#writeProxyContract)

7. Нажимаем `Connect to web3` и подключаем Metamask

8. В секции `1. deposit` вводим следующие значения:

subjectType: 0

subject: адрес вашего сканера (не адрес владельца)

stakeValue: количество которое хотите застейкать, если 500 FORT то значение будет 500000000000000000000

![image](https://user-images.githubusercontent.com/23663132/179518638-969cd82d-789b-4f5e-8d49-a936b214d542.png)

9. Жмём Write и подтверждаем транзакцию в кошельке.

# Готово

**Теперь можно посмотреть статус сканера**

```bash
forta status
```

Если всё работает, статус контейнеров будет отображаться как ОК

![image](https://user-images.githubusercontent.com/23663132/179520050-319e4d7b-f760-4471-8fc1-89c91db5aab4.png)

# Полезные команды

Посмотреть адрес сканера

```bash
forta account address
```

Проверить статус ноды

```bash
forta status
```

Посмотреть сколько ботов использует ноду

```bash
docker ps | grep docker-entrypoint | wc -l
```

Проверить логи

```bash
journalctl -u forta -f -o cat
```

Остановить ноду

```bash
systemctl stop forta
```

Запустить ноду

```bash
systemctl start forta
```

Проверить качество работы

```bash
curl https://api.forta.network/stats/sla/scanner/<FORTA_SCANNER_ADDRESS>?startTime=2022-04-24T00:00:00Z | jq .statistics.avg
```

100% reward: SLA ≥ 0.9
80% reward: SLA ≥ 0.75
No reward: SLA < 0.75
