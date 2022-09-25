<h3>### VPN ###</h3>

<h4>Описание домашнего задания</h4>

<p>Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников</p>

<ul>
<li>Между двумя виртуалками поднять vpn в режимах<br />
● tun;<br />
● tap;<br />
Прочувствовать разницу.</li>
<li>Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на виртуалку.</li>
<li>*Самостоятельно изучить, поднять ocserv и подключиться с хоста к виртуалке<br />
Формат сдачи ДЗ - vagrant + ansible</li>
</ul>

<h4>1. TUN/TAP режимы VPN</h4>

<p>В домашней директории создадим директорию vpn, в котором будут храниться настройки виртуальных машин:</p>

<pre>[user@localhost otus]$ mkdir ./vpn
[user@localhost otus]$</pre>

<p>Перейдём в директорию vpn:</p>

<pre>[user@localhost otus]$ cd ./vpn/
[user@localhost vpn]$</pre>

<p>Создадим файл Vagrantfile:</p>

<pre>[user@localhost vpn]$ vi ./Vagrantfile</pre>

<p>1. Заполним следующим содержимым:</p>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  config.vm.define "server" do |server|
    server.vm.hostname = "server.loc"
    server.vm.network "private_network", ip: "192.168.10.10"
  end
  config.vm.define "client" do |client|
    client.vm.hostname = "client.loc"
    client.vm.network "private_network", ip: "192.168.10.20"
  end
end</pre>

<p>Запустим эти виртуальные машины:</p>

<pre>[user@localhost vpn]$ vagrant up</pre>

<p>Проверим состояние созданной и запущенной машины:</p>

<pre>[user@localhost vpn]$ vagrant status
Current machine states:

server                    running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost vpn]$</pre>

<p>2. Заходим на ВМ server и зайдём под пользователем root:</p>

<pre>[user@localhost vpn]$ vagrant ssh server
[vagrant@server ~]$ sudo -i
[root@server ~]# </pre>

<p>Выполняем следующие действия:<br />
● устанавливаем epel репозиторий:</p>

<pre>[root@server ~]# yum install -y epel-release</pre>

<p>● устанавливаем пакет openvpn, easy-rsa и iperf3</p>

<pre>[root@server ~]# yum install -y openvpn iperf3</pre>

<p>● Отключаем SELinux (при желании можно написать правило для openvpn) setenforce 0 (работает до ребута)</p>

<pre>[root@server ~]# setenforce 0
[root@server ~]#</pre>

<p>3. Настройка openvpn сервера:<br />
● создаём файл-ключ</p>

<pre>[root@server ~]# openvpn --genkey --secret /etc/openvpn/static.key
[root@server ~]#</pre>

<p>● создаём конфигурационный файл vpn-сервера</p>

<pre>vi /etc/openvpn/server.conf</pre>

<pre>dev tap
ifconfig 10.10.10.1 255.255.255.0
topology subnet
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3</pre>

<p>● Запускаем openvpn сервер и добавляем в автозагрузку:</p>

<pre>[root@server ~]# systemctl enable openvpn@server --now
Created symlink from /etc/systemd/system/multi-user.target.wants/openvpn@server.service to /usr/lib/systemd/system/openvpn@.service.
[root@server ~]#</pre>

<p>4. Настройка openvpn клиента.<br />
В отдельном терминале зайдем на ВМ client и зайдём под пользователем root:

<pre>[user@localhost vpn]$ vagrant ssh client
[vagrant@client ~]$ sudo -i
[root@client ~]#</pre>

<p>Как и ВМ server выполним следующие действия:<br />
● устанавливаем epel репозиторий:</p>

<pre>[root@client ~]# yum install -y epel-release</pre>

<p>● устанавливаем пакет openvpn, easy-rsa и iperf3</p>

<pre>[root@client ~]# yum install -y openvpn iperf3</pre>

<p>● Отключаем SELinux (при желании можно написать правило для openvpn) setenforce 0 (работает до ребута)</p>

<pre>[root@client ~]# setenforce 0
[root@client ~]#</pre>

<p>● создаём конфигурационный файл клиента:</p>

<pre>vi /etc/openvpn/client.conf</pre>

<p>● Файл будет содержать следующий конфиг:</p>

<pre>[root@client ~]# vi /etc/openvpn/client.conf</pre>

