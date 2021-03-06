# Wordpress on Nginx + PHP-FPM

This is @abdusfauzi personal steps in preparing my VPS/Dedicated server for running WordPress installation. Due to the nature of Nginx, .htaccess is not supported. We will look into configuration to imimate the how .htaccess normally works.

All files given on http://paste.laravel.com has been put into its respective files in `etc` folder above.

*Make sure to replace <username> or <website> with your own*

## Pre-configuration for DO

+ `yum install nano`
+ `yum install wget`
+ [Install git](https://www.digitalocean.com/community/articles/how-to-install-git-on-a-centos-6-4-vps)

## STEP 1: update to latest CentOS version (for DO refer [here](https://www.digitalocean.com/community/articles/initial-server-setup-with-centos-6).)
```
$ ssh root@your-ip-address
$ yum update
```

#### Check version
`cat /etc/redhat-release`

#### Setup hostname (skipping this for DO)
```
$ echo "HOSTNAME=\<yourhostname\>" >> /etc/sysconfig/network
$ hostname "\<yourhostname\>"
```

#### Update /etc/hosts (Not sure if this is necessary for DO; skipping for now)
+ `$ nano /etc/hosts`

```
add new line: \<ip address\>    \<yourhostname\>.example.com    \<yourhostname\>
add new line: \<ipv6 address\>    \<yourhostname\>.example.com    \<yourhostname\>
```

## STEP 2: get important Repo
#### Install important repo
* `rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm`
* `rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm`

#### Install yum-priorities for repo config
* `$ yum install yum-priorities`


## STEP 3: install mysql, nginx, php-fpm, memcached
####Install mysql

```
$ yum install mysql mysql-server
$ chkconfig --levels 235 mysqld on
$ service mysqld start

// Check mysqld server in running and run secure installation (to set password to root)
$ netstat -tap | grep mysql

// Set your SQL password please!
$ mysql_secure_installation
```

#### Now, lets install nginx

```
$ yum install nginx
$ chkconfig --levels 235 nginx on
$ service nginx start
$ ifconfig eth0 | grep inet | awk '{ print $2 }'
// Visit your ip address to check on nginx static page
```

#### Now, lets install php-fpm

```
$ yum install php-fpm php-cli php-mysql php-gd php-imap php-ldap php-odbc php-pear php-xml php-xmlrpc php-magickwand php-magpierss php-mbstring php-mcrypt php-mssql php-shout php-snmp php-soap php-tidy php-pecl-apc sendmail sendmail-cf
```

#### edit /etc/php.ini to set cgi.fix_pathinfo=0;

```
$ nano /etc/php.ini
```
and change; 
```
.. cgi.fix_pathinfo=0;

// OPTIONAL:-
// Edit timezone to your location (Asia/Kuala_Lumpur)
// date.timezone = "Asia/Kuala_Lumpur"
// $ ln -sf /usr/share/zoneinfo/Asia/Kuala_Lumpur /etc/localtime
```

```
$ chkconfig --levels 235 php-fpm on
$ service php-fpm start
$ chkconfig --levels 235 sendmail on
$ service sendmail start
```

#### Now, lets install memcached
```
$ yum install memcached php-memcached
$ nano /etc/sysconfig/memcached
$ > OPTIONS="-l 127.0.0.1”
$ chkconfig --levels 235 memcached on
$ service memcached start
```

### (Skipping) STEP 4: new user for FTP and SSH 
* useradd \<username\>
* passwd \<username\>
* cd /srv
* mkdir www
* cd www
* mkdir \<website\>
* mkdir \<website\>/html
* chown -R user:usergroup \<website\>

### (Skipping) STEP 5: configure nginx, php-fpm, php session to memcached
#### Edit nginx configuration file

`$ nano /etc/nginx/nginx.conf` and change these;
```
.. worker_processes    8;
.. keeplive_timeout    2;
```
#### (Skipping) Easier, follow this format http://paste.laravel.com/15PT
> Here, we set some configuration for php-fpm to run on socket. Then, we edit the default virtual host configuration
`$ nano /etc/nginx/conf.d/default.conf`

###### Follow this format http://paste.laravel.com/162K

+ Now, add those `global/… config` files

```
$ cd /etc/nginx
$ mkdir global
$ cd global
```
`$ nano restrictions.conf` =>  [http://paste.laravel.com/15PY](http://paste.laravel.com/15PY)  
`$ nano wordpress.conf` => [http://paste.laravel.com/1h8p](http://paste.laravel.com/1h8p)  
`$ nano w3-total-cache.conf` => [http://paste.laravel.com/15Q6](http://paste.laravel.com/15Q6)  
`$ service nginx restart`


#### Now, we edit php-fpm configuration

```
$ nano /etc/php-fpm.d/www.conf
.. listen = /tmp/php-fpm.sock
.. user = \<username\>
.. group = \<username\>
.. php_value[session.save_handler] = memcached
.. php_value[session.save_path] = “127.0.0.1:11211"
$ service php-fpm restart
$ service memcached restart
```

## (Skipping) STEP 6: install FTP (vsftpd)
```
$ yum install vsftpd
$ nano /etc/vsftpd/vsftpd.conf
.. anonymous_enable=NO
.. chroot_local_user=YES
.. add => user_config_dir=/etc/vsftpd/vsftpd_user_conf
.. add => use_localtime=YES
```
#### Save
```
$ mkdir /etc/vsftpd/vsftpd_user_conf
$ nano /etc/vsftpd/vsftpd_user_conf/<username>
.. dirlist_enable=YES
.. download_enable=YES
.. local_root=/srv/www/<website>
.. write_enable=YES
```
#### Save
`$ service vsftpd restart`

## STEP 7: install phpMyAdmin
`$ yum install phpmyadmin`
#### Now, create new mysql user since root has been denied. Alternatively, refer [here](https://www.digitalocean.com/community/articles/how-to-install-wordpress-with-nginx-on-ubuntu-12-04).

> Notice all sql queries must end with `;`

```
$ mysql -u root -p
$ CREATE USER <username>@localhost IDENTIFIED BY <password>;
$ GRANT ALL PRIVILEGES ON * . * TO <username>@localhost;
$ FLUSH PRIVILEGES;
exit
```

### STEP 8: Grant <username> to sudoers
* usermod -a -G wheel \<username\>
* visudo
* __Uncomment %wheel lines__
* __Add new line below root   ALL=(ALL)  ALL__
* \<username\>    ALL=(ALL)   ALL 
* __ESC key__
* :wq

### STEP 9: install WordPress files
* su \<username\>
* cd /srv/www/\<website\>/html
* wget http://wordpress.org/latest.tar.gz
* tar -xzvf latest.tar.gz
* mv wordpress/* ./
* rmdir wordpress
* rm latest.tar.gz
* __Visit website and install__

### STEP 10: install NewRelic
* rpm -Uvh http://yum.newrelic.com/pub/newrelic/el5/x86_64/newrelic-repo-5-3.noarch.rpm
* yum install newrelic-php5
* newrelic-install install
* yum install newrelic-sysmond
* nano /etc/php.d/newrelic.ini
* > newrelic.appname = "\<website name\>"
* service php-fpm restart
* service newrelic-sysmond start

###vSTEP 11: Disable root SSH login and change Port 22 to Port 215
* nano /etc/ssh/sshd_config
* > Port 215
* > PermitRootLogin no
* service sshd restart

### STEP 12: Enable firewall settings
* iptables -L
* nano /etc/iptables.firewall.rules
* __Fill up with this http://paste.laravel.com/164b__
* iptables-restore < /etc/iptables.firewall.rules
* /sbin/service iptables save
* __You will face raw nat filter error (http://blog.btnotes.com/articles/606.html)__
* mv /etc/init.d/iptables /etc/init.d/iptables.bkp
* nano /etc/init.d/iptables
* __Paste this content http://paste.laravel.com/164n__
* __SAVE__
* service iptables restart

### STEP 13: install Fail2Ban for failed login access (bruteforce)
* yum install fail2ban
* configure setting: http://www.fail2ban.org/wiki/index.php/MANUAL_0_8#Configuration

----

## References
* [Installing Nginx with PHP5 and PHP-FPM and MySQL support on Centos 6.4](http://www.howtoforge.com/installing-nginx-with-php5-and-php-fpm-and-mysql-support-on-centos-6.4)
* [How To Create A New User & Grant Permissions in MySQL](https://www.digitalocean.com/community/articles/how-to-create-a-new-user-and-grant-permissions-in-mysql)
* [Securing Your Server](https://library.linode.com/securing-your-server)
