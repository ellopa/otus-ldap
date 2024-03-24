## Vagrant-стенд c LDAP на базе FreeIPA

**Цель домашнего задания**
- Научиться настраивать LDAP-сервер и подключать к нему LDAP-клиентов

**Описание домашнего задания**
- Установить FreeIPA
- Написать Ansible-playbook для конфигурации клиента

**Дополнительное задание**
- *Настроить аутентификацию по SSH-ключам
- **Firewall должен быть включен на сервере и на клиенте

Критерии оценивания
Статус «Принято» ставится при выполнении следующих условий:
1. Сcылка на репозиторий GitHub.
2. Vagrantfile, который будет разворачивать виртуальные машины
3. Документация по каждому заданию:
  Создайте файл README.md и снабдите его следующей информацией:
  - название выполняемого задания;
  - текст задания;
  - описание команд и их вывод;
  - особенности проектирования и реализации решения, 
  - заметки, если считаете, что имеет смысл их зафиксировать в репозитории.
  
### Введение
**LDAP (Lightweight Directory Access Protocol** — легковесный протокол доступа к каталогам) —  это протокол для хранения и получения данных из каталога с иерархической структурой.
LDAP не является протоколом аутентификации или авторизации 

С увеличением числа серверов затрудняется управление пользователями на этих сервере. LDAP решает задачу централизованного управления доступом. 
С помощью LDAP можно синхронизировать:
 -UID пользователей
 -Группы (GID)
 -Домашние каталоги
 -Общие настройки для хостов 
И т. д. 

**LDAP работает на следующих портах:**
- **389/TCP — без TLS/SSL**
- **636/TCP — с TLS/SSL**

<img src="https://github.com/ellopa/otus-ldap/blob/main/scr_ldap.png" width=100% height=100%>

### Основные компоненты LDAP

**Атрибуты — пара «ключ-значение».**
Пример атрибута: mail: admin@example.com
Записи (entry) — набор атрибутов под именем, используемый для описания чего-либо

Пример записи:
dn: sn=Ivanov, ou=people, dc=digitalocean,dc=com
objectclass: person
sn: Ivanov
cn: Ivan Ivanov

**Data Information Tree (DIT) — организационная структура**, где каждая запись имеет ровно одну родительскую запись и под ней может находиться любое количество дочерних записей. Запись верхнего уровня — исключение
На основе LDAP построенно много решений, например: Microsoft Active Directory, OpenLDAP, FreeIPA и т. д.

В данной лабораторной работе будет рассмотрена установка и настройка FreeIPA. 
**FreeIPA — это готовое решение, включающее в себе:**
  - **Сервер LDAP на базе Novell 389 DS c предустановленными схемами**
  - **Сервер Kerberos**
  - **Предустановленный BIND с хранилищем зон в LDAP**
  - **Web-консоль управления**

**Функциоанальные и нефункциональные требования**
  - ПК на Unix c 8ГБ ОЗУ или виртуальная машина с включенной Nested Virtualization.
  - Созданный аккаунт на GitHub - https://github.com/ 
  - Если Вы находитесь в России, для корректной работы Вам может потребоваться VPN.

**Предварительно установленное и настроенное следующее ПО:**
  - Hashicorp Vagrant (https://www.vagrantup.com/downloads) 
  - Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads). 
  - Любой редактор кода, например Visual Studio Code, Atom и т.д.
 
- После создания [Vagrantfile](Vagrantfile_1), запустим виртуальные машины командой vagrant up. Будут созданы 3 виртуальных машины с ОС CentOS 8 Stream. Каждая ВМ будет иметь по 2ГБ ОЗУ и по одному ядру CPU. 

### 1. Установка FreeIPA сервера

- Настройка FreeIPA-сервер. Подключимся к нему по SSH с помощью команды: vagrant ssh ipa.otus.lan и перейдём в root-пользователя: sudo -i 
- Установим часовой пояс: 
```
timedatectl set-timezone Europe/Moscow
```
```
[vagrant@ipa ~]$ sudo -i
[root@ipa ~]# timedatectl set-timezone Europe/Moscow
```

