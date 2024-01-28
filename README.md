# Otus_SELinux

## 1. Запустить nginx на нестандартном порту 3-мя разными способами:
переключатели setsebool;
добавление нестандартного порта в имеющийся тип;
формирование и установка модуля SELinux.


Заходим на сервер: vagrant ssh
Дальнейшие действия выполняются от пользователя root. Переходим в root пользователя: sudo -i

Для начала проверим, что в ОС отключен файервол, конфигурация nginx настроена без ошибок, работы SELinux.


```
[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@selinux ~]# getenforce
Enforcing
```

Отображается режим Enforcing. Данный режим означает, что SELinux будет блокировать запрещенную активность.


## 1. Способ: Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool

Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта.

```
[root@selinux ~]# cat /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1706433212.912:781): avc:  denied  { name_bind } for  pid=2755 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
[root@selinux ~]# grep 1706433212.912:781 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1706433212.912:781): avc:  denied  { name_bind } for  pid=2755 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled
	Allow access by executing:
	# setsebool -P nis_enabled 1
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx    
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2024-01-28 09:33:45 UTC; 16s ago
  Process: 3017 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3015 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3014 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3019 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3019 nginx: master process /usr/sbin/nginx
           └─3021 nginx: worker process
Jan 28 09:33:45 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 28 09:33:45 selinux nginx[3015]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 28 09:33:45 selinux nginx[3015]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 28 09:33:45 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Проверить работу nginx из браузера. Заходим в любой браузер на хосте и переходим по адресу http://127.0.0.1:4881


()[]


## Способ 2: Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип

Поиск имеющегося типа, для http трафика

```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

Добавим порт в тип http_port_t:

```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

Перезапустим службу nginx и проверим её работу:

```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2024-01-28 09:39:49 UTC; 13s ago
  Process: 3180 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3178 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3177 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3182 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3182 nginx: master process /usr/sbin/nginx
           └─3184 nginx: worker process

Jan 28 09:39:49 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jan 28 09:39:49 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 28 09:39:49 selinux nginx[3178]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 28 09:39:49 selinux nginx[3178]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 28 09:39:49 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

## Способ3: Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux


Попробуем снова запустить nginx. Nginx не запуститься, так как SELinux продолжает его блокировать. Посмотрим логи SELinux, которые относятся к nginx.


```
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sun 2024-01-28 09:41:27 UTC; 19s ago
  Process: 3180 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3208 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 3206 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3182 (code=exited, status=0/SUCCESS)

Jan 28 09:41:27 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jan 28 09:41:27 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 28 09:41:27 selinux nginx[3208]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 28 09:41:27 selinux nginx[3208]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jan 28 09:41:27 selinux nginx[3208]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jan 28 09:41:27 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jan 28 09:41:27 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jan 28 09:41:27 selinux systemd[1]: Unit nginx.service entered failed state.
Jan 28 09:41:27 selinux systemd[1]: nginx.service failed.
[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# grep nginx /var/log/audit/audit.log
type=SOFTWARE_UPDATE msg=audit(1706433212.414:778): pid=2601 uid=0 auid=1000 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='sw="nginx-filesystem-1:1.20.1-10.el7.noarch" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1706433212.742:780): pid=2601 uid=0 auid=1000 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='sw="nginx-1:1.20.1-10.el7.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1706433212.912:781): avc:  denied  { name_bind } for  pid=2755 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1706433212.912:781): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=5655086cc8b8 a2=10 a3=7ffe8af18f60 items=0 ppid=1 pid=2755 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1706433212.912:782): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=SERVICE_START msg=audit(1706434425.244:836): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_STOP msg=audit(1706434789.736:859): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_START msg=audit(1706434789.769:860): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_STOP msg=audit(1706434887.023:864): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1706434887.047:865): avc:  denied  { name_bind } for  pid=3208 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1706434887.047:865): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=556504aa28b8 a2=10 a3=7fffd5e8d730 items=0 ppid=1 pid=3208 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1706434887.047:866): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=AVC msg=audit(1706434928.357:867): avc:  denied  { name_bind } for  pid=3222 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1706434928.357:867): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55cdb6a228b8 a2=10 a3=7ffe519658c0 items=0 ppid=1 pid=3222 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1706434928.357:868): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```

Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту

```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2024-01-28 09:43:38 UTC; 12s ago
  Process: 3255 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3251 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3250 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3257 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3257 nginx: master process /usr/sbin/nginx
           └─3258 nginx: worker process

Jan 28 09:43:38 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 28 09:43:38 selinux nginx[3251]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 28 09:43:38 selinux nginx[3251]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 28 09:43:38 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```




