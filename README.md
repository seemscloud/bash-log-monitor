# Log parser

## How works
 * **initialization step** - first execution will grab all keywords, probably huge amount of text and send notification
 * **progress step** - every next execution will grab just new keywords
 
## Importants
 * `.compared`, .filtered created inside the same directory like the script as hidden file
 * deletion of `.compared` file will start initialization again on next ececution
 * `.filtered` file is a temporary file, removed every end of script


## Requirements
 * `mailx` or `sendmail` client
 * configured service `sendmail` or `postfix`
 * `unix2dos`, `dos2unix`

## Tested on
 - Red Hat Enterprise Linux 6 / 7 / 8
 - CentOS 6 / 7 / 8
 - Oracle Linux 6 / 7 / 8
 - Debian 8 / 9 / 10
 - Ubuntu 14 / 16 / 18

# With validation
```bash
#!/bin/bash

notify() {
  [ -n "$3" ] && SUBJECT="$3" || SUBJECT="Cron Error"
  echo -e "$1" # | mailx -s "$SUBJECT" "$2" # unix2dos/dos2unix
  exit
}

MAX_SIZE="10240000"

DBA1="admin1@hostname.localdomain"
# DBA2="admin2@hostname.localdomain"
# DBA3="admin3@hostname.localdomain"
# DBA4="admin4@hostname.localdomain"

MAILS="$DBA1,$DBA2,$DBA3,$DBA4"

if [ "$#" != 2 ] ; then
  RES="/bin/bash $0 abs_dir_path filename_regex\r\n"
  notify "$RES" "$MAILS"
fi

LOG_DIR="`readlink -f $1`"
LOG_FILE="`echo "$2" | sed "s/\///g"`"

if [ ! -d "$LOG_DIR" ] ; then
  RES="'$LOG_DIR' must exist as dir\r\n"
  notify "$RES" "$MAILS"
fi

if [ -d "$LOG_DIR/$LOG_FILE" ] ; then
  RES="'$LOG_DIR/$LOG_FILE' cannot be dir\r\n"
  notify "$RES" "$MAILS"
fi

FILENAME="`basename "$0"`"
FILE_PATH="`readlink -f "$0"`"
DIR_PATH="`dirname "$FILE_PATH"`"

FILT="$DIR_PATH/.$FILENAME.$LOG_FILE.filtered"
COMP="$DIR_PATH/.$FILENAME.$LOG_FILE.compared"

if [ -d "$FILT" ] || [ -d "$COMP" ] ; then
  RES="Cannot be dir (one of is):\r\n - '$FILT'\r\n - '$COMP'\r\n"
  notify "$RES" "$MAILS"
fi

touch "$FILT" "$COMP" 2>/dev/null

if [ "$?" != "0" ] ; then
  RES="Could not 'touch' (one of):\r\n - '$FILT'\r\n - '$COMP'\r\n"
  notify "$RES" "$MAILS"
fi

KW1="err|crit|fail|warn|alert|emerg|denied|deny"
KW2="unread|unreach|miss|problem|block|terminat|exclude"
KW3="reject|inject|eject|remove|purge|clean|clear|close"
KW4="password check failed|authentication failure"

KEYWORDS="$KW1|$KW2|$KW3|$KW4"

find "$LOG_DIR" -mindepth 1 -maxdepth 1 -type f -name "$LOG_FILE" \
        -exec grep -aiE "$KEYWORDS" {} 2>/dev/null \; >> "$FILT"

RES="`diff "$FILT" "$COMP"`"

cat "$FILT" > "$COMP"

rm -f "$FILT"

SUBJ="Found Keywords! ($LOG_FILE)"

if [[ "`echo $RES | wc -c`" -gt "$MAX_SIZE" ]] ; then
  RES="Result too large ($COMP)\r\n"
fi

[ -n "$RES" ] && notify "$RES" "$MAILS" "$SUBJ"
```

# Example install

## /root/crontab.d/log-monitor.cron
```bash
*/5 * * * * /bin/bash /root/crontab.d/log-monitor/generic.sh /var/log yum.log
*/5 * * * * /bin/bash /root/crontab.d/log-monitor/generic.sh /var/log kern.log
*/5 * * * * /bin/bash /root/crontab.d/log-monitor/generic.sh /var/log maillog
*/5 * * * * /bin/bash /root/crontab.d/log-monitor/generic.sh /var/log daemon.log
*/5 * * * * /bin/bash /root/crontab.d/log-monitor/generic.sh /var/log cron*
*/5 * * * * /bin/bash /root/crontab.d/log-monitor/generic.sh /var/log secure*

*/5 * * * * /bin/bash /root/crontab.d/log-monitor/httpd.sh /var/log/httpd app_access.log*
```

### `generic.sh` vs `httpd.sh`
 There is just one difference between this files, different keywords
 
`generic.sh`
```bash
# ...
 
KW1="err|crit|fail|warn|alert|emerg|denied|deny|"
KW2="unread|unreach|miss|problem|block|terminat|exclude"
KW3="reject|inject|eject|remove|purge|clean|clear|close"
KW4="password check failed|authentication failure"

KEYWORDS="$KW1|$KW2|$KW3|$KW4"

# ...
```
 
`httpd.sh`
```bash
# ...
 
KEYWORDS=" 5[0-9]{2}"

# ... 
```

# Example Postfix Config
```
POSTFIX_REPLAYHOST="127.0.0.1"
POSTFIX_MYHOSTNAME="`hostname -f`"

MAIL_ENV="Production"
MAIL_HOST="host1"

postconf -e "myhostname = $POSTFIX_MYHOSTNAME"
postconf -e "mydomain = $myhostname"
postconf -e "myorigin = $mydomain"
postconf -e "inet_protocols = ipv4"
postconf -e "relayhost = $POSTFIX_REPLAYHOST"
postconf -e "smtp_generic_maps = hash:/etc/postfix/generic"
postconf -e "smtp_header_checks = regexp:/etc/postfix/header_checks"

cat >/etc/postfix/generic << EndOfMessage
root@`hostname -f`    root@`hostname -f`
zabbix-agent@`hostname -f`    zabbix-agent@`hostname -f`
zabbix@`hostname -f`    zabbix@`hostname -f`
EndOfMessage

chmod 644 /etc/postfix/generic

cat >/etc/postfix/header_checks  << EndOfMessage
/^From: root@`hostname -f`/ REPLACE From: "$MAIL_ENV ($MAIL_HOST)" <root@`hostname -f`>
/^From: zabbix-agent@`hostname -f`/ REPLACE From: "$MAIL_ENV ($MAIL_HOST) Zabbix Agent" <zabbix-agent@`hostname -f`>
/^From: zabbix@`hostname -f`/ REPLACE From: "$MAIL_ENV ($MAIL_HOST) Zabbix" <zabbix@`hostname -f`>
EndOfMessage

chmod 644 /etc/postfix/header_checks

rm -f /etc/postfix/generic.db

postmap /etc/postfix/generic
postmap -q "root@`hostname -f`" /etc/postfix/generic

service postfix restart && sleep 2 && echo "Message" | mailx -s "Subject" root@hostname.localdomain
```
