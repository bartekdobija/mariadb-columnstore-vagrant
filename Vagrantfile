
# Basic OS configuration
$sysconfig = <<SCRIPT

# disable IPv6
echo "net.ipv6.conf.all.disable_ipv6=1" > /etc/sysctl.d/99-ipv6.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.d/99-ipv6.conf
sysctl -f /etc/sysctl.d/99-ipv6.conf

# socket finetuning
echo "net.core.somaxconn=4096" > /etc/sysctl.d/99-socks.conf
echo "net.ipv4.tcp_fin_timeout=30" >> /etc/sysctl.d/99-socks.conf
echo "net.ipv4.tcp_keepalive_intvl=30" >> /etc/sysctl.d/99-socks.conf
echo "net.ipv4.tcp_keepalive_time=120" >> /etc/sysctl.d/99-socks.conf
echo "net.ipv4.tcp_max_syn_backlog=4096" >> /etc/sysctl.d/99-socks.conf
echo "net.core.rmem_max=16777216" >> /etc/sysctl.d/99-socks.conf
echo "net.core.wmem_max=16777216" >> /etc/sysctl.d/99-socks.conf
echo "net.ipv4.tcp_rmem=4096 87380 16777216" >> /etc/sysctl.d/99-socks.conf
echo "net.ipv4.tcp_wmem=4096 65536 16777216" >> /etc/sysctl.d/99-socks.conf
echo "net.ipv4.tcp_no_metrics_save=1" >> /etc/sysctl.d/99-socks.conf
echo "net.core.netdev_max_backlog=30000" >> /etc/sysctl.d/99-socks.conf

# swapiness
echo "10" > /proc/sys/vm/vfs_cache_pressure
echo "vm.swappiness=1" > /etc/sysctl.d/99-swapiness.conf
sysctl -f /etc/sysctl.d/99-swapiness.conf

# THP - reboot needed
echo "transparent_hugepage=never" > /etc/grub.d/20_thp

# dirty ratio
echo "vm.dirty_ratio = 10" > /etc/sysctl.d/99-dirtyratio.conf
echo "vm.dirty_background_ratio = 3" >> /etc/sysctl.d/99-dirtyratio.conf
sysctl -f /etc/sysctl.d/99-swapiness.conf

# this should be a persistent config
ulimit -n 65536
ulimit -s 10240
ulimit -c unlimited

echo "mariadb soft nproc 64000" > /etc/security/limits.d/mariadb.conf
echo "mariadb hard nproc 64000" >> /etc/security/limits.d/mariadb.conf
echo "mariadb soft nofile 64000" >> /etc/security/limits.d/mariadb.conf
echo "mariadb hard nofile 64000" >> /etc/security/limits.d/mariadb.conf

  # create FS
  if [ ! -e /opt/media ]; then
    mkdir -p /opt/media \
      && mkfs.ext4 -q -F /dev/sdb \
      && mount /dev/sdb /opt/media
  fi

  adduser mysql

  mkdir -p /opt/media/{db,tmp} \
    && chown -R mysql:mysql /opt/media/db \
    && chmod 777 /opt/media/tmp

  service iptables stop && chkconfig iptables off

  # Add entries to /etc/hosts
  ip=$(ifconfig eth1 | awk -v host=$(hostname) '/inet addr/ {print substr($2,6)}')
  host=$(hostname)
  echo "127.0.0.1 localhost" > /etc/hosts
  echo "$ip $host" >> /etc/hosts

SCRIPT

$mariadb = <<SCRIPT

  yum -y install expect perl perl-DBI openssl zlib perl-DBD-MySQL boost boost-devel wget

  if [ "$(ls -al | grep mariadb-columnstore.*rpm | wc -l)" == "0" ]; then
    echo "Downloading & Installing MariaDB ColumnStore..."
    wget -q -P /tmp https://downloads.mariadb.com/ColumnStore/1.0.7/centos/x86_64/7/mariadb-columnstore-1.0.7-1-centos7.x86_64.rpm.tar.gz \
      && tar zxf /tmp/mariadb-columnstore-1.0.7-1-centos7.x86_64.rpm.tar.gz \
      && rpm -i mariadb-columnstore*.rpm
  fi

cat << MARIADBCNF > /usr/local/mariadb/columnstore/mysql/my.cnf

[client]
port = 3306
socket = /usr/local/mariadb/columnstore/mysql/lib/mysql/mysql.sock