<pre>dev tap
remote 192.168.10.10
ifconfig 10.10.10.2 255.255.255.0
topology subnet
route 192.168.10.0 255.255.255.0
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3</pre>

<p>● На ВМ client в директорию /etc/openvpn/ скопируем файл-ключ static.key, который был создан на сервере.</p>

<p>● Запускаем openvpn клиент и добавляем в автозагрузку:</p>

<pre>[root@client ~]# systemctl enable openvpn@client --now
Created symlink from /etc/systemd/system/multi-user.target.wants/openvpn@client.service to /usr/lib/systemd/system/openvpn@.service.
[root@client ~]#</pre>

<p>5. Далее замерим скорость в туннеле.<br />
● на ВМ server (openvpn-сервер) запускаем iperf3 в режиме сервера:</p>

<pre>iperf3 -s &</pre>

<p>● на ВМ client (openvpn-клиент) запускаем iperf3 в режиме клиента и замеряем скорость в туннеле:</p>

<pre>iperf3 -c 10.10.10.1 -t 40 -i 5</pre>

<p>Получаем следующие результаты:
на ВМ server (openvpn-сервер):</p>

<pre>[root@server ~]# iperf3 -s &
[2] 22417
[root@server ~]# iperf3: error - unable to start listener for connections: Address already in use
iperf3: exiting
Accepted connection from 10.10.10.2, port 59492
[  5] local 10.10.10.1 port 5201 connected to 10.10.10.2 port 59494
[ ID] Interval           Transfer     Bandwidth
[  5]   0.00-1.00   sec  14.8 MBytes   124 Mbits/sec                  
[  5]   1.00-2.00   sec  15.0 MBytes   126 Mbits/sec                  
[  5]   2.00-3.01   sec  16.6 MBytes   139 Mbits/sec                  
[  5]   3.01-4.00   sec  16.7 MBytes   142 Mbits/sec                  
[  5]   4.00-5.00   sec  16.0 MBytes   134 Mbits/sec                  
[  5]   5.00-6.01   sec  16.1 MBytes   134 Mbits/sec                  
[  5]   6.01-7.00   sec  16.1 MBytes   135 Mbits/sec                  
[  5]   7.00-8.00   sec  15.5 MBytes   131 Mbits/sec                  
[  5]   8.00-9.00   sec  15.7 MBytes   132 Mbits/sec                  
[  5]   9.00-10.00  sec  16.5 MBytes   138 Mbits/sec                  
[  5]  10.00-11.00  sec  17.1 MBytes   144 Mbits/sec                  
[  5]  11.00-12.00  sec  16.4 MBytes   137 Mbits/sec                  
[  5]  12.00-13.01  sec  16.9 MBytes   141 Mbits/sec                  
[  5]  13.01-14.01  sec  15.8 MBytes   133 Mbits/sec                  
[  5]  14.01-15.00  sec  15.7 MBytes   132 Mbits/sec                  
[  5]  15.00-16.00  sec  16.4 MBytes   137 Mbits/sec                  
[  5]  16.00-17.00  sec  15.4 MBytes   129 Mbits/sec                  
[  5]  17.00-18.00  sec  16.3 MBytes   137 Mbits/sec                  
[  5]  18.00-19.00  sec  15.6 MBytes   131 Mbits/sec                  
[  5]  19.00-20.00  sec  15.9 MBytes   134 Mbits/sec                  
[  5]  20.00-21.00  sec  16.5 MBytes   138 Mbits/sec                  
[  5]  21.00-22.00  sec  14.9 MBytes   125 Mbits/sec                  
[  5]  22.00-23.01  sec  16.5 MBytes   138 Mbits/sec                  
[  5]  23.01-24.01  sec  15.5 MBytes   130 Mbits/sec                  
[  5]  24.01-25.00  sec  14.3 MBytes   120 Mbits/sec                  
[  5]  25.00-26.00  sec  13.2 MBytes   111 Mbits/sec                  
[  5]  26.00-27.00  sec  14.7 MBytes   123 Mbits/sec                  
[  5]  27.00-28.00  sec  14.7 MBytes   123 Mbits/sec                  
[  5]  28.00-29.00  sec  16.7 MBytes   140 Mbits/sec                  
[  5]  29.00-30.00  sec  15.5 MBytes   130 Mbits/sec                  
[  5]  30.00-31.00  sec  16.0 MBytes   135 Mbits/sec                  
[  5]  31.00-32.00  sec  16.4 MBytes   137 Mbits/sec                  
[  5]  32.00-33.00  sec  16.3 MBytes   136 Mbits/sec                  
[  5]  33.00-34.00  sec  16.5 MBytes   139 Mbits/sec                  
[  5]  34.00-35.00  sec  16.0 MBytes   135 Mbits/sec                  
[  5]  35.00-36.00  sec  16.8 MBytes   141 Mbits/sec                  
[  5]  36.00-37.00  sec  15.5 MBytes   130 Mbits/sec                  
[  5]  37.00-38.00  sec  15.9 MBytes   134 Mbits/sec                  
[  5]  38.00-39.00  sec  16.0 MBytes   134 Mbits/sec                  
[  5]  39.00-40.00  sec  16.3 MBytes   137 Mbits/sec                  
[  5]  40.00-40.04  sec   590 KBytes   119 Mbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth
[  5]   0.00-40.04  sec  0.00 Bytes  0.00 bits/sec                  sender
[  5]   0.00-40.04  sec   636 MBytes   133 Mbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
^C
[2]+  Exit 1                  iperf3 -s
[root@server ~]#</pre>

