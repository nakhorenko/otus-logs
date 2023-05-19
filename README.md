## Настраиваем центральный сервер для сбора логов
<details>
  <summary>Основное задание</summary>
Укажем часовой пояс (Московское время):
```
[root@web ~]# cp /usr/share/zoneinfo/Europe/Moscow /etc/localtime
```

Перезупустим службу NTP Chrony
```
[root@web ~]# systemctl restart chronyd
```
Проверим, что служба работает корректно и проверим время
```
[root@web ~]# systemctl status chronyd
● chronyd.service - NTP client/server
Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
Active: active (running) since Sun 2021-12-12 12:56:51 MSK; 8h ago
Docs: man:chronyd(8)
man:chrony.conf(5)

[root@web ~]# date
Fri May 19 16:18:12 MSK 2023
```
Для установки nginx сначала нужно установить epel-release и nginx
yum install -y nginx
```
[root@web ~]# yum install epel-release
[root@web ~]# yum install nginx
```
Проверка работоспособности вебсервера
```
[root@web ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-05-19 16:25:06 MSK; 1s ago
  Process: 3561 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3559 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3558 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3563 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3563 nginx: master process /usr/sbin/nginx
           └─3565 nginx: worker process
[root@web ~]# ss -tunl | grep 80
tcp    LISTEN     0      128       *:80                    *:*                  
tcp    LISTEN     0      128    [::]:80                 [::]:*            
```
На втором сервере (log) проверяем наличие rsyslog, и настраиваем его
```
[root@log ~]# systemctl status rsyslog
● rsyslog.service - System Logging Service
   Loaded: loaded (/usr/lib/systemd/system/rsyslog.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2023-05-19 13:32:28 UTC; 3s ago
     Docs: man:rsyslogd(8)
           http://www.rsyslog.com/doc/
 Main PID: 3409 (rsyslogd)
   CGroup: /system.slice/rsyslog.service
           └─3409 /usr/sbin/rsyslogd -n
```
Находим строки udp & tcp с портом 514 и раскомменчиваем их
```
# Provides UDP syslog reception
module(load="imudp")
input(type="imudp" port="514")

# Provides TCP syslog reception
module(load="imtcp")
input(type="imtcp" port="514")
```
В конец файла добавляются строчки
```
#Add remote logs
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~
```
Сохраняемся и рестартуем сервис
```
[root@log ~]# systemctl restart rsyslog
```
Должны быть открыты порты 514 по tcp и udp
```
[root@log ~]# ss -tunl | grep 514
udp    UNCONN     0      0         *:514                   *:*                  
udp    UNCONN     0      0      [::]:514                [::]:*                  
tcp    LISTEN     0      25        *:514                   *:*                  
tcp    LISTEN     0      25     [::]:514                [::]:*
```
Идём в настройки nginx на web
```
[root@web ~]# nano /etc/nginx/nginx.conf
```
Добавляем в строки error_log и access_log строчки. Всё настроено корректно.
```
error_log syslog:server=192.168.50.15:514,tag=nginx_error;
access_log syslog:server=192.168.50.15:514,tag=nginx_access,severity=info combined;
[root@web ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@web ~]# systemctl restart nginx
```
Идем на log сервер и проверяем директорию с логами
```
[root@log rsyslog]# cd /var/log/rsyslog/
[root@log rsyslog]# ll
total 0
drwx------. 2 root root 122 May 19 14:01 log
drwx------. 2 root root  30 May 19 13:50 web
```
```
[root@log web]# tail -f nginx_access.log 
May 19 17:05:42 web nginx_access: 192.168.50.1 - - [19/May/2023:17:05:42 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36"
May 19 17:05:43 web nginx_access: 192.168.50.1 - - [19/May/2023:17:05:43 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36"
May 19 17:05:43 web nginx_access: 192.168.50.1 - - [19/May/2023:17:05:43 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36"
May 19 17:05:43 web nginx_access: 192.168.50.1 - - [19/May/2023:17:05:43 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36"
May 19 17:05:43 web nginx_access: 192.168.50.1 - - [19/May/2023:17:05:43 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36"
May 19 17:05:43 web nginx_access: 192.168.50.1 - - [19/May/2023:17:05:43 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36"
May 19 17:05:55 web nginx_access: 192.168.50.1 - - [19/May/2023:17:05:55 +0300] "GET / HTTP/1.1" 200 4833 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36"
```
удалил картинку и лого из директории /usr/share/nginx/html
```
[root@log web]# tail -f nginx_error.log  
May 19 17:05:55 web nginx_error: 2023/05/19 17:05:55 [error] 3925#3925: *9 open() "/usr/share/nginx/html/img/centos-logo.png" failed (2: No such file or directory), client: 192.168.50.1, server: _, request: "GET /img/centos-logo.png HTTP/1.1", host: "192.168.50.10", referrer: "http://192.168.50.10/"
May 19 17:05:55 web nginx_error: 2023/05/19 17:05:55 [error] 3925#3925: *10 open() "/usr/share/nginx/html/img/header-background.png" failed (2: No such file or directory), client: 192.168.50.1, server: _, request: "GET /img/header-background.png HTTP/1.1", host: "192.168.50.10", referrer: "http://192.168.50.10/"
```

