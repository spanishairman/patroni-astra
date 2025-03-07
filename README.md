#### Отказоустойчивый кластер PostgreSQL

Построение кластера высокой доступности для _СУБД_ на базе __PostgreSQL__, __Patroni__, __Etcd__, __Haproxy__ под управлением операционной системы
специального назначения __Astra Linux 1.7__.

##### Общее описание стенда

Схема:

![Schema](/images/PSQLCluster.drawio.png)

Список виртуальных хостов, участвующих в кластере, с описанием их функциональных ролей:

| Hostname  | IP-address | Description | OS |
| --- | --- | --- | --- |
| etcd1server  | 192.168.1.1  | Служба распределенного хранения пар ключ-значение (конфигураций) _Etcd_, 1 нода | Astra Linux 1.7 |
| etcd2server  | 192.168.1.2  | Служба распределенного хранения пар ключ-значение (конфигураций) _Etcd_, 2 нода | Astra Linux 1.7 |
| psql1server  | 192.168.1.3  | Сервер баз данных _PostgreSQL_, служба управления кластером _Patroni_, 1 нода  | Astra Linux 1.7  |
| psql2server  | 192.168.1.4  | Сервер баз данных _PostgreSQL_, служба управления кластером _Patroni_, 2 нода  | Astra Linux 1.7  |
| proxy1server  | 192.168.1.5  | _Haproxy_ - единая точка подключения к кластеру СУБД со стороны клиентского приложения, 1 нода  | Astra Linux 1.7  |
| proxy2server  | 192.168.1.6  | _Haproxy_ - единая точка подключения к кластеру СУБД со стороны клиентского приложения, 2 нода  | Astra Linux 1.7  |
| web1server  | 192.168.1.7  | Клиентское приложение для работы с базами данных, в нашем случае, это служба _Ejabberd_  | Debian Bookworm  |

Стандартное описание виртуальной машины в ![Vagrantfile](/files/Vagrantfile) на примере _etcd1server_:

<details>
<summary>Vagrantfile code</summary>

```
Vagrant.configure("2") do |config|
  config.vm.define "Astra17-etcd1" do |etcd1server|
  etcd1server.vm.box = "/home/max/vagrant/images/astralinux17g"
  etcd1server.vm.network :private_network,
       :type => 'ip',
       :libvirt__forward_mode => 'veryisolated',
       :libvirt__dhcp_enabled => false,
       :ip => '192.168.1.1',
       :libvirt__netmask => '255.255.255.240',
       :libvirt__network_name => 'vagrant-libvirt-inet1',
       :libvirt__always_destroy => false
  etcd1server.vm.provider "libvirt" do |lvirt|
      lvirt.memory = "1024"
      lvirt.cpus = "1"
      lvirt.title = "Astra17-etcd1Server"
      lvirt.description = "Виртуальная машина на базе дистрибутива Astra Linux. etcd1Server"
      lvirt.management_network_name = "vagrant-libvirt-mgmt"
      lvirt.management_network_address = "192.168.121.0/24"
      lvirt.management_network_keep = "true"
      lvirt.management_network_mac = "52:54:00:27:28:83"
  end
  etcd1server.vm.provision "shell", inline: <<-SHELL
      brd='*************************************************************'
      echo "$brd"
      echo 'Set Hostname'
      hostnamectl set-hostname etcd1server
      echo "$brd"
      sed -i 's/astra17/etcd1server/' /etc/hosts
      sed -i 's/astra17/etcd1server/' /etc/hosts
      echo '192.168.1.1 etcd1server.local etcd1server' >> /etc/hosts
      echo '192.168.1.2 etcd2server.local etcd2server' >> /etc/hosts
      echo '192.168.1.3 psql1server.local psql1server' >> /etc/hosts
      echo '192.168.1.4 psql2server.local psql2server' >> /etc/hosts
      echo '192.168.1.5 proxy1server.local proxy1server' >> /etc/hosts
      echo '192.168.1.6 proxy2server.local proxy2server' >> /etc/hosts
      echo '192.168.1.7 web1server.local web1server' >> /etc/hosts
      echo "$brd"
      echo 'Если ранее не были установлены, то установим необходимые  пакеты'
      echo "$brd"
      mount -o loop /home/vagrant/installation-1.7.5.9-16.10.23_16.58.iso /media/localiso/
      export DEBIAN_FRONTEND=noninteractive
      apt update
      SHELL
  end
```
</details>

