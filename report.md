# Báo cáo về việc thực hành triển khai Cluster Website **(WordPress)**

#### I, Mô hình triển khai.
Ở đây, chúng em triển khai mô hình gồm có 2 Node làm HAproxy, KeepAlive, Failover và 3 Node dùng để đặt Wordpress cùng DB cluster.

 * WP1	10.0.0.138	WP+DB cluster
 * WP2	10.0.0.139	WP+DB cluster
 * WP3	10.0.0.140	WP+DB cluster
 * HA1		10.0.0.135	HAproxy+KeepAlive+LB
 * HA3 	10.0.0.136	HAproxy+KeepAlive+LB

Ta cấu hình trên cả 5 máy 2 file /etc/hosts và /etc/hostname
Cấu hình lại file */etc/hostsname*
```
sudo echo <node name> > /etc/hostsname
```
![]()
Cấu hình lại file */etc/hosts* thêm các dòng sau
```
10.0.0.138      WP1
10.0.0.139      WP2
10.0.0.140      WP3
10.0.0.135      HA1
10.0.0.136      HA2
```
![]()

II, Cấu hình các máy chứa Wordpress và DB Cluster.
-------------------------------------------------------------------
Ở đây ta sử dụng 2 cách để đồng bộ dữ liệu trong DB cluster là Percona XtraDB hoặc Galera MySQL Cluster.

***1-1, Cấu hình DB cluster sử dụng Percona XtraDB***

***1.1, Cài đặt My SQL Multi Cluster - Percona Extra DB***

Để tránh việc xung đột với cài đặt bạn cần gỡ bỏ MySQL để tránh bị các lỗi không cần thiết. Dùng câu lệnh sau:

```
sudo apt-get remove apparmor
```
b, Cài đặt các gói bổ trợ và update hệ điều hành
```
wget https://repo.percona.com/apt/percona-release_0.1-4.$(lsb_release -sc)_all.deb
sudo dpkg -i percona-release_0.1-4.$(lsb_release -sc)_all.deb
sudo apt-get update
```
c, Cài đặt gói Percona XtraDB Cluster server
```
sudo apt-get install percona-xtradb-cluster-56
```

***1.2, Cấu hình từng node***

**Chú ý**: Sau khi cài đặt  Percona thì dịch vụ Mysql tự động được bật, chúng ta cần tắt dịch vụ trên cả 3 node đi bằng lệnh:
``` 
/etc/init.d/mysql stop
```

**Node 1**
Cấu hình lại file */etc/mysql/my.cnf* và thêm nội dung sau:
```
sh
[mysqld]

datadir=/var/lib/mysql
user=mysql

# Path to Galera library
wsrep_provider=/usr/lib/libgalera_smm.so

# Cluster connection URL contains the IPs of node#1, node#2 and node#3
wsrep_cluster_address=gcomm://10.0.0.138,10.0.0.139,10.0.0.140

# In order for Galera to work correctly binlog format should be ROW
binlog_format=ROW

# MyISAM storage engine has only experimental support
default_storage_engine=InnoDB

# This changes how InnoDB autoincrement locks are managed and is a requirement for Galera
innodb_autoinc_lock_mode=2

# Node #1 address
wsrep_node_address=10.0.0.138

# SST method
wsrep_sst_method=xtrabackup-v2

# Cluster name
wsrep_cluster_name=Cluster-Wordpress

# Authentication for SST method
wsrep_sst_auth="sstuser:s3cretPass"
```
Sau đó ta chạy bootstrap trên Node1 để làm bootstrap kết nối với các máy còn lại
```
/etc/init.d/mysql bootstrap-pxc
```
Ta có thể kiểm tra kết quả trong mysql shell bằng lệnh
```
show status like 'wsrep%';
``` 

sẽ có kết quả như hình dưới đây.

![]()

Trong MySQL shell tạo user với các quyền thích hợp.

```
mysql> CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 's3cretPass';
mysql> GRANT PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';
mysql> FLUSH PRIVILEGES;
mysql> exit;
```

**Node 2**
Tương tự node 1 chỉnh file /etc/mysql/my.cnf chỉ đổi dòng wsrep_node_address=10.0.0.138 => wsrep_node_address=10.0.0.139





***1-2, Cấu hình DB cluster sử dụng Galera MySQL cluster bản 5.6.***

