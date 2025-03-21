# OTUS.Lesson17.SELinux
Описание домашнего задания
1. Запустить Nginx на нестандартном порту 3-мя разными способами:
переключатели setsebool;
добавление нестандартного порта в имеющийся тип;
формирование и установка модуля SELinux.
К сдаче:
README с описанием каждого решения (скриншоты и демонстрация приветствуются). 

2. Обеспечить работоспособность приложения при включенном selinux.
развернуть приложенный стенд https://github.com/Nickmob/vagrant_selinux_dns_problems; 
выяснить причину неработоспособности механизма обновления зоны (см. README);
предложить решение (или решения) для данной проблемы;
выбрать одно из решений для реализации, предварительно обосновав выбор;
реализовать выбранное решение и продемонстрировать его работоспособность.

Ход выполнения домашнего задания:
ЗАДАНИЕ 1: Запустить Nginx на нестандартном порту 3-мя разными способами:
Скачиваем vagtantfile и запускаем виртуалку
Проверяем статус selinux:
```
[root@alma ~]# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```
Проверим статус nginx:
```
[root@alma ~]# systemctl status nginx
× nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: failed (Result: exit-code) since Thu 2025-03-20 13:47:33 UTC; 2min 10s ago
    Process: 1828 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 1829 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
        CPU: 24ms

Mar 20 13:47:33 alma.alma systemd[1]: Starting The nginx HTTP and reverse proxy server...
Mar 20 13:47:33 alma.alma nginx[1829]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Mar 20 13:47:33 alma.alma nginx[1829]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Mar 20 13:47:33 alma.alma nginx[1829]: nginx: configuration file /etc/nginx/nginx.conf test failed
Mar 20 13:47:33 alma.alma systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
Mar 20 13:47:33 alma.alma systemd[1]: nginx.service: Failed with result 'exit-code'.
Mar 20 13:47:33 alma.alma systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
```
Проверяем статус фаервола и конфиг nginx
```
[root@alma ~]# systemctl status firewalld
Unit firewalld.service could not be found.
[root@alma ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Далее проверим режим работы SELinux: getenforce 
```
[root@alma ~]# getenforce
Enforcing