<p>на ВМ client (openvpn-клиент):</p>

<pre>[root@client ~]# iperf3 -c 10.10.10.1 -t 40 -i 5
Connecting to host 10.10.10.1, port 5201
[  4] local 10.10.10.2 port 59494 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-5.01   sec  81.0 MBytes   136 Mbits/sec   90    160 KBytes
[  4]   5.01-10.01  sec  79.9 MBytes   134 Mbits/sec   38    187 KBytes
[  4]  10.01-15.00  sec  81.7 MBytes   137 Mbits/sec   76    199 KBytes
[  4]  15.00-20.00  sec  79.4 MBytes   133 Mbits/sec   55    142 KBytes
[  4]  20.00-25.00  sec  77.9 MBytes   131 Mbits/sec   33    221 KBytes
[  4]  25.00-30.00  sec  74.8 MBytes   125 Mbits/sec   57    215 KBytes
[  4]  30.00-35.01  sec  81.3 MBytes   136 Mbits/sec   56    210 KBytes
[  4]  35.01-40.00  sec  80.4 MBytes   135 Mbits/sec   82    182 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-40.00  sec   636 MBytes   133 Mbits/sec  487             sender
[  4]   0.00-40.00  sec   636 MBytes   133 Mbits/sec                  receiver

iperf Done.
[root@client ~]#</pre>

<p>6. Повторяем пункты 1-5 для режима работы tun. Конфигурационные файлы сервера и клиента изменятся только в директиве dev. Делаем выводы о режимах, их достоинствах и недостатках.</p>

<p>Остановим обе машины:</p>

<pre>[root@client ~]# systemctl stop openvpn@client
[root@client ~]#</pre>

<pre>[root@server ~]# systemctl stop openvpn@server
[root@server ~]#</pre>

<p>Изменим openvpn конфиг на ВМ server:</p>

<pre>[root@server ~]# vi /etc/openvpn/server.conf
<b>dev tun</b>
ifconfig 10.10.10.1 255.255.255.0
topology subnet
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3</pre>

<p>Изменим openvpn конфиг на ВМ client:</p>

<pre>[root@client ~]# vi /etc/openvpn/client.conf
<b>dev tun</b>
remote 192.168.10.10
ifconfig 10.10.10.2 255.255.255.0
topology subnet
route 192.168.10.0 255.255.255.0
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3</pre>

<p>Запустим на обеих машинах openvpn сервисы:</p>

<pre>[root@server ~]# systemctl start openvpn@server
[root@server ~]#</pre>

<pre>[root@client ~]# systemctl start openvpn@client
[root@client ~]#</pre>

<p>Снова проведём замеры скорости в туннеле и получаем следующие результаты:</p>