> [!NOTE]
> Все дальнейшие действия по разворачиванию кластера выполняются с помощью плейбуков __Ansible__, соответствующие описания выполняемых действий также прилагаются.

##### Настройка __iptables__

Любая информационная система подвержена атакам и может содержать уязвимости в системном и прикладном ПО. Даже при наличии межсетевого экрана на периметре,
использование фильтрации пакетов на уровне хоста не бывает лишним.

Зададим минимальный набор правил для каждого узла кластера:

```
        iptables -P INPUT DROP
        iptables -P OUTPUT DROP
        iptables -P FORWARD DROP
        iptables -A INPUT -i lo -j ACCEPT
        iptables -A INPUT ! -i lo -d 127.0.0.0/8 -j REJECT
        iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
        iptables -A OUTPUT -j ACCEPT
        iptables -A INPUT -s 192.168.121.1 -m tcp -p tcp --dport 22 -j ACCEPT
        iptables -A INPUT -s 192.168.1.0/28 -m tcp -p tcp -j ACCEPT
        iptables -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
```

Для серверов __Haproxy__ дополнительно откроем входящие подключения по протоколу [VRRP](https://ru.wikipedia.org/wiki/VRRP):

```
        iptables -A INPUT -p vrrp -d 224.0.0.18 -j ACCEPT
```

А для web-сервера входящие подключения по протоколам http/https:

```
        iptables -A INPUT -s 192.168.121.1 -m multiport -m tcp -p tcp --dports 22,5280 -j ACCEPT
```

Плейбук, для соответствующих настроек межсетевого экранирования прилагается ![здесь](/files/play/psqla/1.iptables.yml)

##### Установка __PostgreSQL__ на серверы _psql1server_, _psql2server_

Для реализации условий по реализации кластера, на серверах группы _psqlserver_ потребуется выполнить следующие действия:
  - [x] Создание системных пользователей:
    - _ejabberd_ - учётная запись, с которой клиентское приложение подключается к кластеру;
    - _replica_role_ - учётная запись для выполнения репликации между нодами кластера;
    - _super_role_ - - учётная запись, с помощью которой _Patroni_ подключается и управляет экземплярами СУБД;
  - [x] Установка пакетов "postgresql", "python3-psycopg2", "acl";
  - [x] Настройка _postgresql.conf_;
  - [x] Настройка MAC (Mandatory Access Control);
  - [x] Настройка _pg_hba_ для управления разрешениями для внешних и локальных подключений к кластеру;
  - [x] Создание ролей для репликации и управления кластером;
  - [x] Настройка и запуск первончальной репликации между узлами кластера без участия _Patroni_;
  - [x] Создание базы данных и роли для клиентского приложения.

> [!TIP]
> Создание системных пользователей, соответствующих ролям в _СУБД_ и назначение им уровней доступа в _MAC_ требуется только в _Astra Linux Special Edition_.
> В случае использования любого другого дистрибутива _Linux_, подобных действий выполнять не потребуется.

Соответствующий ![плейбук](/files/play/psqla/2.psql.yml)

##### _Etcd_ - установка, настройка на серверах группы _etcdserver_

Для установки _Etcd_ в _Astra Linux SE 1.7_ необходимо раскомментировать дополнительные репозитории программного обеспечения, это:

```
deb https://download.astralinux.ru/astra/stable/1.7_x86-64/repository-base/ 1.7_x86-64 main contrib non-free
deb https://download.astralinux.ru/astra/stable/1.7_x86-64/repository-extended/ 1.7_x86-64 main contrib non-free
```
Далее обновить кеш пакетов и установить требуемые.

После установки, добавить или изменить следующие параметры:

```
ETCD_LISTEN_PEER_URLS
ETCD_LISTEN_CLIENT_URLS
ETCD_NAME
ETCD_HEARTBEAT_INTERVAL
ETCD_ELECTION_TIMEOUT
ETCD_ENABLE_V2
ETCD_INITIAL_ADVERTISE_PEER_URLS
ETCD_ADVERTISE_CLIENT_URLS
ETCD_INITIAL_CLUSTER_TOKEN
ETCD_INITIAL_CLUSTER_TOKEN
ETCD_INITIAL_CLUSTER
ETCD_INITIAL_CLUSTER_STATE
```
Запустить службы и собрать кластер.

![Плейбук](/files/play/psqla/3.etcd.yml)
