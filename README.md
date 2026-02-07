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
root@host:~#

```




2. Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).
3. Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.