<pre>[root@server ~]# iperf3 -s &
[2] 22479
[root@server ~]# iperf3: error - unable to start listener for connections: Address already in use
iperf3: exiting
Accepted connection from 10.10.10.2, port 59496
[  5] local 10.10.10.1 port 5201 connected to 10.10.10.2 port 59498
[ ID] Interval           Transfer     Bandwidth
[  5]   0.00-1.01   sec  15.0 MBytes   125 Mbits/sec                  
[  5]   1.01-2.00   sec  17.3 MBytes   146 Mbits/sec                  
[  5]   2.00-3.00   sec  17.9 MBytes   149 Mbits/sec                  
[  5]   3.00-4.01   sec  17.8 MBytes   149 Mbits/sec                  
[  5]   4.01-5.00   sec  17.8 MBytes   151 Mbits/sec                  
[  5]   5.00-6.00   sec  17.9 MBytes   149 Mbits/sec                  
[  5]   6.00-7.00   sec  17.9 MBytes   151 Mbits/sec                  
[  5]   7.00-8.00   sec  17.6 MBytes   148 Mbits/sec                  
[  5]   8.00-9.00   sec  18.0 MBytes   151 Mbits/sec                  
[  5]   9.00-10.01  sec  17.3 MBytes   144 Mbits/sec                  
[  5]  10.01-11.00  sec  17.7 MBytes   149 Mbits/sec                  
[  5]  11.00-12.00  sec  17.7 MBytes   149 Mbits/sec                  
[  5]  12.00-13.00  sec  17.8 MBytes   150 Mbits/sec                  
[  5]  13.00-14.00  sec  16.5 MBytes   138 Mbits/sec                  
[  5]  14.00-15.01  sec  17.1 MBytes   144 Mbits/sec                  
[  5]  15.01-16.00  sec  17.7 MBytes   149 Mbits/sec                  
[  5]  16.00-17.00  sec  17.8 MBytes   149 Mbits/sec                  
[  5]  17.00-18.00  sec  15.6 MBytes   131 Mbits/sec                  
[  5]  18.00-19.00  sec  17.9 MBytes   150 Mbits/sec                  
[  5]  19.00-20.00  sec  17.9 MBytes   150 Mbits/sec                  
[  5]  20.00-21.00  sec  17.6 MBytes   148 Mbits/sec                  
[  5]  21.00-22.00  sec  17.9 MBytes   150 Mbits/sec                  
[  5]  22.00-23.00  sec  17.7 MBytes   148 Mbits/sec                  
[  5]  23.00-24.01  sec  17.6 MBytes   147 Mbits/sec                  
[  5]  24.01-25.00  sec  17.9 MBytes   151 Mbits/sec                  
[  5]  25.00-26.00  sec  15.4 MBytes   130 Mbits/sec                  
[  5]  26.00-27.00  sec  16.4 MBytes   138 Mbits/sec                  
[  5]  27.00-28.00  sec  17.9 MBytes   150 Mbits/sec                  
[  5]  28.00-29.00  sec  17.2 MBytes   143 Mbits/sec                  
[  5]  29.00-30.01  sec  17.8 MBytes   149 Mbits/sec                  
[  5]  30.01-31.00  sec  17.0 MBytes   144 Mbits/sec                  
[  5]  31.00-32.00  sec  17.6 MBytes   148 Mbits/sec                  
[  5]  32.00-33.00  sec  17.1 MBytes   143 Mbits/sec                  
[  5]  33.00-34.00  sec  17.9 MBytes   150 Mbits/sec                  
[  5]  34.00-35.00  sec  17.0 MBytes   143 Mbits/sec                  
[  5]  35.00-36.00  sec  17.9 MBytes   150 Mbits/sec                  
[  5]  36.00-37.00  sec  17.3 MBytes   145 Mbits/sec                  
[  5]  37.00-38.00  sec  17.2 MBytes   144 Mbits/sec                  
[  5]  38.00-39.00  sec  18.5 MBytes   155 Mbits/sec                  
[  5]  39.00-40.00  sec  17.4 MBytes   146 Mbits/sec                  
[  5]  40.00-40.06  sec  1.05 MBytes   141 Mbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth
[  5]   0.00-40.06  sec  0.00 Bytes  0.00 bits/sec                  sender
[  5]   0.00-40.06  sec   698 MBytes   146 Mbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
^C
[2]+  Exit 1                  iperf3 -s
[root@server ~]#</pre>

