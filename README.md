# sxema
https://app.diagrams.net/?src=about#L123.drawio#%7B"pageId"%3A"e6KJ7m68IgASiborYQRq"%7D
https://app.diagrams.net/?src=about#L123.drawio#%7B"pageId"%3A"e6KJ7m68IgASiborYQRq"%7D

====
info!
https://demoekz2024.tilda.ws
https://vk.com/doc534963213_676416735?hash=svhVSGVh3aOmr14qRhYNQQ5l1RjKPzq0w0SNFcg6OmL&dl=hW2pztdIkrwE4PBBoyBmjx6iupkAKwIvS2YP7EqC8i4
=========
txt!
Настройка интернета
iptables –t nat  -A POSTROUTING –o ens33 –j MASQUERADE
apt install  iptables-persistent


chattr +i - запрет на изменения
chattr -i - разрешение на изменение

НА ВСЕХ МАШИНАХ 
nano /etc/resolv.conf
nameserver 192.168.200.200
nameserver 8.8.8.8







Web-L Rtr-L SRV Rtr-r CLI ISP Web-r
nano /etc/network/interfaces
auto ens…
iface ens… inet static

Rtr-L SRV Rtr-r
nano /etc/sysctl.conf
net.ipv4.ip_forward = 1 

Rtr-L  
nano /etc/nftables.conf
table ip nat {
	chain portrouting {
	type nat hook postrouting priority 0;
	ip saddr 192.168.200.0/24 oifname ens33 counter masquerade;
}

Rtr-r
table ip nat {
	chain portrouting {
	type nat hook postrouting priority 0;
	ip saddr 172.16.100.0/24 oifname ens33 counter masquerade;
}


nft -f /etc/nftables.conf – команда проверки
systemctl enable --now nftables добавить его в автозагрузку и активировать 
Для этого достаточно выполнить с WEB-L, WEB-R или SRV трассировку до сервера ISP при помощи команды traceroute -n 5.5.5.1

nano /etc/gre.up
Rtr-L
#!/bin/bash
ip tunnel add tun0 mode gre local 4.4.4.100 remote 5.5.5.100
ip addr add 10.5.5.1/30 dev tun0
ip link set up tun0
ip route add 172.16.100.0/24 via 10.5.5.2

Rtr-r
#!/bin/bash
ip tunnel add tun0 mode gre local 5.5.5.100 remote 4.4.4.100
ip addr add 10.5.5.2/30 dev tun0
ip link set up tun0
ip route add 192.168.200.0/24 via 10.5.5.1

После того, как скрипты были написаны, необходимо дать им права на выполнение при помощи команды chmod +x /etc/gre.up
/etc/gre.up

nano /etc/crontab
Для этого на RTR-R и RTRL редактируется файл /etc/crontab и в конце добавляется строка 
@reboot root /etc/gre.up

В первую очередь необходимо установить данный пакет на RTR-R и RTR-L при помощи команды apt install strongswan
nano /etc/ipsec.conf

Rtr-r
conn vpn
	auto=start
	type=tunnel
	authby=secret
	left=5.5.5.100
	right=4.4.4.100
	leftsubnet=0.0.0.0/0
	rightsubnet=0.0.0.0/0
	leftprotoport=gre
	rightprotoport=gre
	ike=aes128-sha256-modp3072
	esp=aes128-sha256

Rtr-L
conn vpn
	auto=start
	type=tunnel
	authby=secret
	left=4.4.4.100
	right=5.5.5.100
	leftsubnet=0.0.0.0/0
	rightsubnet=0.0.0.0/0
	leftprotoport=gre
	rightprotoport=gre
	ike=aes128-sha256-modp3072
	esp=aes128-sha256

nano /etc/ipsec.secrets
4.4.4.100  5.5.5.100  : PSK “P@ssw0rd”
systemctl enable --now ipsec
ipsec status проверка

/etc/nftables.conf

Rtr-L
	udp dport 53 accept;
	tcp dport 80 accept;
	tcp dport 443 accept;
	ct state {established, related} accept;
	ip protocol gre accept;
	ip protocol icmp accept;
	udp dport 500 accept;
	ip saddr 192.168.200.0/24 accept;
	ip saddr 172.16.100.0/24 accept;
	ip version 4 drop;
nft -f /etc/nftables.conf

Rtr-r
	tcp dport 80 accept;
	tcp dport 443 accept;
	ct state {established, related} accept;
	ip protocol gre accept;
	ip protocol icmp accept;
	udp dport 500 accept;
	ip saddr 192.168.200.0/24 accept;
	ip saddr 172.16.100.0/24 accept;
	ip version 4 drop;

Rtr-L
tcp dport 2222 counter packets 0 bytes 0
chain prerouting {
	type nat hook prerouting priority filter; policy accept;
	tcp dport 2222 dnat to 192.168.200.100:22
Rtr-r
tcp dport 2244 accept
chain prerouting {
	type nat hook prerouting priority filter; policy accept;
	tcp dport 2244 dnat to 172.16.100.100:22

создать пользователя на WEB-l WEB-R
adduser user
c isp
ssh user@4.4.4.100 –p 2222

ISP
apt install bind9
nano /etc/bind/named.conf.options


Где forwardes заменить значение с 0.0.0.0 на 8.8.8.8
dnssec-validation no;
allow-query {any;};
recursion yes;
listen-on {any; };
};


nano /etc/bind/named.conf.default-zones
zone “demo.wsr” {
	type master;
	file “/etc/bind/demo.wsr”;
	forwarders {};
};

Для создания шаблона зоны demo.wsr необходимо перейти в каталог /etc/bind и скопировать файл db.local в /etc/bind с названием demo.wsr. Сделать это можно при помощи команды 
cd /etc/bind; cp db.local demo.wsr
nano demo.wsr
После for  demo.wsr

$ORIGIN demo.wsr
@	  IN	 NS 	demo.wsr.
@	  IN	 A	3.3.3.1
isp	  IN 	 A 	3.3.3.1
www	  IN	 A	4.4.4.100
www	  IN	 A	5.5.5.100
internet	 IN 	CNAME	 	isp
$ORIGIN int.demo.wsr.
@	  IN	 NS 	int.demo.wsr.
@	  IN	 A	4.4.4.100
named-checkconf
named-checkconf- z
systemctl restart bind9


CLI
Настроить ip
ip 3.3.3.10/24
gateway 3.3.3.1
dns 1 3.3.3.1
dns 2 8.8.8.8

RTR-L
/etc/nftables.conf
udp dport 53 dnat to 192.168.200.200:53; (по желанию)



SRV
apt install bind9
nano /etc/bind/named.conf.options
Где forwardes заменить значение с 0.0.0.0 на 3.3.3.1

dnssec-validation no;
allow-query {any;};
recursion yes;
listen-on {any; };
allow-recursion {172.16.100.0/24; 192.168.200.0/24; };
};
nano /etc/bind/named.conf.default-zones
zone “int.demo.wsr” {
	type master;
	file “/etc/bind/int.demo.wsr”;
};
zone “200.168.192.in-addr.arpa” {
	type master;
	file “/etc/bind/left.reverse”;
};
zone “100.16.172.in-addr.arpa” {
	type master;
	file “/etc/bind/right.reverse”;
};
cd /etc/bind
cp db.local int.demo.wsr
cp db.local left.reverse
cp db.local right.reverse
ls


Зона int.demo.wsr
@	IN	NS	int.demo.wsr.
@	IN	A	192.168.200.200
web-l	IN	A	192.168.200.100
web-r	IN	A	172.16.100.100
srv	IN	A	192.168.200.200
rtr-l	IN	A	192.168.200.254
rtr-r	IN	A	172.16.100.254
ntp	IN	CNAME	srv
dns	IN	CNAME	srv
webapp	IN	CNAME	web-l


Зона left.reverse
@	IN	SOA	200.168.192.in-addr.arpa. root.200.168.192.in-addr.arpa. (

@	IN	NS	int.demo.wsr.
@	IN	A	192.168.200.200
100	PTR	web-l.int.demo.wsr.
200	PTR	srv.int.demo.wsr.
254	PTR	rtr-l.int.demo.wsr.


Зона right.reverse
@	IN	SOA	100.16.172.in-addr.arpa. root.100.16.172.in-addr.arpa. (

@	IN	NS	int.demo.wsr.
@	IN	A	192.168.200.200
100	PTR	web-r.int.demo.wsr.
254	PTR	rtr-r.int.demo.wsr.

ISP
apt install chrony
nano /etc/chrony/chrony.conf
server 127.127.1.0
allow 5.5.5.0/24
allow 4.4.4.0/24
allow 3.3.3.0/24
local stratum 4
systemctl restart chronyd


CLI
Выполнить
regedit
В открывшемся окне редактора реестра необходимо перейти по пути HKEY_
LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Parameters и найти ключ 
NtpServer
Задать значение 3.3.3.1
В командной строке 
net stop w32time — отключаем сервис w32time;
net start w32time — включаем сервис w32time.
После перезапуска сервера необходимо выполнить команду w32tm /resync /rediscover 
для синхронизации с новым NTP-сервером
Далее можно выполнить проверку состояния синхронизации при помощи команды
w32tm /query /status

На сервере времени ISP можно ввести команду chronyc clients

SRV
На новых неразмеченных дисках необходимо создать разделы при помощи утилиты 
cfdisk
cfdisk /dev/sdX , где Х — буква того диска, который вы желаете форматировать.
Например, разметка диска /dev/sda будет выглядеть вот так:
cfdisk /dev/sda — введите данную команду в терминал.
Откроется псевдографическая утилита, на данном этапе необходимо указать метку 
раздела — значение по умолчанию, GPT.

Далее необходимо выбрать New.

Далее указывается размер создаваемого раздела, по умолчанию указывается все свободное пространство целевого диска. Нажатием Enter происходит соглашение с размером 
раздела.

После чего необходимо перейти в опцию меню Type. 


В указанном списке выбирается Linux RAID.

Вручную пишется слово yes, чтобы принять изменения и отформатировать диск.

После внесенных правок необходимо покинуть утилиту cfdisk, выбрав параметр Quit.

Необходимо повторить данную процедуру также и со вторым добавленным диском. 

Созданные разделы можно увидеть в выводе команды lsblk

apt install mdadm

mdadm --create /dev/md0 –level=1 --raid-devices=2 /dev/sda1 /dev/sdc1

Процесс сборки RAID-массива можно наблюдать в выводе файла /proc/mdstat
cat /proc/mdstat

mdadm --detail --scan --verbose | tee -a /etc/mdadm/mdadm.conf

Далее необходимо пересоздать initramfs с поддержкой данного массива при помощи 
команды update-initramfs-u — данная команда обновит информацию о монтируемых при загрузке разделах RAID

Готовый массив также отобразится в выводе утилиты lsblk

Форматирование можно произвести при помощи команды
mkfs.ext4 /dev/md0

Далее необходимо создать точку монтирования данного массива, сделать это можно 
при помощи команды
mkdir /mnt/storage

После этого необходимо добавить запись в файл /etc/fstab


/dev/md0	/mnt/storage	ext4	defaults	0	0

Протестировать автоматическое монтирование без перезагрузки машины можно при 
помощи команды mount –av

Необходимо перезагрузить машину с помощью команды reboot
После успешной перезагрузки вводится команда lsblk

apt install samba

nano /etc/samba/smb.conf

[share]
path =/mnt/storage
browseable = yes
read only = no
guest ok = yes
force user = smbuser
Необходимо создать пользователя командой useradd smbuser и внести следующие изменения в /etc/samba/smb.conf

map to guest = bad user
guest account = smbuser

systemctl restart smbd
systemctl restart nmbd



WEB-L
mkdir /opt/share
mount //192.168.200.200/share /opt/share/ -o guest
ls /opt/share

nano /etc/fstab

//192.168.200.200/share  /opt/share	cifs	defaults,guest,rm	0	0


Проверить возможность монтирования без перезагрузки можно при помощи команды 
mount -av


SRV

apt install libcrypt-openssl-* -y 

/usr/lib/ssl/openssl.cnf

В строке 48 следует изменить значение dir на /var/ca
В строке 78 изменить default_days на 1024
В строке 134 изменить значение страны на RU
В строке 139 поставить точку вместо значения, в строке 144 прописать DEMO.WSR
В строке 151 и также поставить точку

cd /usr/lib/ssl/misc
nano CA.pl
В строке 28 изменить значение days на 1024
В строке 37 изменить значение на /var/ca
exit

cd /usr/lib/ssl/misc
./CA.pl -newca

Ввести пароль
Где common name: WSR CA

cp /var/ca/cacert.pem /usr/local/share/ca-certificates/cacert.crt
update-ca-certificates

Проверить работоспособность доверия можно при помощи команды
openssl verify /usr/local/share/ca-certificates/cacert.crt

nano /etc/ssh/sshd_config

Отредактировать 34-ю строчку в данном документе yes

systemctl restart sshd

scp root@192.168.100.200:/var/ca/cacert.pem ./cacert.crt





CLI

scp -P 2222 root@4.4.4.100:/root/ca* ./



























Докер на linux
После выполнения установки Docker
WEB-l b Web-R
apt install docker docker.io

После настройки виртуальной машины необходимо ввести команду mount /dev/cdrom 
/media/cdrom

cp /media/cdrom/app.tar /root 

docker load < app.tar


Проверить, что образ импортировался корректно, можно при помощи команды 
docker images

При помощи команды docker run -d -p 5000:5000 --name app app


docker ps

Посмотреть логи приложения можно при помощи команды docker logs app

apt install curl
curl 127.0.0.1



SRV 
./CA.pl -newreq-nodes
Enter
./CA.pl -sign
Enter

/root/www - mkdir /root/www


mv /usr/lib/ssl/misc/newcert.pem /root/www/

mv /usr/lib/ssl/misc/newkey.pem /root/www/

RTR-L и RTR-R тоже необходимо отредактировать /etc/ssh/
sshd_config по аналогии с предыдущим заданием


RTR-L
scp new* root@192.168.200.254:/root/


RTR-R
scp new* root@172.16.100.254:/root/

SRV
scp /var/ca/cacert.pem root@192.168.100.254:/root/
scp /var/ca/cacert.pem root@172.16.100.254:/root/


RTR-R и RTR-L
apt install haproxy
mkdir /root/www/
cd /root/www/
cp /root/newcert.pem /root/www/
cp /root/newkey.pem /root/www/
mv newkey.pem newcert.pem.key



cp /root/cacert.pem /usr/local/share/ca-certificates/cacert.crt



nano /etc/haproxy/haproxy.cfg
RTR-R
frontend www.demo.wsr
	bind 5.5.5.100:80
	bind 5.5.5.100:443 ssl crt /root/www/newcert.pem
	http-request redirect scheme https unless  { ssl_fc }
	default_backend web

backend web
	balance roundrobin
	option httpchk
	server web-r 172.16.100.100:5000 check
	server web-l 4.4.4.100:443 check ssl ca-file /usr/local/share/ca-certificates/cacert.crt



RTR-L
frontend www.demo.wsr
	bind 4.4.4.100:80
	bind 4.4.4.100:443 ssl crt /root/www/newcert.pem
	http-request redirect scheme https unless  { ssl_fc }
	default_backend web

backend web
	balance roundrobin
	option httpchk
	server web-r 192.168.200.100:5000 check
	server web-l 5.5.5.100:443 check ssl ca-file /usr/local/share/ca-certificates/cacert.crt

systemctl restart haproxy


На CLI

http://www.demo.wsr
