# ssh-logging-assignment
## SSH Logging as a Metric - an Assignment

This repository contain steps to install and configure SSH centralized logging, along with a script to show simple report on how many log-in were made on each host.

There are 2 kind of host used in this step, Server as the centralized log server, and Client as hosts that send the auth log to the centralized Server.

The step have been made specifically for Debian 9 (stretch) and/or 10 (buster), but it should also work on other Debian based system too, eg. Ubuntu. Other distro may have different configuration.

> **NOTE:**
>
> All commands below are in root privileges, use prefix `sudo` on ecah command if you want to run as other user.

### Make sure all node has rsyslog installed

```bash
apt-get --no-install-recommends install rsyslog
```

### On Server host, configure the rsyslog to accept logging from other system

```bash
sed -i '/imudp/ s/^#//' /etc/rsyslog.conf
```

### On Server host, restart the rsyslog service

```bash
systemctl restart rsyslog
```

### On Client host, configure rsyslog to send sshd related log to Server host

```bash
cat > /etc/rsyslog.d/sshd.conf <<EOF
if \$programname == 'sshd' then @ip.remote.rsyslog.server
& ~
EOF
```

### On Client host, restart the rsyslog service

```bash
systemctl restart rsyslog
```

### On Server host, create following script on /usr/sbin/slogreport to show simple report

```bash
#!/bin/bash

sshd_log_file=${1}

if [ -r ${sshd_log_file} ]; then
    sshd_log=$(egrep " sshd\[[0-9]+\]: (Accepted|Failed|Illegal|Invalid) " ${sshd_log_file})

    IFS=$'\n' read -r -d '' -a hosts < <(awk '{print $4}' <<<${sshd_log} | sort -u && printf '\0')

    for host in "${hosts[@]}"; do
        IFS=$'\n' read -r -d '' -a attempts < <(egrep " ${host} sshd\[[0-9]+\]: " <<<${sshd_log} | tee >(grep -c ": Invalid") >(grep -c ": Illegal") >(grep -c ": Failed") >(grep -c ": Accepted") >/dev/null | tee && printf '\0')
        total=$((attempts[0] + attempts[1] + attempts[2] + attempts[3]))
        echo "Host ${host} had ${total} attempt, where ${attempts[0]} login success, ${attempts[1]} failed login, ${attempts[2]} illegal user and ${attempts[3]} invalid login"
    done
fi

```

### Dont forget to make the script executable

```bash
chmod +x /usr/sbin/slogreport
```

### Run with /var/log/auth.log as an argument

```bash
slogreport /var/log/auth.log
```

### Sample output
  
```
Host aclient01 had 10 attempt, where 3 login success, 4 failed login, 0 illegal user and 3 invalid login
Host aclient02 had 5 attempt, where 2 login success, 2 failed login, 0 illegal user and 1 invalid login
Host aserver00 had 1 attempt, where 1 login success, 0 failed login, 0 illegal user and 0 invalid login
```