***1.1, Add các repo, key của package Galera MySQL.***

Ngoài Galera ra thì có thể sử dụng Percona Xtra DB cũng có khả năng sync DB.

Việc add các repo này cần được thực hiện trên 3 node chứa WP và DB cluster.

```
-apt install software-properties-common -y
-sudo apt-key adv --keyserver keyserver.ubuntu.com --recv 44B7345738EBDE52594DAD80D669017EBC19DDBA
-sudo add-apt-repository 'deb [arch=amd64,i386] http://releases.galeracluster.com/ubuntu/ xenial main'
-sudo apt-get update
```

***1.2, Cài đặt MySQL và Galera***

Việc này cũng cần được thực hiện trên cả 3 máy
```
sudo apt-get install galera-3 galera-arbitrator-3 mysql-wsrep-5.6
sudo apt install rsync
```

Khi cài đặt ta sẽ cần set password cho user của MySQL (mặc định sẽ là root)
Việc cài đặt password này sau khi galera cluster được triển khai thì cả 3 máy sẽ tự động sync password MySQL cho nhau cùng chung 1 password.
![]()

***1.3. Cấu hình MySQL trong node WP1.***

Tạo file *galera.cnf* để chứa các cấu hình cho cluster.
```
sudo vim /etc/mysql/conf.d/galera.cnf
```

Cấu hình bên trong file *galera.cnf*
```
# MySQL config
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0

# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Galera Cluster Configuration
wsrep_cluster_name="WPcluster"
wsrep_cluster_address="gcomm://10.0.0.138,10.0.0.139,10.0.0.140"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Galera Node Configuration
wsrep_node_address="10.0.0.138"
wsrep_node_name="WP1"
```

**MySQL config**: dùng để cấu hình MYSQL cho phép cluster. MySQL sẽ không hoạt động nếu mysqld được cấu hình vào localhost hoặc IP address chỉ định và Galera Cluster cũng sẽ không hoạt động với MyISAM hoặc 1 vài engine tương tự nó.

**Galera Provider Configuration**: được dùng để cấu hình các gói cung cấp khả năng wsrep(write set replication API) của MySql.

**Galera Synchronization Configuration**: định dạng cluster, định danh các cluster member bằng IP. 

*wsrep_cluster_name*="*cluster name*"

*wsrep_cluster_address*="*gcomm://first_ip,second_ip,third_ip,etc...*"

**Galera Synchronization Configuration**: xác định phương thức để sync các database.

**Galera Node Configuration**: Xác định IP và tên của node.

*wsrep_node_address*="*IP current node*"

*wsrep_node_name*="*node name*"


Tiếp tục ta cấu hình file */etc/mysql/my.cnf* để tránh việc MySQL tự động bind IP vào 127.0.0.1 và gây conflict với cấu hình của galera. Ta comment dòng thứ 47 trong file */etc/mysql/my.cnf*.

```
47 # bind-address          = 127.0.0.1
```

***1.4, Cấu hình MySQL trên các node còn lại***

Tại các node còn lại ta cũng tạo thêm file galera.cnf như ở node để cấu hình galera cluster. Trong file galera.cnf ta chỉ thay đổi lại phần **Galera Node Configuration** theo cấu hình của node.
NODE2- WP2: ***/etc/mysql/conf.d/galera.cnf***
```
# MySQL config
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0

# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Galera Cluster Configuration
wsrep_cluster_name="WPcluster"
wsrep_cluster_address="gcomm://10.0.0.138,10.0.0.139,10.0.0.140"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Galera Node Configuration
wsrep_node_address="10.0.0.139"
wsrep_node_name="WP2"
```
Node3-WP3: ***/etc/mysql/conf.d/galera.cnf***
```
# MySQL config
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0

# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Galera Cluster Configuration
wsrep_cluster_name="WPcluster"
wsrep_cluster_address="gcomm://10.0.0.138,10.0.0.139,10.0.0.140"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Galera Node Configuration
wsrep_node_address="10.0.0.140"
wsrep_node_name="WP3"
```

Ngoài ra ta cũng  tiếp tục comment dòng thứ 47 trong file */etc/mysql/my.cnf* tại từng node.

***1.5, Triển khai galera cluster***

Trên cả 3 node ta tắt MySQL đang chạy.

``` 
sudo systemctl stop mysql 
```