<pre>[root@client ~]# iperf3 -c 10.10.10.1 -t 40 -i 5
Connecting to host 10.10.10.1, port 5201
[  4] local 10.10.10.2 port 59498 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-5.00   sec  87.0 MBytes   146 Mbits/sec   58    157 KBytes
[  4]   5.00-10.01  sec  88.3 MBytes   148 Mbits/sec   50    133 KBytes
[  4]  10.01-15.00  sec  87.1 MBytes   146 Mbits/sec   64    193 KBytes
[  4]  15.00-20.00  sec  87.2 MBytes   146 Mbits/sec   59    174 KBytes
[  4]  20.00-25.00  sec  88.5 MBytes   148 Mbits/sec   22    268 KBytes
[  4]  25.00-30.00  sec  85.0 MBytes   143 Mbits/sec   47    219 KBytes
[  4]  30.00-35.00  sec  86.5 MBytes   145 Mbits/sec   75    181 KBytes
[  4]  35.00-40.01  sec  88.6 MBytes   148 Mbits/sec   57    218 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-40.01  sec   698 MBytes   146 Mbits/sec  432             sender
[  4]   0.00-40.01  sec   698 MBytes   146 Mbits/sec                  receiver

iperf Done.
[root@client ~]#</pre>

<p>Разница tun и tap режимов:</p>

<p>TAP:</p>

<p>Преимущества:<br />
ведёт себя как настоящий сетевой адаптер (за исключением того, что он виртуальный);<br />
может осуществлять транспорт любого сетевого протокола (IPv4, IPv6, IPX и прочих);<br />
работает на 2 уровне, поэтому может передавать Ethernet-кадры внутри тоннеля;<br />
позволяет использовать мосты.</p>

<p>Недостатки:<br />
в тоннель попадает broadcast-трафик, что иногда не требуется;<br />
добавляет свои заголовки поверх заголовков Ethernet на все пакеты, которые следуют через тоннель;<br />
в целом, менее масштабируем из-за предыдущих двух пунктов;<br />
не поддерживается устройствами Android и iOS (по информации с сайта OpenVPN).</p>

<p>TUN:</p>

<p>Преимущества:<br />
передает только пакеты протокола IP (3й уровень);<br />
сравнительно (отн. TAP) меньшие накладные расходы и, фактически, ходит только тот IP-трафик, который предназначен конкретному клиенту.</p>

<p>Недостатки:<br />
broadcast-трафик обычно не передаётся;<br />
нельзя использовать мосты.</p>

<h4>2. RAS на базе OpenVPN</h4>

<p>Для выполнения данного задания создадим новый Vagrantfile:</p>

<pre></pre>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.define "rasvpn" do |rasvpn|
    rasvpn.vm.hostname = "rasvpn.loc"
    rasvpn.vm.network "private_network", ip: '192.168.20.10', adapter: 2, bridge: "wlp2s0"
  end
end</pre>

<p>Запустим эту виртуальную машину:</p>

<pre>[user@localhost vpn]$ vagrant up</pre>

<p>Подключаемся к ней по ssh и заходим под пользователем root:</p>

<pre>[user@localhost vpn]$ vagrant ssh rasvpn
[vagrant@rasvpn ~]$ sudo -i
[root@rasvpn ~]#</pre>

<p>Отключим SELinux:</p>

<pre>[root@rasvpn ~]# setenforce 0
[root@rasvpn ~]#</pre>

<p>1. Устанавливаем репозиторий EPEL:</p>

<pre>[root@rasvpn ~]# yum install -y epel-release</pre>

<p>2. Устанавливаем необходимые пакеты:</p>

<pre>[root@rasvpn ~]# yum install -y openvpn easy-rsa</pre>

<p>3. Переходим в директорию /etc/openvpn/ и инициализируем pki:</p>

<pre>[root@rasvpn ~]# cd /etc/openvpn/
[root@rasvpn openvpn]# </pre>

<pre>[root@rasvpn openvpn]# <b>/usr/share/easy-rsa/3.0.8/easyrsa init-pki</b>

init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /etc/openvpn/pki

[root@rasvpn openvpn]#</pre>

<p>4. Сгенерируем необходимые ключи и сертификаты для сервера:</p>

<pre>[root@rasvpn openvpn]# <b>echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa build-ca nopass</b>
Using SSL: openssl OpenSSL 1.0.2k-fips  26 Jan 2017
Generating RSA private key, 2048 bit long modulus
.......................................................................................+++
.................................................................................................................................................+++
e is 65537 (0x10001)
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:
CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/etc/openvpn/pki/ca.crt


[root@rasvpn openvpn]#</pre>