## Настройка аудита, контролирующего изменения конфигурации nginx
Настроим аудит изменения конфигурации nginx.
Добавим правило, которое будет отслеживать изменения в конфигруации nginx. Для этого в конец
файла /etc/audit/rules.d/audit.rules добавим следующие строки:
```
-w /etc/nginx/nginx.conf -p wa -k nginx_conf
-w /etc/nginx/default.d/ -p wa -k nginx_conf
[root@web ~]# service auditd restart
Stopping logging:                                          [  OK  ]
Redirecting start to /bin/systemctl start auditd.service
```
Я поменял в настройках nginx worker_connections 2048
Вывод лога аудита
```
grep nginx_conf /var/log/audit/audit.log
type=CONFIG_CHANGE msg=audit(1684506063.254:1067): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1684506063.254:1068): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1684506116.683:1074): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1684506116.683:1075): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1684506116.699:1078): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1684506116.699:1079): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1684506126.468:1085): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1684506126.468:1086): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1684506126.468:1089): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1684506126.468:1090): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
type=SYSCALL msg=audit(1684506161.244:1092): arch=c000003e syscall=2 success=yes exit=3 a0=253c590 a1=441 a2=1b6 a3=63 items=2 ppid=3331 pid=22792 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=4 comm="nano" exe="/usr/bin/nano" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
type=SYSCALL msg=audit(1684506172.998:1093): arch=c000003e syscall=2 success=yes exit=3 a0=2540ea0 a1=241 a2=1b6 a3=7ffcb8fd9e60 items=2 ppid=3331 pid=22792 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=4 comm="nano" exe="/usr/bin/nano" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
```
Все логи собираются на удаленном сервере, аудит собирает данные об изменениях в nginx.conf

Далее настроим пересылку логов на удаленный сервер
```
[root@web ~]# nano /etc/audit/auditd.conf:
log_format = RAW
name_format = HOSTNAME
```
В au-remote.conf меняем active на yes
```
[root@web ~]# nano /etc/audisp/plugins.d/au-remote.conf
active = yes
direction = out
path = /sbin/audisp-remote
type = always
#args =
format = string
```
И в /etc/audisp/audisp-remote.conf указываем удаленный сервер
```
[root@web ~]# nano /etc/audisp/audisp-remote.conf
remote_server = 192.168.50.15
[root@web ~]# service auditd restart
```
Откроем 60 порт на log сервере
```
[root@log web]# nano /etc/audit/auditd.conf
tcp_listen_port = 60
[root@log web]# service auditd restart
```
Поменяв атрибут файла настроек nginx, видим на лог сервере
```
[root@log web]# grep web /var/log/audit/audit.log
node=web type=DAEMON_START msg=audit(1684506804.436:2150): op=start ver=2.8.5 format=raw kernel=3.10.0-1127.el7.x86_64 auid=4294967295 pid=22962 uid=0 ses=4294967295 subj=system_u:system_r:auditd_t:s0 res=success
node=web type=CONFIG_CHANGE msg=audit(1684506804.592:1099): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1684506804.592:1100): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1684506804.597:1101): audit_backlog_limit=8192 old=8192 auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 res=1
node=web type=CONFIG_CHANGE msg=audit(1684506804.598:1102): audit_failure=1 old=1 auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 res=1
node=web type=CONFIG_CHANGE msg=audit(1684506804.598:1103): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1684506804.598:1104): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=SERVICE_START msg=audit(1684506804.601:1105): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=auditd comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
node=web type=SYSCALL msg=audit(1684507161.743:1106): arch=c000003e syscall=268 success=yes exit=0 a0=ffffffffffffff9c a1=a94420 a2=1ed a3=7ffc9bdc85a0 items=1 ppid=3331 pid=23002 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=4 comm="chmod" exe="/usr/bin/chmod" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
node=web type=CWD msg=audit(1684507161.743:1106):  cwd="/etc"
node=web type=PATH msg=audit(1684507161.743:1106): item=0 name="nginx/nginx.conf" inode=272853 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=NORMAL cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
node=web type=PROCTITLE msg=audit(1684507161.743:1106): proctitle=63686D6F64002B78006E67696E782F6E67696E782E636F6E66
```
</details>

позже добавлю с настройками через ансибл