Kiểm tra bằng lệnh

```
sudo systemctl status mysql
```
Nếu lệnh trên trả về như hình sau có nghĩa là gluster đã được cấu hình thành công trong các bước trên.

![]()

**Trên node1-WP1.**

Ta khởi động lại MySQL đã được tắt đi nhưng không thể khởi động bằng lệnh 
``` 
sudo systemctl start mysql 
``` 
do hiện tại chưa có node nào kết nối với nó. Ta cần khởi động trực tiếp bằng lệnh 
``` 
/etc/init.d/mysql start --wsrep-new-cluster
``` 
với option *--wsrep-new-cluster* như trong ảnh dưới đây

![]()

 nhằm bỏ qua việc cố gắng kết nối đến các node khác ngay lập tức của MySQL. Ta có thể kiểm tra với lệnh 
 ``` 
 /etc/init.d/mysql status 
 ```
 ![]()
 
 Với các node còn lại ta có thể start MySQL như một service bình thường bằng lệnh systemctl.
 
 Tuy nhiên cần phải chú ý là ta không chạy MySQL server mà chỉ chạy một service Galera cluster của nó nên nếu sử dụng câu lệnh 
 ```
 service mysql status
 ``` 
 thì nó sẽ không trả về là đang running như hình dưới
 
 ![]()
 
Ta có thể kiểm tra xem cluster đã hoạt động trên MySQL hay chưa qua câu lệnh
``` 
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'" 
```
lệnh này sẽ show ra số lượng MySQL cluster đang kết nối. Hiện tại mới chỉ có một node1 khởi động nên sẽ có số lượng kết nối như hình dưới.
![]()

**Trên node2-WP2**
Ta khởi động bằng lệnh 
```
systemctl start mysql 
``` 
Và cũng sử dụng lệnh 

``` 
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'" 
```

để kiểm tra và lần này số lượng hiển thị sẽ tăng lên thành 2.
![]()

Cũng như vậy ta khởi động MySQL cluster và check số lượng cluster trên **node3-WP3**
![]()

***1.6, Kiểm tra việc đồng bộ giữa các node qua Galera***
Sau khi triển khai Galera Cluster tất cả các node còn lại sẽ được tự động sync password của node triển khai Galera đầu tiên.

Tại node1-WP1 ta tạo một DB test và đưa vào nó các table cùng các thông tin trong table đó.
![]()

Sang node2 ta truy cập mysql và kiểm tra bằng lệnh


    SELECT * FROM test.equipment


nếu có thể thấy được DB test như bên node1 đã tạo ra thì có nghĩa là node1 và node2 đã đồng bộ DB với nhau.
![]()

Tiếp tục tại node2 ta đưa thêm một thông tin nữa cùng vào DB test này
![]()

Sang node3 ta cũng truy cập vào mysql shell như node2 và kiểm tra DB test
![]()

và nhập thêm thông tin vào DB test
![]()

Quay trở lại node1, ta kiểm tra xem các thông tin được nhập từ node2 và node3 đã đồng bộ  chưa.
![]()

Đến thời điểm này thì cả 3 node WP1, WP2, WP3 đã được cài đặt và triển khai Galera MySQL cluster thành công.


***2, Cấu hình Sync web content sử dụng GLUSTERFS***
***2.1, Cài đặt gluster-server và các package***
 Trên cả 3 node ta sử dụng câu lệnh 
 ``` 
 sudo apt install -y glusterfs-* 
 ``` 
 để cài đặt GlusterFS server và GlusterClient.
 
Sau đó ta khởi động config *rpcbind*  để GlusterFS-Client có thể được tự động mount port sau khi khởi động Gluster-server .
```
systemctl start rpcbind
systemctl enable rpcbind
```

Cùng với đó ta tạo một thư mục trên tất các các máy để có thể làm mount point. Ở đây, ta sẽ tạo thư mục 
```
/wordpress/www
``` 
làm data dir bằng câu lệnh.

```
mkdir -p /wordpress/www
```
Ngoài ra ta cũng tạo ra thêm mount point là thư mục sẽ chứa wordpress mà trong bài này là 
```
/var/www/html
```
```
mkdir -p /var/www/html
```

Trên một node bất kỳ, ta bắt đầu triển khai GlusterFS để bắt đầu sync thư mục sẽ chứa web content