- Установим утилиту chrony: yum install -y chrony
- Запустим chrony и добавим его в автозагрузку 
```
systemctl enable chronyd
systemctl start chronyd
```
- В CentOS по умолчанию уже есть NTP-клиент Chrony. Обычно он всегда включен и добавлен в автозагрузку. Проверить работу службы можно командой: systemctl status chronyd
```
[root@ipa ~]# systemctl status chronyd     
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2024-03-21 12:41:05 MSK; 1h 36min ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
  Process: 391 ExecStartPost=/usr/libexec/chrony-helper update-daemon (code=exited, status=0/SUCCESS)
  Process: 336 ExecStart=/usr/sbin/chronyd $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 357 (chronyd)
   CGroup: /system.slice/chronyd.service
           └─357 /usr/sbin/chronyd

Mar 21 12:41:14 localhost.localdomain chronyd[357]: Selected source 213.33.238.106
Mar 21 12:41:14 localhost.localdomain chronyd[357]: System clock wrong by 1.945410 seconds, adjustme...ted
Mar 21 12:41:16 localhost.localdomain chronyd[357]: System clock was stepped by 1.945410 seconds
Mar 21 12:41:18 localhost.localdomain chronyd[357]: Can't synchronise: no selectable sources
Mar 21 12:41:18 localhost.localdomain chronyd[357]: Selected source 213.33.238.106
Mar 21 12:41:22 ipa.otus.lan chronyd[357]: Selected source 94.247.111.10
Mar 21 12:41:28 ipa.otus.lan chronyd[357]: Source 213.33.238.106 replaced with 82.142.168.18
Mar 21 12:41:34 ipa.otus.lan chronyd[357]: Selected source 162.159.200.123
Mar 21 12:42:29 ipa.otus.lan chronyd[357]: Selected source 94.247.111.10
Mar 21 12:43:33 ipa.otus.lan chronyd[357]: Selected source 162.159.200.123
Hint: Some lines were ellipsized, use -l to show in full.
```

>- Если требуется, поменяем имя нашего сервера: hostnamectl set-hostname <имя сервера>

- Выключим Firewall: systemctl stop firewalld
- Отключим автозапуск Firewalld: systemctl disable firewalld
- Остановим Selinux: setenforce 0
```
[root@ipa ~]# systemctl stop firewalld
[root@ipa ~]# systemctl disable firewalld
[root@ipa ~]# setenforce 0
```
- Поменяем в файле /etc/selinux/config, параметр Selinux на disabled - vi /etc/selinux/config
```
[root@ipa ~]# vi /etc/selinux/config
[root@ipa ~]# cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```
- **Для дальнейшей настройки FreeIPA нам потребуется, чтобы DNS-сервер хранил запись о нашем LDAP-сервере. В рамках данной лабораторной работы мы не будем настраивать отдельный DNS-сервер и просто добавим запись в файл /etc/hosts**

```
[root@ipa ~]# vi /etc/hosts
[root@ipa ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain
::1         localhost localhost.localdomain
192.168.57.10 ipa.otus.lan ipa
```
$Установим модуль DL1: yum install -y @idm:DL1
- Установим FreeIPA-сервер: yum install -y ipa-server
- yum update nss
```
[root@ipa ~]# yum install -y ipa-server
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.docker.ru
 * extras: mirror.docker.ru
 * updates: mirror.yandex.ru
Resolving Dependencies

.....

Dependency Updated:
  cyrus-sasl-lib.x86_64 0:2.1.26-24.el7_9                krb5-libs.x86_64 0:1.15.1-55.el7_9               
  libwbclient.x86_64 0:4.10.16-25.el7_9                  openldap.x86_64 0:2.4.44-25.el7_9                
  samba-client-libs.x86_64 0:4.10.16-25.el7_9            samba-common.noarch 0:4.10.16-25.el7_9           
  samba-common-libs.x86_64 0:4.10.16-25.el7_9            samba-libs.x86_64 0:4.10.16-25.el7_9             
  systemd.x86_64 0:219-78.el7_9.9                        systemd-libs.x86_64 0:219-78.el7_9.9             
  systemd-sysv.x86_64 0:219-78.el7_9.9                  

Complete!
```
- Запустим скрипт установки: ipa-server-install. Далее, нам потребуется указать параметры нашего LDAP-сервера, после ввода каждого параметра нажимаем Enter, если нас устраивает параметр, указанный в квадратных скобках, то можно сразу нажимать Enter:
```
Do you want to configure integrated DNS (BIND)? [no]: no
Server host name [ipa.otus.lan]: <Нажимем Enter>
Please confirm the domain name [otus.lan]: <Нажимем Enter>
Please provide a realm name [OTUS.LAN]: <Нажимем Enter>
Directory Manager password: <Указываем пароль минимум 8 символов> lkjH78nm
Password (confirm): <Дублируем указанный пароль> lkjH78nm
IPA admin password: <Указываем пароль минимум 8 символов> lkjH78nm
Password (confirm): <Дублируем указанный пароль>
NetBIOS domain name [OTUS]: <Нажимем Enter>
Do you want to configure chrony with NTP server or pool address? [no]: no
The IPA Master Server will be configured with:
Hostname:       ipa.otus.lan
IP address(es): 192.168.57.10
Domain name:    otus.lan
Realm name:     OTUS.LAN

The CA will be configured with:
Subject DN:   CN=Certificate Authority,O=OTUS.LAN
Subject base: O=OTUS.LAN
Chaining:     self-signed
Проверяем параметры, если всё устраивает, то нажимаем yes
Continue to configure the system with these values? [no]: yes
```
- Далее начнётся процесс установки. Процесс установки занимает примерно 10-15 минут (иногда время может быть другим). Если мастер успешно выполнит настройку FreeIPA то в конце мы получим сообщение: The ipa-server-install command was successful

