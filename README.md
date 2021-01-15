# BIND DNS

Примеры настройка DNS BIND.

Дано:

- днс сервера master *ns01 192.168.50.10*, slave *ns02 192.168.50.11*
- клиенты *client 192.168.50.15*, *client2 192.168.50.16*


## Добавление серверов в зону днс

Добавим имена *web1* и *web2* для *client* и *client2* соответственно в зону dns.lab. Для этого информацию о клиентах в конфигурационный файл *named.dns.lab* и увеличим параметр serial на один, для того чтобы slave днс узнал об изменении зоны.

	$TTL 3600
	$ORIGIN dns.lab.
	@               IN      SOA     ns01.dns.lab. root.dns.lab. (
	                            2711201407 ; serial
	                            3600       ; refresh (1 hour)
	                            600        ; retry (10 minutes)
	                            86400      ; expire (1 day)
	                            600        ; minimum (10 minutes)
	                        )

	                IN      NS      ns01.dns.lab.
	                IN      NS      ns02.dns.lab.

	; DNS Servers
	ns01            IN      A       192.168.50.10
	ns02            IN      A       192.168.50.11
	web1            IN      A       192.168.50.15
	web2            IN      A       192.168.50.16

Перезапускаем named.service и проверяем наших клиентов командой nslookup

	[root@ns01 vagrant]# systemctl restart named.service 
	[root@ns01 vagrant]# systemctl status named          
	● named.service - Berkeley Internet Name Domain (DNS)
	   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
	   Active: active (running) since Thu 2021-01-14 14:11:54 UTC; 9s ago
	  Process: 4291 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
	  Process: 4304 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
	  Process: 4302 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
	 Main PID: 4307 (named)
	   CGroup: /system.slice/named.service
	           └─4307 /usr/sbin/named -u named -c /etc/named.conf

	[root@ns01 vagrant]# nslookup web1.dns.lab 192.168.50.10
	Server:		192.168.50.10
	Address:	192.168.50.10#53

	Name:	web1.dns.lab
	Address: 192.168.50.15

	[root@ns01 vagrant]# nslookup web2.dns.lab 192.168.50.10
	Server:		192.168.50.10
	Address:	192.168.50.10#53

	Name:	web2.dns.lab
	Address: 192.168.50.16

## Добавление новой зоны

Добавим новую зонy newdns.lab. Создадим новый конфигурационный файл */etc/named/named.newdns.lab*
Примечание: Когда мы добавляем новую зону, которая явно не совпадает с нашей старой зоной, конфигурационный файл выглядит немного по другому. Также не лишним будет проверить кофиги командой *namrd-checkconf -z*. 

	$TTL 3600
	newdns.lab.      IN      SOA     ns01.dns.lab. root.dns.lab. (
	                            2711201408 ; serial
	                            3600       ; refresh (1 hour)
	                            600        ; retry (10 minutes)
	                            86400      ; expire (1 day)
	                            600        ; minimum (10 minutes)
	                        )

	newdns.lab.      IN      NS      ns01.dns.lab.
	newdns.lab.      IN      NS      ns02.dns.lab.

	; DNS Servers
	ns01            IN      A       192.168.50.10
	ns02            IN      A       192.168.50.11
	www            IN      A       192.168.50.15
	www            IN      A       192.168.50.16

И добавляем секцию для зоны *newdns*

	zone "newdns.lab" {
	    type master;
	    allow-transfer { key "zonetransfer.key"; };
	    file "/etc/named/named.newdns.lab";
	};

Перезапускаем и проверяем:

	[root@ns01 etc]# systemctl status named
	● named.service - Berkeley Internet Name Domain (DNS)
	   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
	   Active: active (running) since Fri 2021-01-15 15:39:21 UTC; 36s ago
	  Process: 23750 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
	  Process: 23763 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
	  Process: 23761 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
	 Main PID: 23766 (named)
	   CGroup: /system.slice/named.service
	           └─23766 /usr/sbin/named -u named -c /etc/named.conf

	[root@ns01 etc]# nslookup www.newdns.lab 192.168.50.10
	Server:		192.168.50.10
	Address:	192.168.50.10#53

	Name:	www.newdns.lab
	Address: 192.168.50.15
	Name:	www.newdns.lab
	Address: 192.168.50.16

## Split DNS