Đầu tiên ta cần kết nối tất cả các node thành một cluster qua câu lệnh

gluster peer probe <IP hoặc host name của IP đã config trong /etc/hosts>

ở đây, ta kết nối từ node1 nên có câu lệnh sau:
```
gluster peer probe 10.0.0.139
gluster peer probe 10.0.0.140
```
Sau đó trên cả 3 máy ta kiểm tra tình trạng kết nối bằng lệnh 
``` 
gluster peer status
```
![]()
Ta thấy rằng state đã hiện là 
```
Peer in Cluster
``` 
là có nghĩa các node đã được nhóm lại thành một cluster.

Tiếp đến ta tạo volume để tạo replica web content.

``` 
sudo gluster volume create volume_name replica num_of_servers transport tcp domain1.com:/path/to/data/directory domain2.com:/path/to/data/directory ... force
```
ở đây ta có cấu lệnh sau:
```
sudo gluster volume create wordpressvol replica 3 transport tcp WP1:/wordpress/www WP2:/wordpress/www WP3:/wordpress/www force
```
sau khi tạo xong ta sẽ có thông báo success, tiếp đến ta khởi động volume này qua câu lệnh
```
sudo gluster volume start wordpressvol
```
câu lệnh trên cũng sẽ trả về thông báo success nếu thành công.
Bây giờ ta có thể kiểm tra tình trạng volume thông qua câu lệnh 
```gluster volume status```
![]()
Trong bảng status này ta sẽ thấy cột Online chuyển sang Y như vậy có nghĩa là volume này đã hoạt động.

Bây giờ ta cần cấu hình lại mount point để sử dụng GlusterFS trong file **/etc/fstab** trên cả 3 node.

Đưa thêm dòng sau vào cuối file **/etc/fstab**
```
localhost:/wordpressvol	/var/www/html  glusterfs   defaults,_netdev      0 0
```
như ảnh dưới đây.
![]()

sau đó ta sử dụng câu lệnh 
``` 
mount -a 
``` 
để bắt đầu mount và bắt đầu sync.

***3, Cài đặt Wordpress***
***3.1, Cài đặt apache và php ***
Ta sử dụng câu lệnh sau trên cả 3 node.

```
apt install -y apache2 php7.0 libapache2-mod-php7.0 php7.0-mcrypt php7.0-mysql 
```

Tiếp đến là thay đổi user cho thư mục /var/www/html bằng câu lệnh

```
sudo chown -R www-data:www-data /var/www/html
```

Tại một node bất kỳ

***3.2, Tạo DB wordpress ***
Trên node bất kỳ, truy cập vào MySQL shell
Tại đó ta tạo ra một DB wordpress để làm wordpress DB như anh dưới
![]()
Các câu lệnh được sử dụng là:
```
CREATE DATABASE wordpress;
CREATE USER wordpress@localhost IDENTIFIED BY 'wordpress';
GRANT ALL PRIVILEGES ON wordpress.* TO wordpress@localhost;
FLUSH PRIVILEGES;
```
Ta chuyển sang các node khác để xác nhận wordpress DB này đã được sync
![]()
![]()

***3.3, Cài đặt Wordpresss**
Tải wordpress bản mới nhất về và giải nén.
```
wget http://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
```
ta sẽ có thư mục wordpress, ta sẽ copy toàn bộ file bên trong đưa vào thư mục */var/www/html* và remove file index.html
```
cp -Rf wordpress/* /var/www/blog/
rm /var/www/html/index.html
```
Tạo thư mục chứa các file upload và set permission.
```
mkdir /var/www/html/wp-content/uploads
chmod 777 /var/www/html/wp-content/uploads
```

Sau đó, ta có thể qua các node khác để kiểm tra xem các node đó đã được đồng bộ chưa, nếu có kết quả như hình dưới có nghĩa là tất cả các node đã được đồng bộ với nhau.
![]()

Tiếp đến cấu hình file wp-config.php được copy từ file wp-config-sample.php trong folder /var/www/html
```
cp wp-config-sample.php wp-config.php
```
Thay đổi các dòng 23, 26, 29 thành như dưới đây.
```
 23 define('DB_NAME', 'wordpress');
 24 
 25 /** MySQL database username */
 26 define('DB_USER', 'wordpress');
 27 
 28 /** MySQL database password */
 29 define('DB_PASSWORD', 'wordpress');
```