- **При вводе параметров установки мы вводили 2 пароля:**
- **Directory Manager password** — это пароль администратора сервера каталогов, У этого пользователя есть полный доступ к каталогу.
- **IPA admin password** — пароль от пользователя FreeIPA admin

- После успешной установки FreeIPA, проверим, что сервер Kerberos может выдать нам билет: kinit admin
```
[vagrant@ipa ~]$ sudo -i
[root@ipa ~]# kinit admin
Password for admin@OTUS.LAN: 
[root@ipa ~]# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: admin@OTUS.LAN

Valid starting     Expires            Service principal
03/22/24 08:34:09  03/23/24 08:33:54  krbtgt/OTUS.LAN@OTUS.LAN
```
>- Для удаление полученного билета воспользуемся командой: kdestroy

- Мы можем зайти в Web-интерфейс нашего FreeIPA-сервера, для этого на нашей хостой машине нужно прописать следующую строку в файле Hosts:
192.168.57.10 ipa.otus.lan
>- В Unix-based системах файл хост находится по адресу /etc/hosts, в Windows — c:\Windows\System32\Drivers\etc\hosts. Для добавления строки потребуются права администратора.

- После добавления DNS-записи откроем c нашей хост-машины веб-страницу vagranvagra http://ipa.otus.lan

<img src="https://github.com/ellopa/otus-ldap/blob/main/scr_ldap_1.png" width=80% height=80%>
- Откроется окно управления FreeIPA-сервером. В имени пользователя укажем admin, в пароле укажем наш IPA admin password и нажмём войти. 

<img src="https://github.com/ellopa/otus-ldap/blob/main/scr_ldap_2.png" width=80% height=80%>
- Откроется веб-консоль упрвления FreeIPA. Данные во FreeIPA можно вносить как через веб-консоль, так и средствами коммандной строки.

На этом установка и настройка FreeIPA-сервера завершена.

### 2. Ansible playbook для конфигурации клиента

Настройка клиента похожа на настройку сервера. На хосте также нужно:
 - Настроить синхронизацию времени и часовой пояс
 - Настроить (или выключить) firewall
 - Настроить (или выключить) SElinux
 - В файле hosts должна быть указана запись с FreeIPA-сервером и хостом

>- Хостов, которые требуется добавить к серверу может быть много, для упращения нашей работы выполним настройки с помощью Ansible:
- В каталоге с нашей лабораторной работой создадим каталог Ansible: mkdir ansible
- В каталоге ansible создадим файл hosts со следующими параметрами:
```
[clients]
client1.otus.lan ansible_host=192.168.57.11 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/client1.otus.lan/virtualbox/private_key
client2.otus.lan ansible_host=192.168.57.12 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/client2.otus.lan/virtualbox/private_key
```
>- Файл содержит группу clients в которой прописаны 2 хоста: 
>- client1.otus.lan
>- client2.otus.lan
>- Также указаны и ip-адреса, имя пользователя от которого будет логин и ssh-ключ.

- Далее создадим файл [](/provision_1.yml) в котором непосредственно будет выполняться настройка клиентов: 

