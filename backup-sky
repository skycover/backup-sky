#!/bin/sh

# LICENSE
# This software is developed at SkyCover company
# for the "SkyCover Backup" cloud service.
# http://www.skycover.ru/skycover-backup
# The origin code location is
# http://github.com/skycover/backup-sky
# This software is licensed under GPLv3
# Author: Dmitry Chernyak, dmi@skycover.ru, 2011
# 
# DISCLAIMER
# We are not responsible for any effects of using this software.
#
# DESCRIPTION
# Technically this is the wrapper around duplicity (http://duplicity.nongnu.org/)
# to simplyfy config-based backups.
# Go http://github.com/skycover/backup-sky for README and instructions

#########################################################
# System names
#########################################################
pname=backup-sky
etc=/etc
var=/var

# last config update version
BKPSKY_CONF_MAJOR=1
BKPSKY_CONF_MINOR=2

#########################################################
# Default settings. Will be overriden by env file
#########################################################
BKPSKY_CURENT_VERSION=1.2
ACCOUNT="newuser"
SERVER="nonexistent.skycover.ru"
BKPSKY_SERVER="ssh://\$ACCOUNT@\$SERVER"
BKPSKY_IDENTITY="$HOME/.ssh/id_rsa"
BKPSKY_SIGNKEY=""
PASSPHRASE="secret"
BKPSKY_VOLSIZE=50
BKPSKY_SSH_OPTS=""
BKPSKY_LOG=$var/log/$pname
BKPSKY_CACHE=$var/cache/$pname

#########################################################
# Configuration upgrade procedure
#########################################################
write_env() {
cat <<EOF >$etc/$pname/env
# Software version
BKPSKY_VERSION=$BKPSKY_CURENT_VERSION

# Your account's name
ACCOUNT="$ACCOUNT"

# The backup server
SERVER="$SERVER"

# Full backup path
# You can also use file:///path/to/somewhere
BKPSKY_SERVER="$BKPSKY_SERVER"

# SSH key to access service
# Generate with: ssh-keygen -t rsa -b 2048
BKPSKY_IDENTITY="$BKPSKY_IDENTITY"

# GPG key ID to sign your data (this protect integrity)
# key should be an 8 character hex string, like AA0E73D2
BKPSKY_SIGNKEY="$BKPSKY_SIGNKEY"

# Passphrase to encrypt your data. MUST be non-dictionary
# and at least 16 characters long.
PASSPHRASE="$PASSPHRASE"

# Volume size in Mb
# With large uploads and good channels use greater values
BKPSKY_VOLSIZE=$BKPSKY_VOLSIZE

# Additional SSH connection options
BKPSKY_SSH_OPTS="$BKPSKY_SSH_OPTS"

# Here log files will be placed for each target
BKPSKY_LOG="$BKPSKY_LOG"

# Here signature files will be stored
BKPSKY_CACHE="$BKPSKY_CACHE"
EOF
}

#########################################################
# Self-check and fresh installation
#########################################################
test -d $etc/$pname || mkdir  $etc/$pname
chmod 700 $etc/$pname

if [ ! -f $etc/$pname/env ]; then
  write_env
  echo Default environment was created.
  echo You should edit $etc/$pname/env with your settings
  ex=1
else
  . $etc/$pname/env

  if [ -z "$BKPSKY_VERSION" ]; then
    BKPSKY_MAJOR=0
    BKPSKY_MINOR=0
  else
    BKPSKY_MAJOR=`echo $BKPSKY_VERSION|cut -d. -f1`
    BKPSKY_MINOR=`echo $BKPSKY_VERSION|cut -d. -f2`
  fi
  # WARNING: no way to downgrade in the check!
  if [ "$BKPSKY_MAJOR" -lt "$BKPSKY_CONF_MAJOR" -o "$BKPSKY_MINOR" -lt "$BKPSKY_CONF_MINOR" ]; then
    write_env
    echo Environment was updated with new version.
    ex=1
  fi
