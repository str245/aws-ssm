# Over SSM
Quick setup and usage guide for SSH access over SSM to private AWS EC2 instances

### - N O T E - 
This guide is not intended to be a step-by-step howto. It is a conceptual explanation of how to use SSM
 to connect to AWS instances or services (SSH, SCP, SOCKs, DB). Requires basic prior knowledge of AWS 
 and Bash (Or any other programming language), in short, a Devops profile.

### Features:

- Can connect to your private instances inside your VPC without jumping through a public-facing bastion.
- Don't need to store any SSH keys locally or on the server.
- Only require necessary IAM Role (+cli credentials) and ability to reach their regional SSM endpoint (via HTTPS).
- It's unlikely to find yourself blocked by network-level security, making it a great choice if you need to get out to the internet from inside a restrictive or private network

### Requirements
- [python3](https://www.python.org/about/gettingstarted/) (ONLY, not 2.x or 3.x Emulator)
- [awscli v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) (ONLY v2)
- [session-manager-plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)

### Installation
1) Clone
2) Place scripts in a familiar local directory e.g. `~/bin/` and add to PATH -> `echo "export PATH="$HOME/bin${PATH:+:${PATH}}""| tee -a ~/.bashrc` (or `~/.bash_profile`)
3) Install required python modules  -> `pip3 install --user -r /path/to/ssm-tool/requirements.txt`
4) Add snippet to SSH config (see below)
5) macOS users may need to install newer versions of `bash` and `openssh` with `brew install`

### SSH config
Copy and paste the following snippet to the top of your SSH config file (`~/.ssh/config`) **or** add to the bottom and remove any other config matching against host `i-*`:
```
Match exec "grep -qs '^Host.*%n' %d/.ssh/ssmtool-*"
  Include ssmtool-*

Match Host i-*
  ProxyCommand ssh-ssm.sh %h %r
  IdentityFile ~/.ssh/ssm-ssh-tmp
  StrictHostKeyChecking no
  PasswordAuthentication no
  ChallengeResponseAuthentication no
  ```

##  Usage
**Listing instances**
```
[devops@testbox ~]$ ssm-tool --profile home-dev
+--------------------------+---------------------+---------------+------------+--------------+
| tag[name]                | instance            | ip address    | ssm-agent* | platform     |
+--------------------------+---------------------+---------------+------------+--------------+
| home-dev-jumpbox-01      | i-0xxxxxxxxxxxx79d6 | 10.xxx.24.9   | True       | Amazon Linux |
| home-dev-confluenceasg   | i-0xxxxxxxxxxxx9007 | 10.xxx.24.1xx | False      | CentOS Linux |
| home-dev-bambooasg       | i-0xxxxxxxxxxxx29b9 | 10.xxx.24.2xx | False      | CentOS Linux |
| home-dev-jiraasg         | i-0xxxxxxxxxxxxc331 | 10.xxx.24.2xx | False      | CentOS Linux |
+--------------------------+---------------------+---------------+------------+--------------+
 * ssm-agent column refers to whether the agent is up-to-date
```
**Update ssm-agent on all instances (if need)**
```
[devops@testbox ~]$ ssm-tool --profile home-dev --update
success
```

**Connecting to an instance over SSH using `ssm-tool` and instance id:**
```
[devops@testbox ~]$ ssm-tool --profile home-dev --ssh centos@i-0xxxxxxxxxxxx29b9
Last login: Fri May  8 10:54:38 2020 from localhost
[centos@ip-10-xxx-24-2xx ~]$ sudo -i
[root@ip-10-xxx-24-2xx ~]#
[root@ip-10-xxx-24-2xx ~]# logout
[centos@ip-10-xxx-24-2xx ~]$ logout
Connection to i-0xxxxxxxxxxxx29b9 closed.
```

**Using ssm-tool to generate and configure SSH, then using ssh directly to connect:**

Generate config:
```
[devops@testbox ~]$ ssm-tool --profile home-dev --ssh-conf

ssh config fragment generated and saved to -> /home/devops/.ssh/ssmtool-home-dev
```
Connect over SSH to the jumpbox host using name[tag]:
```
[devops@testbox ~]$ ssh home-dev-jumpbox-01

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
74 package(s) needed for security, out of 154 available
Run "sudo yum update" to apply all updates.
[ec2-user@ip-10-xxx-24-9 ~]$ logout
Connection to i-0xxxxxxxxxxxx79d6 closed.
```
Connect over SSH to the confluence host using IP address:
```
[devops@testbox ~]$ ssh 10.xxx.24.1xx
[centos@ip-10-xxx-24-1xx ~]$ logout
Connection to i-0xxxxxxxxxxxx9007 closed.
```
Connect over SSH to the bamboo host using short hostname: 
```
[devops@testbox ~]$ ssh ip-10-xxx-24-2xx.ap-southeast-2
[centos@ip-10-xxx-24-2xx ~]$ logout
Connection to i-0xxxxxxxxxxxx29b9 closed.
```
**Note:** Feel free to add other names or change the username in the generated SSH config fragment

**Don't need SSH? Connect to an instance over SSM session using instance id:**
```
[devops@testbox ~]$ ssm-tool --profile home-dev --session i-0xxxxxxxxxxxx29b9

Starting session with SessionId: example123-0e467c6bf9f9ae39d
sh-4.2$ sudo -i
[root@ip-10-xxx-24-2xx ~]# logout
sh-4.2$ exit

Exiting session with sessionId: example123-0e467c6bf9f9ae39d.
```
### Examples by Service
SCP:
```
[devops@testbox ~]$ scp ~/bin/ssh-ssm.sh bitbucket-prod.personal:~
ssh-ssm.sh                                                                                       100%  366    49.4KB/s   00:00

[devops@testbox ~]$ ssh bitbucket-prod.personal ls -la ssh\*
-rwxrwxr-x 1 ec2-user ec2-user 366 Jun 04 07:27 ssh-ssm.sh
```

SOCKS:
```
[devops@testbox ~]$ ssh -f -nNT -D 8080 jira-prod.personal
[devops@testbox ~]$ curl -x socks://localhost:8080 ipinfo.io/ip
54.xxx.xxx.49
[devops@testbox ~]$ whois 54.xxx.xxx.49 | grep -i techname
OrgTechName:   Amazon EC2 Network Operations
```

DB tunnel:
```
[devops@testbox ~]$ ssh -f -nNT -oExitOnForwardFailure=yes -L 5432:db1.host.internal:5432 jira-prod.personal
[devops@testbox ~]$ ss -lt4p sport = :5432
State      Recv-Q Send-Q Local Address:Port                 Peer Address:Port
LISTEN     0      128       127.0.0.1:postgres                        *:*                     users:(("ssh",pid=26130,fd=6))
[devops@testbox ~]$ psql --host localhost --port 5432
Password:
```

SSH (with minimum required configuration):
```
[devops@testbox ~]$ jumpbox=$(aws --profile atlassian-prod ec2 describe-instances --filters 'Name=tag:Name,Values=confluence-prod' --output text --query 'Reservations[*].Instances[*].InstanceId')
[elpy@testbox ~]$ echo ${jumpbox}
i-0fxxxxxxxxxxxxe28
[elpy@testbox ~]$ AWS_PROFILE=atlassian-prod ssh ec2-user@${jumpbox}
Last login: Sat Jan 25 08:59:40 2020 from localhost

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-10-xx-x-x06 ~]$ logout
Connection to i-0fxxxxxxxxxxxxe28 closed.
```