```
1) Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool
```
[root@alma ~]# grep denied /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1742478453.811:186): avc:  denied  { name_bind } for  pid=1829 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```
Утилита audit2why покажет почему трафик блокируется. Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled. 
Включим параметр nis_enabled и перезапустим nginx: setsebool -P nis_enabled on
```
[root@alma ~]# setsebool -P nis_enabled on
[root@alma ~]# systemctl restart nginx
[root@alma ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Thu 2025-03-20 13:58:37 UTC; 2s ago
    Process: 1882 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 1883 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 1884 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 1885 (nginx)
      Tasks: 3 (limit: 12124)
     Memory: 2.9M
        CPU: 50ms
     CGroup: /system.slice/nginx.service
             ├─1885 "nginx: master process /usr/sbin/nginx"
             ├─1886 "nginx: worker process"
             └─1887 "nginx: worker process"

Mar 20 13:58:37 alma.alma systemd[1]: Starting The nginx HTTP and reverse proxy server...
Mar 20 13:58:37 alma.alma nginx[1883]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Mar 20 13:58:37 alma.alma nginx[1883]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Mar 20 13:58:37 alma.alma systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@alma ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```
Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим nis_enabled: setsebool -P nis_enabled off
```
[root@alma ~]# systemctl status nginx
× nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: failed (Result: exit-code) since Fri 2025-03-21 06:28:14 UTC; 2s ago
```
2) Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:
Поиск имеющегося типа, для http трафика: semanage port -l | grep http
```
[root@alma ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@alma ~]# semanage port -a -t http_port_t -p tcp 4881
[root@alma ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@alma ~]# systemctl restart nginx
[root@alma ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Fri 2025-03-21 06:30:10 UTC; 5s ago
    Process: 2548 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 2549 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 2550 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 2551 (nginx)
      Tasks: 3 (limit: 12124)
     Memory: 2.9M
        CPU: 56ms
     CGroup: /system.slice/nginx.service
             ├─2551 "nginx: master process /usr/sbin/nginx"
             ├─2552 "nginx: worker process"
             └─2553 "nginx: worker process"

Mar 21 06:30:10 alma.alma systemd[1]: Starting The nginx HTTP and reverse proxy server...
Mar 21 06:30:10 alma.alma nginx[2549]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Mar 21 06:30:10 alma.alma nginx[2549]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Mar 21 06:30:10 alma.alma systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Удалить нестандартный порт из имеющегося типа можно с помощью команды: semanage port -d -t http_port_t -p tcp 4881
рестартуем nginx и видим что он опять не запускается

3)  Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:

Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту: 
grep nginx /var/log/audit/audit.log | audit2allow -M nginx
```
[root@alma ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@alma ~]#  semodule -i nginx.pp
[root@alma ~]# systemctl start nginx
[root@alma ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Fri 2025-03-21 06:33:24 UTC; 8s ago
    Process: 2598 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 2599 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 2600 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 2601 (nginx)
      Tasks: 3 (limit: 12124)
     Memory: 2.9M
        CPU: 53ms
     CGroup: /system.slice/nginx.service
             ├─2601 "nginx: master process /usr/sbin/nginx"
             ├─2602 "nginx: worker process"
             └─2603 "nginx: worker process"

Mar 21 06:33:24 alma.alma systemd[1]: Starting The nginx HTTP and reverse proxy server...
Mar 21 06:33:24 alma.alma nginx[2599]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Mar 21 06:33:24 alma.alma nginx[2599]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Mar 21 06:33:24 alma.alma systemd[1]: Started The nginx HTTP and reverse proxy server.

```


ЗАДАНИЕ 2.
Обеспечение работоспособности приложения при включенном SELinux
Развернем тестовую среду.
```
nsupdate -k /etc/nsupdate -k /etc/named.zonetransfer.key
> server 10.200.3.96
> zone ddns.lab
> update add www.ddns.lab. 60 A 10.200.3.92
> send
update failed: SERVFAIL
> quit 

``` 
Изменения внести не получилось. Давайте посмотрим логи SELinux, чтобы понять в чём может быть проблема.
Для этого воспользуемся утилитой audit2why
```
[root@Client ~]# cat /var/log/audit/audit.log | audit2why
[root@Client ~]# 
```
Подключаемся к VM серверу и смотрим что там 
```
cat /var/log/aucat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1742546699.146:3126): avc:  denied  { write } for  pid=24065 comm="isc-net-0001" name="dynamic" dev="sda4" ino=8988881 scontext=system_u:system_r:named_t:s0 tcontext=unconfined_u:object_r:named_conf_t:s0 tclass=dir permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
```
В логах мы видим, что ошибка в контексте безопасности. Целевой контекст named_conf_t.
Для сравнения посмотрим существующую зону (localhost) и её контекст:
```
[root@ns01 ~]# ls -alZ /var/named/named.localhost
-rw-r-----. 1 root named system_u:object_r:named_zone_t:s0 152 Feb 19 16:04 /var/named/named.localhost
```
У наших конфигов в /etc/named вместо типа named_zone_t используется тип named_conf_t.
Проверим данную проблему в каталоге /etc/named:
```
[root@ns01 ~]# ls -laZ /etc/named
total 28
drw-rwx---.  3 root named system_u:object_r:named_conf_t:s0      119 Mar 21 08:42 .
drwxr-xr-x. 87 root root  system_u:object_r:etc_t:s0            8192 Mar 21 08:43 ..
drw-rwx---.  2 root named unconfined_u:object_r:named_conf_t:s0   56 Mar 21 08:39 dynamic
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      780 Mar 21 08:42 named.3.200.10.rev
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      606 Mar 21 08:39 named.dns.lab
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      605 Mar 21 08:39 named.dns.lab.view1
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      651 Mar 21 08:39 named.newdns.lab
```
Тут мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды: sudo semanage fcontext -l | grep named
Изменим тип контекста безопасности для каталога /etc/named: sudo chcon -R -t named_zone_t /etc/named
```
[root@ns01 ~]# ls -laZ /etc/named
total 28
drw-rwx---.  3 root named system_u:object_r:named_zone_t:s0      119 Mar 21 08:42 .
drwxr-xr-x. 87 root root  system_u:object_r:etc_t:s0            8192 Mar 21 08:43 ..
drw-rwx---.  2 root named unconfined_u:object_r:named_zone_t:s0   56 Mar 21 08:39 dynamic
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      780 Mar 21 08:42 named.3.200.10.rev
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      606 Mar 21 08:39 named.dns.lab
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      605 Mar 21 08:39 named.dns.lab.view1
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      651 Mar 21 08:39 named.newdns.lab
```
Пробуем снова внести изменения с клиента:
[root@Client ~]# nsupdate -k /etc/named.zonetransfer.key
```
> server 10.200.3.96
> zone ddns.lab
> update add www.ddns.lab. 60 A 10.200.3.92
> send
> q 	  
incorrect section name: q
> quit
[root@Client ~]# dig www.ddns.lab

; <<>> DiG 9.16.23-RH <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17366
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 155b8e073f4d592a0100000067dd2c40d358e1a83d078b0a (good)
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	10.200.3.92

;; Query time: 1 msec
;; SERVER: 10.200.3.96#53(10.200.3.96)
;; WHEN: Fri Mar 21 09:07:12 UTC 2025
;; MSG SIZE  rcvd: 85

```
Видим, что изменения применились. Попробуем перезагрузить хосты и ещё раз сделать запрос с помощью dig:
```
; <<>> DiG 9.16.23-RH <<>> www.ddns.lab @10.200.3.96
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 60828
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 88420ba8d0d77fab0100000067dd2e778853abc341c909f6 (good)
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	10.200.3.92

;; Query time: 3 msec
;; SERVER: 10.200.3.96#53(10.200.3.96)
;; WHEN: Fri Mar 21 09:16:39 UTC 2025
;; MSG SIZE  rcvd: 85

```