fi

if [ ! -f $etc/$pname/targets ]; then
  cat <<EOF >$etc/$pname/targets
# Te backup targets specification
#
# Comment strings has the leading hash+space
# Target format:
# * The first word: target's name
# * Other words: Folder names and duplicity arguments like exclusions etc.
# * The rest after the target's name (if exists) is assigned to \$PFX variable
# * The "-" (minus) sign as the first word means continuation of the arguments
# on the new line
# * QUIT word is mandatory at the end
#
# Support scripts:
# When the target is backed up, then $etc/$pname/scripts/TARGET.[pre|post]
# will be executed around the backup comand, to properly maintain the services
# If .pre script set \$skip_backup variable, then the rest of this target
# will not be executes
#
# Examples (man duplicity):
#
# etc /etc
# usr /usr
# - --include \$PFX/local/bin
# - --exclude \$PFX/local
#
# mysql /var/lib/mysql
QUIT
EOF
  echo Default targets file was created.
  echo You should edit $etc/$pname/targets with your backup targets
  ex=1
fi

# Covering possible security issues
chmod 600 $etc/$pname/targets
chmod 600 $etc/$pname/env

################################
# Installation: Script templates
################################
BKPSKY_SCRIPTS=$etc/$pname/scripts
mkdir -p $BKPSKY_SCRIPTS
chmod 700 $etc/$pname/scripts

