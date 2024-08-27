# Подключение ВПН на Ubuntu

*Для подключения потребуется сервер с Линуксом и куча желания*

**1. Обновить все в своей системе. И установить wireguard.**

```bash
sudo apt update
sudo apt install wireguard -y
```

**2. После установки wireguard переходим в папку**
```bash
cd /etc/wireguard
```
**3. Необходимо сгенерировать ключи для сервера**
```bash
sudo wg genkey | sudo tee server_private.key | sudo wg pubkey | sudo tee server_public.key
```
>*После выполнения этой команды у вас в этой директории будут два файла*
>
>*__server_private.key__: приватный ключ сервера.*
>
>*__server_public.key__: публичный ключ сервера.*


___

## Создание конфигурационного файла для сервера

**1. Файл создаем командой**

```bash
sudo nano /etc/wireguard/wg0.conf
```
>*У вас откроется своеобразный редактор. В него вносим соответствующий код*

```ini
[Interface]
PrivateKey = <приватный ключ сервера>
Address = 10.0.0.1/24 # внутренний IP-адрес VPN
ListenPort = 51820 # порт для прослушивания

# Опционально: включение пересылки трафика
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

```

**2. Теперь необходимо создать ключи для клиента.**

*Думаю, что можно создать сразу несколько ключей для клиентов, все зависит от мощностей сервера*

```bash
wg genkey | tee client_private.key | wg pubkey | tee client_public.key
```

> *После этой команды у вас появится еще два файла, но уже для клиентской стороны*
>
>*__client_private.key__: приватный ключ сервера.*
>
>*__client_public.key__: публичный ключ сервера.*

*__Теперь необходимо добавить данные в конфигурационный файл wg0.info клиентский peer__*

```ini
[Peer]
PublicKey = <публичный ключ клиента>
AllowedIPs = 10.0.0.2/32 # IP-адрес клиента в VPN
```

Сохраняем `-> Ctrl+O, Enter, Ctrl+X.`

**На этом настройка конфигурационного файла завершена. Не забудьте только подставить свои данные ключей!**

## Включение IP пересылки

**1. Для работы NAT (если требуется пересылка трафика) необходимо включить пересылку пакетов:**

```bash
sudo nano /etc/sysctl.conf
```

Найдите строку `#net.ipv4.ip_forward=1` и раскомментируйте её, удалив #. 

**2. Затем примените изменения:**
```bash
sudo sysctl -p
```

## Настройка клиента

**1. Создаем файл конфигурации для клиента client1.conf:**
```bash
sudo nano client1.conf
```
**2. Вносим туда данные**

```ini
[Interface]
PrivateKey = <приватный ключ клиента>
Address = 10.0.0.2/24 # IP-адрес клиента в VPN
DNS = 8.8.8.8  # Если требуется

[Peer]
PublicKey = <публичный ключ сервера>
Endpoint = <IP сервера>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```
*Замените <приватный ключ клиента> и <публичный ключ сервера> на соответствующие значения.*

Сохраняем `-> Ctrl+O, Enter, Ctrl+X.`

**Эти данные для клиента можно преобразовать в QR-code и отправить ему.**

**3. Внесение NAT**

*Чтобы правильно настроить NAT, выполните следующие команды*

```bash
sudo iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
```

*Проверьте, что правило было добавлено*

```bash
sudo iptables -t nat -L -n -v
```

**В результате должно появиться правило MASQUERADE для интерфейса ens3.**
___
___
___

# Запуск сервера WireGuard

**1. Запустите интерфейс WireGuard с помощью команды `wg-quick`**

```bash
sudo wg-quick up wg0
```

*Это запустит интерфейс wg0, используя конфигурацию из файла `/etc/wireguard/wg0.conf`*

**2. Проверка работы сервера**

*Чтобы убедиться, что WireGuard работает, вы можете использовать команду*
```bash
sudo wg
```
*Эта команда покажет статус интерфейса wg0, активных клиентов и другую полезную информацию.*

# Настройка брандмауэра (если требуется)

*Если на сервере используется брандмауэр, убедитесь, что порт 51820 (или другой, указанный вами в конфигурации) открыт для входящих соединений:*

Для UFW:

```bash
sudo ufw allow 51820/udp
```
Для iptables:

```bash
sudo iptables -A INPUT -p udp -m udp --dport 51820 -j ACCEPT
```

# Проверка и устранение неполадок

**1. Проверка конфигурации iptables**

*Проверьте, что правила iptables правильно применены*

```bash
sudo iptables -t nat -L -n -v
```
**Вы должны увидеть правило MASQUERADE для интерфейса eth0:**

```plaintext
Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
 1234  5678 MASQUERADE  all  --  *      eth0    10.0.0.0/24           0.0.0.0/0
 ```

**Если правила не отображаются, попробуйте их добавить снова:**


```bash
sudo iptables -A FORWARD -i wg0 -j ACCEPT
sudo iptables -A FORWARD -o wg0 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```


**2. Проверка IP переадресации**

*Убедитесь, что IP переадресация включена на сервере*

```bash
sudo sysctl net.ipv4.ip_forward
```

*Если вывод показывает net.ipv4.ip_forward = 0, включите его*

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

*Или, чтобы сделать это изменение постоянным*

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

**3. Перезапуск интерфейсов**

*Попробуйте перезапустить интерфейсы на сервере и клиенте:*

```bash
sudo wg-quick down wg0
sudo wg-quick up wg0
```



