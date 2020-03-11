# Log parser via crontab

```bash
#!/bin/bash

DBA1=admin1@hostname.localdomain
DBA2=admin2@hostname.localdomain
DBA3=admin3@hostname.localdomain
DBA4=admin4@hostname.localdomain

NOTIFY_MAILS="$DBA1,$DBA2,$DBA3,$DBA4"

notify() {
echo -e "$1" | mailx -s "Notification" $NOTIFY_MAILS
exit
}

SCRIPT_NAME=$0
SCRIPT_PATH=`readlink -f $0`
SCRIPT_DIR=`dirname $SCRIPT_PATH`

[ ! -d "$SCRIPT_DIR" ] && notify "dir '$SCRIPT_DIR' not exists.."

FILT=$SCRIPT_DIR/.$SCRIPT_NAME.filtered
COMP=$SCRIPT_DIR/.$SCRIPT_NAME.compared

[ -d "$FILT" ] && notify "'$FILT' cannot be dir.."
[ -d "$COMP" ] && notify "'$COMP' cannot be dir.."

touch $FILT $COMP 2>/dev/null

[ "$?" != "0" ] && notify "permission denied?\n - $FILT\n - $COMP"

DEFAULT_KEYWORDS_1="err|crit|fail|warn|alert|emerg|denied|deny"
DEFAULT_KEYWORDS_2="unread|unreachable|reject|missing|problem"

DEFAULT_KEYWORDS="$DEFAULT_KEYWORDS_1|$DEFAULT_KEYWORDS_2"

# BEGIN ######################

VARLOG_DIR=/var/log

VARLOG_KW_KERN="$DEFAULT_KEYWORDS"
VARLOG_KW_BOOT="$DEFAULT_KEYWORDS"
VARLOG_KW_AUTH="password check failed|authentication failure|$DEFAULT_KEYWORDS"
VARLOG_KW_DPKG="upgrade|install|purge|remove|$DEFAULT_KEYWORDS"

tail -25000 $VARLOG_DIR/kern.log 2>/dev/null | grep -iE "$VARLOG_KW_KERN" >> $FILT
tail -25000 $VARLOG_DIR/boot.log 2>/dev/null | grep -iE "$VARLOG_KW_BOOT" >> $FILT
tail -25000 $VARLOG_DIR/auth.log 2>/dev/null | grep -iE "$VARLOG_KW_AUTH" >> $FILT
tail -25000 $VARLOG_DIR/dpkg.log 2>/dev/null | grep -iE "$VARLOG_KW_DPKG" >> $FILT

# END ######################

RES=`diff $FILT $COMP`

cat $FILT > $COMP

rm -f $FILT

[ -n "$RES" ] && echo -e "$RES" | mailx -s "Notification" $NOTIFY_MAILS
```