- Template файла /etc/hosts выглядит следующим образом:
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.57.10 ipa.otus.lan ipa
```
>- Почти все модули нам уже знакомы, давайте подробнее остановимся на последней команде echo -e "yes\nyes" | ipa-client-install --mkhomedir --domain=OTUS.LAN --server=ipa.otus.lan --no-ntp -p admin -w otus2022

>- При добавлении хоста к домену мы можем просто ввести команду ipa-client-install и следовать мастеру подключения к FreeIPA-серверу (как было в первом пункте). Однако команда позволяет нам сразу задать требуемые нам параметры:
```
--domain — имя домена
--server — имя FreeIPA-сервера
--no-ntp — не настраивать дополнительно ntp (мы уже настроили chrony)
-p — имя админа домена
-w — пароль администратора домена (IPA password)
--mkhomedir — создать директории пользователей при их первом логине
```
>- Если мы сразу укажем все параметры, то можем добавить эту команду в Ansible и автоматизировать процесс добавления хостов в домен. 
>- Альтернативным вариантом мы можем найти на GitHub отдельные модули по подключениею хостов к FreeIPA-сервер. 

- Запустить 
```
ansible-playbook provision.yml
```
- После подключения хостов к FreeIPA-сервер нужно проверить, что мы можем получить билет от Kerberos сервера: kinit admin
- Если подключение выполнено правильно, то мы сможем получить билет, после ввода пароля. 

```
[vagrant@client1 ~]$ kinit admin
Password for admin@OTUS.LAN: 
[vagrant@client1 ~]$ klist
Ticket cache: KEYRING:persistent:1000:1000
Default principal: admin@OTUS.LAN

Valid starting     Expires            Service principal
03/24/24 06:14:45  03/25/24 06:14:10  krbtgt/OTUS.LAN@OTUS.LAN
```
```
[vagrant@client2 ~]$ kinit admin
Password for admin@OTUS.LAN: 
[vagrant@client2 ~]$ klist
Ticket cache: KEYRING:persistent:1000:1000
Default principal: admin@OTUS.LAN

Valid starting     Expires            Service principal
03/24/24 06:16:11  03/25/24 06:16:05  krbtgt/OTUS.LAN@OTUS.LAN
```
>- Для удаление полученного билета команда: kdestroy

- Создание пользователя. Авторизируемся на сервере: kinit admin и создадим пользователя otus-user 

```
ipa user-add otus-user --first=otus --last=user --password
```
```
[vagrant@ipa ~]$ ipa user-find otus-user
--------------
1 user matched
--------------
  User login: otus-user
  First name: otus
  Last name: user
  Home directory: /home/otus-user
  Login shell: /bin/sh
  Principal name: otus-user@OTUS.LAN
  Principal alias: otus-user@OTUS.LAN
  Email address: otus-user@otus.lan
  UID: 1904400001
  GID: 1904400001
  Account disabled: False
----------------------------
Number of entries returned 1
----------------------------
```

### 3. Ansible playbook для конфигурации сервера и клиента

- Аутентификацию по SSH-ключам
- Firewall включен на сервере и на клиенте
- [playbook](provision.yml)
- Развертывание из [Vagrantfile](Vagrantfile):
```
vagant up --no-provision
vagrant provision ipaserver
```
- Проверка подключения созданного пользователя test_user:
```
[vagrant@client3 ~]$ ssh test_user@localhost
Creating home directory for test_user.
```
```
[vagrant@client3 ~]$ ssh test_user@localhost
Last login: Sun Mar 24 11:07:02 2024 from ::1
[test_user@client3 ~]$ exit
logout
Connection to localhost closed.
```
- Создание  пользователя и проверка - попробуем залогиниться на клиент. Система запросит пароль и попросит ввести новый пароль. 
scr_ldap_3.png

```
elena_leb@ubuntunbleb:~/LDAP_DZ/ansible$ vagrant ssh ipaclient
Last login: Sun Mar 24 11:06:17 2024 from 10.0.2.2
[vagrant@client3 ~]$ kinit otus-user
Password for otus-user@OTUS.LAN: 
Password expired.  You must change it now.
Enter new password: 
Enter it again: 
[vagrant@client3 ~]$ kinit otus-user
Password for otus-user@OTUS.LAN: 
```
```
[vagrant@client3 ~]$ ssh otus-user@localhost
Password: 
Creating home directory for otus-user.
-sh-4.2$ 
```

>**Рекомендуемые источники**
> -Статья о LDAP - https://ru.wikipedia.org/wiki/LDAP
> -Статья о настройке FreeIPA - https://www.dmosk.ru/miniinstruktions.php?mini=freeipa-centos
> -FreeIPA wiki - https://www.freeipa.org/page/Wiki_TODO
> -Статья «Chapter 13. Preparing the system for IdM client installation» - https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/installing_identity_management/preparing-the-system-for-ipa-client-installation_installing-identity-management
> -Статья «про LDAP по-русски» - https://pro-ldap.ru/ 
> -Статья «54. Настройка времени» - https://basis.gnulinux.pro/ru/latest/basis 54/54._%D0%9D%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B9%D0%BA%D0%B0_%D0%B2%D1%80%D0%B5%D0%BC%D0%B5%D0%BD%D0%B8.html