<pre>[root@rasvpn openvpn]# <b>echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa gen-req server nopass</b>
Using SSL: openssl OpenSSL 1.0.2k-fips  26 Jan 2017
Generating a 2048 bit RSA private key
....................................................+++
....+++
writing new private key to '/etc/openvpn/pki/easy-rsa-3413.jAAeTr/tmp.OEVKQa'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [server]:
Keypair and certificate request completed. Your files are:
req: /etc/openvpn/pki/reqs/server.req
key: /etc/openvpn/pki/private/server.key


[root@rasvpn openvpn]#</pre>

<pre>[root@rasvpn openvpn]# <b>echo 'yes' | /usr/share/easy-rsa/3.0.8/easyrsa sign-req server server</b>
Using SSL: openssl OpenSSL 1.0.2k-fips  26 Jan 2017


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a server certificate for 825 days:

subject=
    commonName                = rasvpn


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: Using configuration from /etc/openvpn/pki/easy-rsa-3455.vz4HrX/tmp.eLlr31
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'rasvpn'
Certificate is to be certified until Dec 28 12:01:03 2024 GMT (825 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /etc/openvpn/pki/issued/server.crt


[root@rasvpn openvpn]#</pre>

<pre>[root@rasvpn openvpn]# <b>/usr/share/easy-rsa/3.0.8/easyrsa gen-dh</b>
Using SSL: openssl OpenSSL 1.0.2k-fips  26 Jan 2017
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
........................................+........+..........................................................++*++*

DH parameters of size 2048 created at /etc/openvpn/pki/dh.pem


[root@rasvpn openvpn]#</pre>

<pre>[root@rasvpn openvpn]# <b>openvpn --genkey --secret ta.key</b>
[root@rasvpn openvpn]#</pre>

<p>5. Сгенерируем сертификаты для клиента:</p>

<pre>[root@rasvpn openvpn]# <b>echo 'client' | /usr/share/easy-rsa/3/easyrsa gen-req client nopass</b>
Using SSL: openssl OpenSSL 1.0.2k-fips  26 Jan 2017
Generating a 2048 bit RSA private key
.......................................................+++
.............................................+++
writing new private key to '/etc/openvpn/pki/easy-rsa-3541.OBl0wG/tmp.5eDzKd'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [client]:
Keypair and certificate request completed. Your files are:
req: /etc/openvpn/pki/reqs/client.req
key: /etc/openvpn/pki/private/client.key


[root@rasvpn openvpn]#</pre>

<pre>[root@rasvpn openvpn]# <b>echo 'yes' | /usr/share/easy-rsa/3/easyrsa sign-req client client</b>
Using SSL: openssl OpenSSL 1.0.2k-fips  26 Jan 2017


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a client certificate for 825 days:

subject=
    commonName                = client


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: Using configuration from /etc/openvpn/pki/easy-rsa-3569.o7XyZG/tmp.LpPrp4
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'client'
Certificate is to be certified until Dec 28 12:04:49 2024 GMT (825 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /etc/openvpn/pki/issued/client.crt


[root@rasvpn openvpn]#</pre>

<p>6. Создадим конфигурационный файл /etc/openvpn/server.conf:</p>

<pre>[root@rasvpn openvpn]# vi /etc/openvpn/server.conf
port 1207
proto udp
dev tun
ca /etc/openvpn/pki/ca.crt
cert /etc/openvpn/pki/issued/server.crt
key /etc/openvpn/pki/private/server.key
dh /etc/openvpn/pki/dh.pem
server 10.10.20.0 255.255.255.0
#route 192.168.20.0 255.255.255.0
push "route 192.168.20.0 255.255.255.0"
ifconfig-pool-persist ipp.txt
client-to-client
client-config-dir /etc/openvpn/client
keepalive 10 120
comp-lzo
persist-key
persist-tun
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3</pre>

<p>7. Зададим параметр iroute для клиента</p>

<pre>[root@rasvpn openvpn]# <b>echo 'iroute 192.168.33.0 255.255.255.0' > /etc/openvpn/client/client</b>
[root@rasvpn openvpn]#</pre>

<p>8. Запускаем openvpn сервер и добавляем в автозагрузку.</p>

<pre>[root@rasvpn openvpn]# systemctl enable openvpn@server --now
Created symlink from /etc/systemd/system/multi-user.target.wants/openvpn@server.service to /usr/lib/systemd/system/openvpn@.service.
[root@rasvpn openvpn]#</pre>

<p>9. Скопируем следующие файлы сертификатов и ключ для клиента на хост-машину:</p>

/etc/openvpn/pki/ca.crt
/etc/openvpn/pki/issued/client.crt
/etc/openvpn/pki/private/client.key

<p>Файлы расположим в той же директории, что и client.conf.</p>

<p>10. Создадим конфигурационны файл клиента client.conf на хост-машине:</p>

<pre>[root@localhost openvpn]# vi ./client.conf
dev tun
proto udp
remote 192.168.20.10 1207
client
resolv-retry infinite
ca /etc/openvpn/ca.crt
cert /etc/openvpn/client.crt
key /etc/openvpn/client.key
route 192.168.20.0 255.255.255.0
persist-key
persist-tun
comp-lzo
verb 3</pre>

<p>11. После того, как все готово, подключаемся к openvpn сервер с хост-машины, например, openvpn --config client.conf:</p>

<pre>[user@localhost vpn]$ systemctl start openvpn@client
[user@localhost vpn]$</pre>

<p>12. При успешном подключении проверяем пинг в внутреннему IP адресу сервера в туннеле:</p>

<pre>[root@localhost openvpn]# ping -c 4 10.10.20.1
PING 10.10.20.1 (10.10.20.1) 56(84) bytes of data.
64 bytes from 10.10.20.1: icmp_seq=1 ttl=64 time=2.19 ms
64 bytes from 10.10.20.1: icmp_seq=2 ttl=64 time=1.54 ms
64 bytes from 10.10.20.1: icmp_seq=3 ttl=64 time=1.53 ms
64 bytes from 10.10.20.1: icmp_seq=4 ttl=64 time=1.49 ms

--- 10.10.20.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 1.492/1.691/2.194/0.291 ms
[root@localhost openvpn]#</pre>

<p>13. Также проверяем командой ip r (netstat -rn) на хостовой машине, что сеть туннеля импортирована в таблицу маршрутизации:</p>

<pre>[root@localhost openvpn]# <b>ip r</b>
default via 192.168.1.1 dev wlp2s0 proto dhcp metric 600 
<b>10.10.20.0/24 via 10.10.20.5 dev tun0</b> 
10.10.20.5 dev tun0 proto kernel scope link src 10.10.20.6 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
172.18.0.0/16 dev br-d99b6e707266 proto kernel scope link src 172.18.0.1 
172.19.0.0/16 dev br-358044cf8e8a proto kernel scope link src 172.19.0.1 
172.20.0.0/16 dev br-47922bceb76f proto kernel scope link src 172.20.0.1 
192.168.1.0/24 dev wlp2s0 proto kernel scope link src 192.168.1.152 metric 600 
192.168.10.0/24 dev vboxnet3 proto kernel scope link src 192.168.10.1 
192.168.20.0/24 dev vboxnet4 proto kernel scope link src 192.168.20.1 
192.168.30.0/24 dev vboxnet5 proto kernel scope link src 192.168.30.1 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 
[root@localhost openvpn]#</pre>

<pre>[root@localhost openvpn]# netstat -rn
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG        0 0          0 wlp2s0
<b>10.10.20.0      10.10.20.5      255.255.255.0   UG        0 0          0 tun0</b>
10.10.20.5      0.0.0.0         255.255.255.255 UH        0 0          0 tun0
172.17.0.0      0.0.0.0         255.255.0.0     U         0 0          0 docker0
172.18.0.0      0.0.0.0         255.255.0.0     U         0 0          0 br-d99b6e707266
172.19.0.0      0.0.0.0         255.255.0.0     U         0 0          0 br-358044cf8e8a
172.20.0.0      0.0.0.0         255.255.0.0     U         0 0          0 br-47922bceb76f
192.168.1.0     0.0.0.0         255.255.255.0   U         0 0          0 wlp2s0
192.168.10.0    0.0.0.0         255.255.255.0   U         0 0          0 vboxnet3
192.168.20.0    0.0.0.0         255.255.255.0   U         0 0          0 vboxnet4
192.168.30.0    0.0.0.0         255.255.255.0   U         0 0          0 vboxnet5
192.168.122.0   0.0.0.0         255.255.255.0   U         0 0          0 virbr0
[root@localhost openvpn]#</pre>

<p>Как видим, мы получили соединение с виртуальным RAS сервером с помощью openvpn.</p>


