# ssh-logging-assignment
## SSH Logging as a Metric - an Assignment

This repository contain steps to install and configure SSH centralized logging, along with a script to show simple report on how many log-in were made on each host.

There are 2 kind of host used in this step, Server as the centralized log server, and Client as hosts that send the auth log to the centralized Server.

The step have been made specifically for Debian 9 (stretch) and/or 10 (buster), but it should also work on other Debian based system too, eg. Ubuntu. Other distro may have different configuration.

## Manual Process

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

### On Server host, create following script on `/usr/sbin/slogreport` to show simple report

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

### Run with `/var/log/auth.log` as an argument

```bash
slogreport /var/log/auth.log
```

### Sample output
  
```
Host aclient01 had 10 attempt, where 3 login success, 4 failed login, 0 illegal user and 3 invalid login
Host aclient02 had 5 attempt, where 2 login success, 2 failed login, 0 illegal user and 1 invalid login
Host aserver00 had 1 attempt, where 1 login success, 0 failed login, 0 illegal user and 0 invalid login
```


## Automated Process

We can automate all above processes with the help of Ansible tools. But of course, before we can use ansible to automate installation and configuration above, there are requirements that have to be solved on the target host ( Server and Client ).

On the Server and Client, the requirements for ansible to be working are openssh-server ( for remote connection ), python-minimal and python-setuptools. All of those packages are already installed by default on most distro, so we can just move forward.

On the workstation deployer, the machine where we will execute the ansible, make sure the ansible are available. You can run `pip install -r requirements.txt` inside `ansible-playbook` directory if you already have python-pip installed.

Before we run the ansible, make sure we write target machine on the inventory file at `ansible-playbook/inventory`, with following format:

```ini
[server]
server_hostname   ansible_host=server_ip_address


[client]
client_hostnameX  ansible_host=client_ip_address

```

## On deployer machine, run the `ansible-playbook` command

```bash
cd ansible-playbook && ansible-playbook -u username -Kkb ssh-logging.yaml
```

### Sample Output
```
SSH password: 
BECOME password[defaults to SSH password]: 

PLAY [SSH Logging as a Metric - an Assignment] ****************************************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************************************
ok: [aclient03]
ok: [aclient02]
ok: [aclient01]
ok: [aserver00]
ok: [aclient04]
ok: [aclient05]
ok: [aclient06]

TASK [make sure we have rsyslog installed] ********************************************************************************************************************************************************
ok: [aclient01]
ok: [aserver00]
ok: [aclient02]
ok: [aclient04]
ok: [aclient03]
ok: [aclient05]
ok: [aclient06]

TASK [allow rsyslog Server to accept remote logging - module] *************************************************************************************************************************************
skipping: [aclient01]
skipping: [aclient02]
skipping: [aclient05]
skipping: [aclient03]
skipping: [aclient06]
skipping: [aclient04]
ok: [aserver00]

TASK [allow rsyslog Server to accept remote logging - port] ***************************************************************************************************************************************
skipping: [aclient01]
skipping: [aclient02]
skipping: [aclient03]
skipping: [aclient04]
skipping: [aclient05]
skipping: [aclient06]
ok: [aserver00]

TASK [copy slogreport executable] *****************************************************************************************************************************************************************
skipping: [aclient01]
skipping: [aclient02]
skipping: [aclient03]
skipping: [aclient04]
skipping: [aclient05]
skipping: [aclient06]
ok: [aserver00]

TASK [copy sshd rsyslog template] *****************************************************************************************************************************************************************
skipping: [aserver00]
ok: [aclient01]
ok: [aclient02]
ok: [aclient03]
ok: [aclient05]
ok: [aclient04]
ok: [aclient06]

PLAY RECAP ****************************************************************************************************************************************************************************************
aclient01                  : ok=3    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
aclient02                  : ok=3    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
aclient03                  : ok=3    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
aclient04                  : ok=3    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
aclient05                  : ok=3    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
aclient06                  : ok=3    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
aserver00                  : ok=5    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
```