[mysqld]
port = 3306
socket = /usr/local/mariadb/columnstore/mysql/lib/mysql/mysql.sock
datadir = /usr/local/mariadb/columnstore/mysql/db
skip-external-locking
key_buffer_size = 512M
max_allowed_packet = 1M
table_cache = 512
sort_buffer_size = 4M
read_buffer_size = 4M
read_rnd_buffer_size = 16M
myisam_sort_buffer_size = 64M
thread_cache_size = 8
query_cache_size = 0
# Try number of CPU's*2 for thread_concurrency
#thread_concurrency = 8
thread_stack = 512K
lower_case_table_names=1
group_concat_max_len=512

infinidb_compression_type=2

infinidb_stringtable_threshold=20

# infinidb local query flag
infinidb_local_query = 1

infinidb_diskjoin_smallsidelimit=0
infinidb_diskjoin_largesidelimit=0
infinidb_diskjoin_bucketsize=100
infinidb_um_mem_limit=0

infinidb_use_import_for_batchinsert=1
infinidb_import_for_batchinsert_delimiter=7

basedir                         = /usr/local/mariadb/columnstore/mysql/
character-sets-dir              = /usr/local/mariadb/columnstore/mysql/share/charsets/
lc-messages-dir                 = /usr/local/mariadb/columnstore/mysql/share/
plugin_dir                      = /usr/local/mariadb/columnstore/mysql/lib/plugin

# Replication Master Server (default)
# binary logging is required for replication
# log-bin=mysql-bin
# binlog_format=ROW

# required unique id between 1 and 2^32 - 1
# defaults to 1 if master-host
# uses to 2+ if slave-host
server-id = 1
slave-skip-errors=all

# binary logging - not required for slaves, but recommended
log-bin=/usr/local/mariadb/columnstore/mysql/db/mysql-bin
relay-log=/usr/local/mariadb/columnstore/mysql/db/relay-bin
relay-log-index = /usr/local/mariadb/columnstore/mysql/db/relay-bin.index 
relay-log-info-file = /usr/local/mariadb/columnstore/mysql/db/relay-bin.info

# Point the following paths to different dedicated disks
tmpdir      = /opt/media/tmp/
#log-update     = /path-to-dedicated-directory/hostname

# Uncomment the following if you are using InnoDB tables
#innodb_data_home_dir = /usr/local/mariadb/columnstore/mysql/lib/mysql/
#innodb_data_file_path = ibdata1:2000M;ibdata2:10M:autoextend
#innodb_log_group_home_dir = /usr/local/mariadb/columnstore/mysql/lib/mysql/
#innodb_log_arch_dir = /usr/local/mariadb/columnstore/mysql/lib/mysql/
# You can set .._buffer_pool_size up to 50 - 80 %
# of RAM but beware of setting memory usage too high
#innodb_buffer_pool_size = 384M
#innodb_additional_mem_pool_size = 20M
# Set .._log_file_size to 25 % of buffer pool size
#innodb_log_file_size = 100M
#innodb_log_buffer_size = 8M
#innodb_flush_log_at_trx_commit = 1
#innodb_lock_wait_timeout = 50

[mysqldump]
quick
max_allowed_packet = 16M

[mysql]
no-auto-rehash
# Remove the next comment character if you are not familiar with SQL
#safe-updates

[isamchk]
key_buffer_size = 256M
sort_buffer_size = 256M
read_buffer = 2M
write_buffer = 2M

[myisamchk]
key_buffer_size = 256M
sort_buffer_size = 256M
read_buffer = 2M
write_buffer = 2M

[mysqlhotcopy]
interactive-timeout

MARIADBCNF

SCRIPT

disk = './dataDisk.vdi'

Vagrant.configure(2) do |config|

  config.vm.box = "centos/7"
  config.vm.hostname = "mariadb.instance.com"
  config.vm.network "private_network", type: "dhcp"

  config.vm.provider "virtualbox" do |vb|
    vb.name = "mariadb-vm"
    vb.cpus = 2
    vb.memory = 4096
    vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
    vb.customize ["modifyvm", :id, "--cpuexecutioncap", "100"]
    unless File.exist?(disk)
      vb.customize ['createhd', '--filename', disk, '--variant', 'Fixed', '--size', 5 * 1024]
    end
    vb.customize ['storageattach', :id,  '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', disk]
  end

  config.vm.provision :shell, :name => "sysconfig", :inline => $sysconfig
  config.vm.provision :shell, :name => "mariadb", :inline => $mariadb

end
