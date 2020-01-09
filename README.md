```
#!/bin/bash
#安装依赖，libnuma可选
yum -y -q install libaio libnuma

#创建mysql组和用户
groupadd mysql
useradd -r -g mysql -s /bin/false mysql

#下载安装包并解压
cd /usr/local
if test ! -f mysql-5.7.27-el7-x86_64.tar.gz
then
    wget -c -q https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.27-el7-x86_64.tar.gz
fi
mysql_md5sum=$(/usr/bin/md5sum /usr/local/mysql-5.7.27-el7-x86_64.tar.gz | cut -d" " -f1)
if [ "${mysql_md5sum}" != 10799afd79b860ac46137bba50bd708c ];then
    echo "mysql安装包不完整: md5sum校验失败"
    exit 1
fi
tar zxf /usr/local/mysql-5.7.27-el7-x86_64.tar.gz
ln -s /usr/local/mysql-5.7.27-el7-x86_64 mysql

#基础安装
cd /usr/local/mysql
mkdir mysql-files   #The secure_file_priv system variable limits import and export operations to a specific directory. 
chown mysql:mysql mysql-files
chmod 750 mysql-files
cp support-files/mysql.server /etc/init.d/mysql.server

#修改服务端配置文件my.cnf
sed -i "s#log-error=.*#log-error=/var/log/mysqld.log#" /etc/my.cnf
sed -i "s#pid-file=.*#pid-file=$(grep datadir /etc/my.cnf | cut -d= -f2)/mysqld.pid#" /etc/my.cnf
sed -i "/\[mysqld_safe]/ ilog_timestamps=SYSTEM" /etc/my.cnf
sed -i "/\[mysqld_safe]/ icharacter_set_server=utf8mb4" /etc/my.cnf
sed -i "/\[mysqld_safe]/ iexplicit_defaults_for_timestamp=true" /etc/my.cnf

#修改客户端配置文件mysql-clients.cnf
sed -i "/\[mysql]/ adefault-character-set=utf8mb4" /etc/my.cnf.d/mysql-clients.cnf

#修改启动脚本mysql.server
sed -i "s#^basedir=#basedir=$(grep basedir /etc/my.cnf | cut -d= -f2)#" /etc/init.d/mysql.server
sed -i "s#^datadir=#datadir=$(grep datadir /etc/my.cnf | cut -d= -f2)#" /etc/init.d/mysql.server
sed -i "s#^mysqld_pid_file_path=#mysqld_pid_file_path=$(grep pid-file /etc/my.cnf | cut -d= -f2)#" /etc/init.d/mysql.server

#数据库初始化&启动
bin/mysqld --initialize --user=mysql 
bin/mysql_ssl_rsa_setup &>/dev/null
chown mysql:mysql "$(grep datadir /etc/my.cnf | cut -d= -f2)"/*.pem
bin/mysqld_safe --user=mysql &

#添加环境变量
echo 'export PATH=$PATH:/usr/local/mysql/bin' >> /etc/profile

#创建socker文件，避免客户端每次指定socket
ln -s "$(grep socket /etc/my.cnf | cut -d= -f2)" /tmp/mysql.sock

#设置自启动
chkconfig --add mysql.server

#记录并修改密码，以及安全加固
echo '####################################################################################################'
echo '请刷新环境变量 export PATH=/usr/local/mysql/bin:$PATH'
echo '请记录屏幕打印的密码'
echo '登录并立即修改密码 ALTER USER USER() IDENTIFIED BY '"'your_password'"';'
echo '密码修改完成以后，请执行 mysql_secure_installation 进行安全加固'
echo '####################################################################################################'
