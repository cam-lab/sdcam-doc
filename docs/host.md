# Настройка хоста

## Сетевой интерфейс 

Для работы программы помимо установленных библиотек также требуется наличие сетевого сокета, через который осуществляется обмен данными с прибором. Для этих целей целесообразно иметь выделенный сетевой интерфейс с поддержкой скоростей передачи от 1Gbps и выше. Такой интерфейс удобно настроить со статическим сетевым адресом, дабы избежать сложностей и накладных расходов на поддержку DHCP в целевом приборе.

Создание интерфейса на дистрибутивах **Ubuntu** можно выполнить с помощью команды:

```shell
nmcli con add type ethernet con-name 'camif' ifname enp11s0 ipv4.method manual ipv4.addresses 192.168.10.1/24 ipv4.gateway 192.168.10.255
```

Запуск интерфейса:

```shell
nmcli con up id 'camif'

```

## Размеры сетевых буферов

При работе на высокой скорости размер сетевых буферов ОС оказывает критическое влияние на производительнлость&nbsp;– при их недостаточном размере часть пакетов теряется. Устранить эту проблему можно путём увеличения размеров этих буферов. На ОС сеймейства Ubuntu это делается с помощью следующих команд:

```shell
sudo sysctl -w net.core.wmem_default=26214400
sudo sysctl -w net.core.rmem_default=26214400
sudo sysctl -w net.core.wmem_max=26214400
sudo sysctl -w net.core.rmem_max=26214400
```

Здесь размер буферов увеличен до 25 мегабайт. Как правило, этого достаточно.

Указанные команды устанавливают значения только на текущую сессию. Для того, чтобы установки применялись автоматически при каждом запуске ОС, необходимо их внести в файл `/etc/sysctl.conf`:

```
net.core.wmem_default=26214400
net.core.rmem_default=26214400
net.core.wmem_max=26214400
net.core.rmem_max=26214400
```