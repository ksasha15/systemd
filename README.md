# systemd
#### 1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default).
```
root@host:~# cat << EOF > /etc/default/watchlog
> # Configuration file for my watchlog service
# Place it to /etc/default
# File and word in that file that we will be monitor
WORD="ALERT"
LOG=/var/log/watchlog.log
EOF
root@host:~#
root@host:~# cat > /opt/watchlog.sh
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi
^C
root@host:~# ll /opt/watchlog.sh
-rw-r--r-- 1 root root 132 Feb  7 14:08 /opt/watchlog.sh
root@host:~# chmod +x /opt/watchlog.sh
root@host:~# ll /opt/watchlog.sh
-rwxr-xr-x 1 root root 132 Feb  7 14:08 /opt/watchlog.sh*
root@host:~# cat > /etc/systemd/system/watchlog.service
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/default/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
^C
root@host:~# cat > /etc/systemd/system/watchlog.timer
[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
^C
root@host:~# systemctl start watchlog.timer
root@host:~# systemctl start watchlog.service
root@host:~# tail -n 1000 /var/log/syslog | grep word
2026-02-07T13:54:49.617406+00:00 host kernel: systemd[1]: Started systemd-ask-password-wall.path - Forward Password Requests to Wall Directory Watch.
2026-02-07T13:54:49.617585+00:00 host kernel: audit: type=1400 audit(1770472475.903:2): apparmor="STATUS" operation="profile_load" profile="unconfined" name="1password" pid=775 comm="apparmor_parser"
2026-02-07T22:41:35.388862+00:00 host root: Sat Feb  7 10:41:35 PM UTC 2026: I found word, Master!
2026-02-07T22:42:35.537312+00:00 host root: Sat Feb  7 10:42:35 PM UTC 2026: I found word, Master!
root@host:~#
```
2. Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).
```
root@host:~# apt install spawn-fcgi php php-cgi php-cli apache2 libapache2-mod-fcgid -y
.....
root@host:~# mkdir /etc/spawn-fcgi
root@host:~# cat << EOF > /etc/spawn-fcgi/fcgi.conf
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"
EOF
root@host:~# ll -d /etc/spawn-fcgi/fcgi.conf
-rw-r--r-- 1 root root 331 Feb  7 23:04 /etc/spawn-fcgi/fcgi.conf
root@host:~#
root@host:~# cat << EOF > /etc/systemd/system/spawn-fcgi.service
> [Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
root@host:~# systemctl start spawn-fcgi
root@host:~# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; preset: e>
     Active: active (running) since Sat 2026-02-07 23:40:05 UTC; 1s ago
   Main PID: 12196 (php-cgi)
      Tasks: 33 (limit: 2266)
     Memory: 14.5M (peak: 14.6M)
        CPU: 142ms
     CGroup: /system.slice/spawn-fcgi.service
             ├─12196 /usr/bin/php-cgi
             ├─12198 /usr/bin/php-cgi
             ├─12199 /usr/bin/php-cgi
             ├─12200 /usr/bin/php-cgi
             ├─12201 /usr/bin/php-cgi
             ├─12202 /usr/bin/php-cgi
             ├─12203 /usr/bin/php-cgi
             ├─12204 /usr/bin/php-cgi
             ├─12205 /usr/bin/php-cgi
             ├─12206 /usr/bin/php-cgi
             ├─12207 /usr/bin/php-cgi
             ├─12208 /usr/bin/php-cgi
             ├─12209 /usr/bin/php-cgi
             ├─12210 /usr/bin/php-cgi
             ├─12211 /usr/bin/php-cgi
             ├─12212 /usr/bin/php-cgi
             ├─12213 /usr/bin/php-cgi
             ├─12214 /usr/bin/php-cgi
             ├─12215 /usr/bin/php-cgi
             ├─12216 /usr/bin/php-cgi
             ├─12217 /usr/bin/php-cgi
             ├─12218 /usr/bin/php-cgi
             ├─12219 /usr/bin/php-cgi
             ├─12220 /usr/bin/php-cgi
             ├─12221 /usr/bin/php-cgi
             ├─12222 /usr/bin/php-cgi
             ├─12223 /usr/bin/php-cgi
             ├─12224 /usr/bin/php-cgi
             ├─12225 /usr/bin/php-cgi
             ├─12226 /usr/bin/php-cgi
             ├─12227 /usr/bin/php-cgi
             ├─12228 /usr/bin/php-cgi
             └─12229 /usr/bin/php-cgi

root@host:~#
```
3. Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.
```
root@host:~# apt install nginx -y
.....
root@host:~# cat << EOF > /etc/systemd/system/nginx@.service

# Stop dance for nginx
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
# SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx-%I.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-%I.conf -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx-%I.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
EOF
root@host:~# cp /etc/nginx/nginx.conf /etc/nginx/nginx-first.conf
root@host:~# cp /etc/nginx/nginx.conf /etc/nginx/nginx-second.conf
root@host:~# nano /etc/nginx/nginx-first.conf
root@host:~# nano /etc/nginx/nginx-second.conf
root@host:~# systemctl start nginx@first
root@host:~# systemctl start nginx@second
root@host:~# systemctl status nginx@second
root@host:~# systemctl status nginx@first
```
