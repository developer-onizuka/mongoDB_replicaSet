# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure("2") do |config|
  config.vm.box = "centos/8"
  config.vm.provider :libvirt do |kvm|
    kvm.memory = 16384
    kvm.cpus = 2
  end
#-------------------- master --------------------#
  config.vm.define "ops-manager_192.168.33.12" do |server|
    server.vm.network "private_network", ip: "192.168.33.12"
    server.vm.network "private_network", ip: "192.168.133.12"
    server.vm.hostname = "ops-manager"
    server.vm.provision "shell", privileged: true, inline: <<-SHELL
      sed -i -e 's/PasswordAuthentication\ no/PasswordAuthentication\ yes/g' /etc/ssh/sshd_config
      systemctl restart sshd
      echo "sudo ip link set eth1 mtu 1450" >> /etc/rc.local
      cat << EOF > /etc/yum.repos.d/mongodb-org-5.0.repo  
[mongodb-org-5.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/8/mongodb-org/5.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-5.0.asc
EOF
      dnf install -y mongodb-org
      sudo mkdir -p /data/appdb
      sudo chown -R mongod:mongod /data
      cat <<EOF > /etc/mongod_appdb.conf
systemLog:
  destination: file
  path: "/data/appdb/mongodb.log"
  logAppend: true
storage:
  dbPath: "/data/appdb"
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
processManagement:
  fork: true
  timeZoneInfo: /usr/share/zoneinfo
  pidFilePath: /var/run/mongodb/mongod.pid
net:
  bindIp: 127.0.0.1
  port: 27017
setParameter:
  enableLocalhostAuthBypass: false
EOF
    sudo -u mongod mongod -f /etc/mongod_appdb.conf
    dnf install -y wget
    dnf install -y net-tools
    #wget https://downloads.mongodb.com/on-prem-mms/rpm/mongodb-mms-5.0.3.100.20211005T2044Z-1.x86_64.rpm
    wget https://downloads.mongodb.com/on-prem-mms/rpm/mongodb-mms-5.0.6.100.20220114T1308Z-1.x86_64.rpm
    #sudo rpm -ivh mongodb-mms-5.0.3.100.20211005T2044Z-1.x86_64.rpm
    sudo rpm -ivh mongodb-mms-5.0.6.100.20220114T1308Z-1.x86_64.rpm 
    SHELL
  end
end