if [ ! -f "$BKPSKY_SCRIPTS/mysql.pre" ]; then
cat <<EOF >"$BKPSKY_SCRIPTS/mysql.pre"
if [ \`ps ax|grep mysqld|grep -v grep|wc -l\` -eq 0 ]; then
  mysql_stopped=yes
  echo "WARNING: MySQL is not running."
else
  /etc/init.d/mysql stop
  sleep 5
  if [ \`ps ax|grep mysqld|grep -v grep|wc -l\` -gt 0 ]; then
    echo "ERROR: mysql is not stopped. Skiping backup"
    skip_backup=yes
  fi
fi
EOF
fi
if [ ! -f "$BKPSKY_SCRIPTS/mysql.post" ]; then
cat <<EOF >"$BKPSKY_SCRIPTS/mysql.post"
if [ -z "\$mysql_stopped" ]; then
  /etc/init.d/mysql start
  sleep 5
  test \`ps ax|grep mysqld|grep -v grep|wc -l\` -eq 0 && echo "ERROR: MySQL is not restarted properly. Take care."
fi
EOF
fi

#########################################################
# Real production code starts here
#########################################################

# Personalization check
if [ "$PASSPHRASE" = "secret" -o "$ACCOUNT" = "newuser" -o "$SERVER" = "nonexistent.skycover.ru" ]; then
  echo Please edit the backup settings in $etc/$pname
  ex=1
fi

test -n "$ex" && exit 1

mkdir -p $BKPSKY_LOG
mkdir -p $BKPSKY_CACHE

#########################################################
# Technological variables
#########################################################
BKPSKY_ARGS="--archive-dir $BKPSKY_CACHE"
if [ -n "$BKPSKY_IDENTITY" ]; then
  #BKPSKY_SSH_OPTS="$BKPSKY_SSH_OPTS -oIdentityFile=$BKPSKY_IDENTITY"
  BKPSKY_SSH_OPTS="-oIdentityFile=$BKPSKY_IDENTITY"
fi
if [ -n "$BKPSKY_SSH_OPTS" ]; then
  BKPSKY_ARGS="$BKPSKY_ARGS --ssh-options=\"$BKPSKY_SSH_OPTS\""
fi
if [ -n "$BKPSKY_SIGNKEY" ]; then
  BKPSKY_ARGS="$BKPSKY_ARGS --sign-key=$BKPSKY_SIGNKEY"
fi
if [ -n "$BKPSKY_VOLSIZE" ]; then
  BKPSKY_ARGS="$BKPSKY_ARGS --volsize=$BKPSKY_VOLSIZE"
fi
export PASSPHRASE

#########################################################
# The backup code
#########################################################
cmd=$1

if [ -z "$cmd" ]; then
  exec $0 help
  exit 1
fi

if [ -n "$2" -a "$2" != "all" ]; then
  target="$2"
fi
test -n "$2" && shift
shift

do_backup() {
  echo Backing up target: $NAME
  test -f "$BKPSKY_SCRIPTS/$NAME.pre" && . "$BKPSKY_SCRIPTS/$NAME.pre"
  test -z "$skip_backup" && eval duplicity $method $SOURCE $BKPSKY_SERVER/$NAME --name $NAME $BKPSKY_ARGS|\
    tee -a $BKPSKY_LOG/$NAME.log
  test -f "$BKPSKY_SCRIPTS/$NAME.post" && . "$BKPSKY_SCRIPTS/$NAME.post"
}

do_status() {
  echo Status of target: $NAME
  duplicity collection-status $BKPSKY_SERVER/$NAME --name $NAME $BKPSKY_ARGS
}

do_list() {
  echo Contents of target: $NAME
  duplicity list-current-files $BKPSKY_SERVER/$NAME --name $NAME $BKPSKY_ARGS
}

do_restore() {
    echo Restore target: $NAME
    duplicity restore $BKPSKY_SERVER/$NAME --name $NAME $BKPSKY_ARGS $DEST
}

do_targets() {
    echo Target: $NAME
}

exec_func() {
  if [ -z "$1" ]; then
    echo "Internal error: loop func is not defined"
    exit 1
  fi
  loop_func=$1
  if [ -n "$NAME" -a -z "$backup_done" -a -z "$skip_backup" ]; then
    $loop_func
    backup_done="yes"
  fi 
}

main_loop() {
  cat $etc/$pname/targets|while read WORD REST; do
    case "$WORD" in
    "#")
      # comment
    ;;
    "")
      # empty line
    ;;
    "-")
      # continue arguments
      SOURCE="$SOURCE $REST"
    ;;
    "QUIT")
      exec_func $1
      exit
    ;;
    *)
      # target head
      # execute previous backup if not yet. it's safe
      exec_func $1
      NAME="$WORD"
      SOURCE="$REST"
      PFX="$SOURCE"
      backup_done=""
      if [ -n "$target" -a "$target" != "$NAME" ]; then
	skip_backup=yes
      else
	skip_backup=""
      fi
    ;;
    esac
  done
}

case $cmd in
backup)
  method=$@
  main_loop do_backup
;;
status)
  main_loop do_status
;;
list)
  if [ -z $target ]; then
    echo Can list only particular target. Please specify.
    exit 1
  fi
  main_loop do_list
;;
restore)
  # We won't restore files on all targets at once
  if [ -z $target ]; then
    echo Can restore only particular target. Please specify.
    exit 1
  fi
  DEST=$1
  if [ -z "$DEST" ]; then
    echo destination path expected for restore
    exit 1
  fi
  main_loop do_restore
;;
targets)
  main_loop do_targets
;;
*)
  cat <<EOF
Usage $0 <backup|status|list|restore|targets> [[TARGET|all] [full|inc] [duplicity args]]

backup TARGET - backup target
    The full backup is maiden first.
    Then incremental backups are created (unless "full" is specified)

status TARGET - show collection status

list TARGET - list files in the backup
    TARGET must be specified exactly

restore TARGET [directory] - restore TARGET into directory
    TARGET must be specified exactly

targets - list all target names

TARGET - the target name from $etc/$pname/targets
    "all" means all defined targets
    void TARGET assumes "all"
    "mysql-full" is treated with mysql start/stop scripts upon backup

The software origin is http://github.com/skycover/backup-sky/
EOF

  exit 1
;;
esac