Sau đó ta restart lại service apache2 tại cả 3 máy

Ta có thể truy cập cả 3 node cùng 1 wordpress.
![]()
![]()
![]()

## III, Triển khai HAproxy, Load balance, keepalive trên 2 node là HA1 và HA2

-------------------------------------------------------------------------------------------------

HA1: 10.0.0.135

HA2: 10.0.0.136

V-IP: 10.0.0.131

***Triển khai HAproxy, Load balance***
Bước 1:

Trên cả 2 node.

Ta cài đặt haproxy bằng câu lệnh

``` 
apt install haproxy -y 
```
Cho phép Haproxy khởi chạy khi boot.

``` 
echo ENABLED=1 >> /etc/default/haproxy
```

Cấu hình haproxy.cfg thêm vào những đoạn config như dưới đây.

```
frontend wordpress
        bind 10.0.0.135:80
        mode http
        default_backend Cluster_WP

backend Cluster_WP
        mode http
        balance roundrobin
        option forwardfor
        http-request set-header X-Forwarded-Port %[dst_port]
        http-request add-header X-Forwarded-Proto https if { ssl_fc }
        option httpchk HEAD / HTTP/1.1rnHost:localhost
        server WP1      10.0.0.138:80
        server WP2      10.0.0.139:80
        server WP3      10.0.0.140:80

listen  stats
        bind :9000
        stats   enable
        stats   hide-version
        stats   refresh 5s
        stats   show-node
        stats   uri/stats
```
Ta có thể check file config bằng câu lệnh

```
haproxy -c -f /etc/haproxy/haproxy.cfg
```
nếu cấu hình không có gì sai câu lệnh sẽ trả về *Configuration file is valid*

và lúc này ta có thể restart lại haproxy để load config.

```
service haproxy restart
```

Ta có thể truy cập vào IP của HA1 hoặc HA2 và được Load balance ra 3 node WP1, WP2, WP3

Hoặc truy cấp IPHA1/IPHA2:9000/stats để kiểm tra tình trạng load balance như hình dưới.
![]()
![]()

***Triển khai keepalived ***

Cả 2 HA1 và HA2 đều lấy 1 IP là 10.0.0.131 làm IP của chúng, nếu có 1 node trong 2 node HA bị lỗi, node còn lại sẽ lấy IP đó và tiếp tục công việc.

Đầu tiên ta cài đặt các gói phụ trợ keepalived.

```
sudo apt-get install build-essential libssl-dev
```
Sau đó ta cài đặt keepalive vào cả 2 node.

``` 
apt install keepalived -y 
```

Tiếp đến ta tạo file config để cấu hình keepalive.

Tại cả 2 node.

Ta cấu hình file /etc/keepalived/keepalived.conf
``` 
vim /etc/keepalived/keepalived.conf 
```

Tại node 1

Cấu hình như dưới đây tại HA1
```
! Configuration File for keepalived

global_defs {
   notification_email {
     root@vccloud.vn
   }
   notification_email_from lb1@vccloud.vn
   smtp_server localhost
   smtp_connect_timeout 30
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 101
    priority 200
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass wordpress
    }
    virtual_ipaddress {
        10.0.0.131
    }
}

```

Tại node 2
```
! Configuration File for keepalived

global_defs {
   notification_email {
     root@vccloud.vn
   }
   notification_email_from lb2@vccloud.vn
   smtp_server localhost
   smtp_connect_timeout 30
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 101
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass wordpress
    }
    virtual_ipaddress {
        10.0.0.131
    }
}

```

Lúc này ta có thể restart service keepalive trên 2 máy và kiểm tra. 
![]()
![]()

Như 2 hình trên ta thấy được rằng tại HA1 đã có thêm IP nữa là 10.0.0.131 tại interface ens33 như trong config bởi vì HA1 có priority cao hơn HA2.

Truy cập vào HA qua IP 10.0.0.131 ta có logs sau.
![]()

***Test Failover***
Bây giờ ta shutdown node HA1.
Ngay lập tức node HA2 sẽ nhận IP 10.0.0.131 từ node 1 và trở thành Master.
![]()
![]()

Và khi node HA1 khởi động lại thì HA2 sẽ trở lại thành BACKUP
![]()
![]()
