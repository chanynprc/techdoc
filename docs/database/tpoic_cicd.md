## CICD

### Jenkins

ECS: ssh root@39.99.234.218

ADB: gp-8vb2jcx22102f06te

#### 安装

```shell
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

yum install fontconfig java-17-openjdk
yum erase java-1.7.0-openjdk.x86_64 java-1.7.0-openjdk-devel.x86_64
yum erase java-1.8.0-openjdk.x86_64 java-1.8.0-openjdk-devel.x86_64
yum install java-11-openjdk.x86_64 java-11-openjdk-devel.x86_64

systemctl enable jenkins
systemctl --full status jenkins
systemctl start jenkins

journalctl -xe

YOURPORT=8080
PERM="--permanent"
SERV="$PERM --service=jenkins"

systemctl start firewalld
firewall-cmd $PERM --new-service=jenkins
firewall-cmd $SERV --set-short="Jenkins ports"
firewall-cmd $SERV --set-description="Jenkins port exceptions"
firewall-cmd $SERV --add-port=$YOURPORT/tcp
firewall-cmd $PERM --add-service=jenkins
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --reload


firewall-cmd --zone=public --add-port=8080/tcp --permanent
systemctl restart firewalld.service
firewall-cmd --list-ports


8b9d1168ed5647cdac44008e0316169f


getDatabaseConnection(type: 'GLOBAL')
{
def sqlString="insert into t1 (b) values (current_timestamp);"
sql sql:sqlString
}

create table t1 (a serial, b timestamp);
insert into t1 (b) values (current_timestamp);
```













