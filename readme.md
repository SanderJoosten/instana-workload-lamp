# instana-workloads-lamp

## Update
~~~
yum update -y
~~~

## Instana setup
### Install:
~~~
yum -y install python3-pip
pip3 install instana
curl -o setup_agent.sh https://setup.instana.io/agent && chmod 700 ./setup_agent.sh && sudo ./setup_agent.sh -a lWrVCHHoSKOV1Ix8AevOeA -d lWrVCHHoSKOV1Ix8AevOeA -t dynamic -e aiops3.amsiic.ibm.com:1444 -y -s
~~~

### Configure:
~~~
#Zone
echo "# Hardware & Zone" >> /opt/instana/agent/etc/instana/configuration.yaml
echo "com.instana.plugin.generic.hardware:" >> /opt/instana/agent/etc/instana/configuration.yaml
echo " enabled: true # disabled by default" >> /opt/instana/agent/etc/instana/configuration.yaml
echo " availability-zone: 'IBM Innovation Studio / VMware ESX / LAMP'" >> /opt/instana/agent/etc/instana/configuration.yaml

#HTTPD
echo "com.instana.plugin.httpd:" >> /opt/instana/agent/etc/instana/configuration.yaml
echo " tracing:" >> /opt/instana/agent/etc/instana/configuration.yaml
echo " enabled: false" >> /opt/instana/agent/etc/instana/configuration.yaml
~~~

## HTTPD
### Install:
~~~
yum -y install httpd httpd-tools
~~~

### Configure:
~~~
#Enable status page (required for Instana)
echo "<Location /server-status>" >> /etc/httpd/conf/httpd.conf 
echo " SetHandler server-status" >> /etc/httpd/conf/httpd.conf 
echo "</Location>" >> /etc/httpd/conf/httpd.conf 
echo "LoadModule status_module lib/httpd/modules/mod_status.so" >> /etc/httpd/conf/httpd.conf 
echo "ExtendedStatus On" >> /etc/httpd/conf/httpd.conf 
echo "ServerName localhost" >> /etc/httpd/conf.d/servername.conf

mkdir -p /etc/systemd/system/httpd.service.d/
echo "[Service]" >> /etc/systemd/system/httpd.service.d/restart.conf
echo "Restart=always" >> /etc/systemd/system/httpd.service.d/restart.conf
echo "RestartSec=5s" >> /etc/systemd/system/httpd.service.d/restart.conf
sudo systemctl reload httpd
chown apache:apache /var/www/html -R

#Change default port
sed 's/^Listen 80/Listen 8080/g' /etc/httpd/conf/httpd.conf > /etc/httpd/conf/httpd.conf.new
cat /etc/httpd/conf/httpd.conf.new > /etc/httpd/conf/httpd.conf
semanage port -m -t http_port_t -p tcp 8080
~~~

### Firewall:
~~~
#Firewall
firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --permanent --zone=public --add-port=8080/tcp
firewall-cmd --permanent --zone=public --add-service=https
systemctl reload firewalld
~~~

### Enable & start:
~~~
#Start
sudo systemctl daemon-reload
systemctl restart httpd
systemctl enable httpd
~~~

## MySQL/MariaDB
### Install:
~~~
#MySQL/MariaDB
yum install mariadb-server mariadb -y
~~~

### Configure:
~~~
systemctl start mariadb
systemctl enable mariadb
mysql_secure_installation

mysql -sfu root <<EOS
-- set root password
UPDATE mysql.user SET Password=PASSWORD('ibm4all') WHERE User='root';
-- delete anonymous users
--DELETE FROM mysql.user WHERE User='';
-- delete remote root capabilities
--DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1');
-- drop database 'test'
--DROP DATABASE IF EXISTS test;
-- also make sure there are lingering permissions to it
--DELETE FROM mysql.db WHERE Db='test' OR Db='test\\_%';
-- make changes immediately
FLUSH PRIVILEGES;
EOS
~~~

## PHP
### Install:
~~~
yum -y install php php-fpm php-mysqlnd php-opcache php-gd php-xml php-mbstring
~~~

### Configure:
~~~
echo "pm.status_path = /status" >> /etc/php-fpm.d/www.conf 
systemctl start php-fpm
systemctl enable php-fpm
systemctl enable php-fpm
setsebool -P httpd_execmem 1
setsebool -P httpd_can_network_connect on
~~~

## Create test PHP workload
~~~
echo "<?php phpinfo(); ?>" > /var/www/html/info.php
echo "SetHandler application/x-httpd-php" >> /etc/httpd/conf/httpd.conf 
systemctl restart httpd
~~~