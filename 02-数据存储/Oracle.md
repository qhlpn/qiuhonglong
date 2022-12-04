```
https://www.bilibili.com/video/BV1Ka4y177Wd
https://github.com/samli008/RAC/blob/master/oracle19c_rac/rac.sh
```

##### 依赖包安装（rac1&&rac2）

```
yum install binutils -y
yum install compat-libcap1 -y
yum install compat-libstdc++-33 -y
yum install gcc -y
yum install gcc-c++ -y
yum install glibc -y
yum install glibc-devel -y
yum install ksh -y
yum install libgcc -y
yum install libstdc++ -y
yum install libstdc++-devel -y
yum install libaio -y
yum install libaio-devel -y
yum install libXext -y
yum install libXtst -y
yum install libX11 -y
yum install libXau -y
yum install libxcb -y
yum install libXi -y
yum install make -y
yum install sysstat -y
yum install unixODBC -y
yum install unixODBC-devel -y
yum -y install iscsi-initiator-utils
yum -y install device-mapper-multipath
yum -y install smartmontools
yum -y install readline*

其它教程安装不成功，很大原因是教程依赖包不适合当下虚拟机的Linux内核
```

##### 软件包下载（rac1）

```
wget "http://182.42.255.53/istack-common/oracle/oracle-database-preinstall-19c-1.0-1.el7.x86_64.rpm?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=DO7CMN17GK43Q94CMYXS%2F20220519%2Fneimengzhongxinzone1-zonegroup%2Fs3%2Faws4_request&X-Amz-Date=20220519T121351Z&X-Amz-Expires=604800&X-Amz-SignedHeaders=host&X-Amz-Signature=a7791af406803bd21611f72788e99a6e9101019f907a216547960acca04d3c93" -O oracle-database-preinstall-19c-1.0-1.el7.x86_64.rpm

wget "http://182.42.255.53/istack-common/oracle/cvuqdisk-1.0.10-1.rpm?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=DO7CMN17GK43Q94CMYXS%2F20220519%2Fneimengzhongxinzone1-zonegroup%2Fs3%2Faws4_request&X-Amz-Date=20220519T121516Z&X-Amz-Expires=604800&X-Amz-SignedHeaders=host&X-Amz-Signature=56db69e977f7e3a9cd5803965e931047e63db8df2cab006ab9199dba2e179756" -O cvuqdisk-1.0.10-1.rpm

wget "http://182.42.255.53/istack-common/oracle/LINUX.X64_193000_db_home.zip?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=DO7CMN17GK43Q94CMYXS%2F20220519%2Fneimengzhongxinzone1-zonegroup%2Fs3%2Faws4_request&X-Amz-Date=20220519T121811Z&X-Amz-Expires=604800&X-Amz-SignedHeaders=host&X-Amz-Signature=f7d5ac82c6bad96fc35aae3a4b06fd6e34f3181ea24fd948c35ad54b67fb1859" -O LINUX.X64_193000_db_home.zip

wget "http://182.42.255.53/istack-common/oracle/LINUX.X64_193000_grid_home.zip?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=DO7CMN17GK43Q94CMYXS%2F20220519%2Fneimengzhongxinzone1-zonegroup%2Fs3%2Faws4_request&X-Amz-Date=20220519T035129Z&X-Amz-Expires=604800&X-Amz-SignedHeaders=host&X-Amz-Signature=abd8496c9e6eefcfc44172175b876dc4050f7035da04de9eca3755c4a5fde9d5" -O LINUX.X64_193000_grid_home.zip
```

##### 补丁安装（rac1&&rac2）

```
使用上步骤下载的软件
yum -y install oracle-database-preinstall-19c-1.0-1.el7.x86_64.rpm
rpm -ivh cvuqdisk-1.0.10-1.rpm
```

##### 配置 step1（rac1&&rac2）

```
# 8核16G，双网卡，vip，2块共享数据盘

# config /etc/hosts
10.254.76.2 rac1
10.254.76.3 rac2
10.254.96.2 rac1prv
10.254.96.3 rac2prv
10.254.76.12 rac1vip
10.254.76.13 rac2vip
10.254.76.11 scanvip

# modify user and group both nodes
userdel -r oracle
userdel -r grid
groupdel oinstall
groupdel dba
groupadd -g 5001 oinstall
groupadd -g 5002 dba
groupadd -g 5003 asmdba
groupadd -g 5004 asmoper
groupadd -g 5005 asmadmin
useradd -u 6001 -g oinstall -G asmadmin,asmdba,asmoper grid
useradd -u 6002 -g oinstall -G dba,asmdba,asmadmin oracle

echo "oracle" | passwd --stdin grid
echo "oracle" | passwd --stdin oracle

# create install dir both nodes
mkdir /opt/oracle
mkdir -p /opt/oracle/app/grid
mkdir -p /opt/oracle/app/19c/grid
chown -R grid:oinstall /opt/oracle
mkdir -p /opt/oracle/app/oraInventory
chown -R grid:oinstall /opt/oracle/app/oraInventory
mkdir -p /opt/oracle/app/oracle/product/19c/dbhome_1
chown -R oracle:oinstall /opt/oracle/app/oracle
chmod -R 775 /opt/oracle

# stop firewalld and selinux
systemctl stop firewalld.service
systemctl disable firewalld.service
systemctl status firewalld.service

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
getenforce

# prepare ASM disk on both nodes
# mount two iscsi-disk and get their ID_SERIAL
ls -l /dev/disk/by-id

# asm-ocr 仲裁盘、asm-data 数据盘
cat > /etc/udev/rules.d/99-oracle-asmdevices.rules << EOF
KERNEL=="sd*", ENV{ID_SERIAL}=="3206b076e6d016f6f",SYMLINK+="asm/asm-ocr", OWNER="grid", GROUP="asmadmin", MODE="0660"
KERNEL=="sd*", ENV{ID_SERIAL}=="3206c076e6d016f6f",SYMLINK+="asm/asm-data", OWNER="grid", GROUP="asmadmin", MODE="0660"
EOF
systemctl restart systemd-udev-trigger
ll /dev |grep asm

# update ntpdate
yum install -y ntpdate
ntpdate cn.pool.ntp.org

# config some parameters
memTotal=$(grep MemTotal /proc/meminfo | awk '{print $2}')
totalMemory=$((memTotal / 2048))
shmall=$((memTotal / 4))
if [ $shmall -lt 2097152 ]; then
	shmall=2097152
fi
shmmax=$((memTotal * 1024 - 1))
if [ "$shmmax" -lt 4294967295 ]; then
	shmmax=4294967295
fi
cat <<EOF>>/etc/sysctl.conf
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = $shmall
kernel.shmmax = $shmmax
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
net.ipv4.conf.eth0.rp_filter = 1
net.ipv4.conf.eth1.rp_filter = 2
EOF

sysctl -p

cat <<EOF>>/etc/security/limits.conf
oracle soft nofile 1024
oracle hard nofile 65536
oracle soft stack 10240
oracle hard stack 32768
oracle soft nproc 2047
oracle hard nproc 16384
oracle hard memlock 134217728
oracle soft memlock 134217728
grid soft nofile 1024
grid hard nofile 65536
grid soft stack 10240
grid hard stack 32768
grid soft nproc 2047
grid hard nproc 16384
EOF

cat <<EOF>>/etc/pam.d/login
session required pam_limits.so 
session required /lib64/security/pam_limits.so
EOF

## 重启一下主机
```

##### 配置 step2（rac1）

```
# grid env config
cat >> /home/grid/.bash_profile << EOF
umask 022
export ORACLE_SID=+ASM1
export ORACLE_BASE=/opt/oracle/app/grid
export ORACLE_HOME=/opt/oracle/app/19c/grid
export INVENTORY_LOCATION=/opt/oracle/app/oraInventory
export PATH=.:\$PATH:\$HOME/bin:\$ORACLE_HOME/bin
EOF

# oracle env config
cat >> /home/oracle/.bash_profile << EOF
umask 022
export ORACLE_BASE=/opt/oracle/app/oracle
export ORACLE_HOME=\$ORACLE_BASE/product/19c/dbhome_1
export ORACLE_UNQNAME=oracledb
export ORACLE_SID=oracledb1
export INVENTORY_LOCATION=/opt/oracle/app/oraInventory
export PATH=.:\$PATH:\$HOME/bin:\$ORACLE_HOME/bin
EOF
```

##### 配置 step3（rac2）

```
# grid env config
cat >> /home/grid/.bash_profile << EOF
umask 022
export ORACLE_SID=+ASM2
export ORACLE_BASE=/opt/oracle/app/grid
export ORACLE_HOME=/opt/oracle/app/19c/grid
export INVENTORY_LOCATION=/opt/oracle/app/oraInventory
export PATH=.:\$PATH:\$HOME/bin:\$ORACLE_HOME/bin
EOF

# oracle env config
cat >> /home/oracle/.bash_profile << EOF
umask 022
export ORACLE_BASE=/opt/oracle/app/oracle
export ORACLE_HOME=\$ORACLE_BASE/product/19c/dbhome_1
export ORACLE_UNQNAME=oracledb
export ORACLE_SID=oracledb2
export INVENTORY_LOCATION=/opt/oracle/app/oraInventory
export PATH=.:\$PATH:\$HOME/bin:\$ORACLE_HOME/bin
EOF
```

##### 安装 grid step1（rac1）

```
# in root user 
mkdir /soft
cp LINUX.X64_193000_db_home.zip /soft/
cp LINUX.X64_193000_grid_home.zip /soft/
chown -R grid:oinstall /soft

# switch grid user
su - grid
cd $ORACLE_HOME

# unzip soft
unzip /soft/LINUX.X64_193000_grid_home.zip

# config grid user ssh auth
./oui/prov/resources/scripts/sshUserSetup.sh -user grid -hosts "rac1 rac2" -advanced -noPromptPassphrase

# install grid soft
# 参数需要跟环境对应：
# INVENTORY_LOCATION、ORACLE_BASE、scanName、clusterNodes、networkInterfaceList、diskDiscoveryString、disks

${ORACLE_HOME}/gridSetup.sh -ignorePrereq -waitforcompletion -silent \
 -responseFile ${ORACLE_HOME}/install/response/gridsetup.rsp \
INVENTORY_LOCATION=/opt/oracle/app/oraInventory \
SELECTED_LANGUAGES=en,en_GB \
oracle.install.option=CRS_CONFIG \
ORACLE_BASE=/opt/oracle/app/grid \
oracle.install.asm.OSDBA=asmdba \
oracle.install.asm.OSASM=asmadmin \
oracle.install.asm.OSOPER=asmoper \
oracle.install.crs.config.scanType=LOCAL_SCAN \
oracle.install.crs.config.gpnp.scanName=scanvip \
oracle.install.crs.config.gpnp.scanPort=1521 \
oracle.install.crs.config.ClusterConfiguration=STANDALONE \
oracle.install.crs.config.configureAsExtendedCluster=false \
oracle.install.crs.config.clusterName=gridcluster \
oracle.install.crs.config.gpnp.configureGNS=false \
oracle.install.crs.config.autoConfigureClusterNodeVIP=false \
oracle.install.crs.config.clusterNodes=rac1:rac1vip,rac2:rac2vip \
oracle.install.crs.config.networkInterfaceList=eth0:10.254.76.0:1,eth1:10.254.96.0:5 \
oracle.install.asm.configureGIMRDataDG=false \
oracle.install.crs.config.useIPMI=false \
oracle.install.asm.storageOption=ASM \
oracle.install.asmOnNAS.configureGIMRDataDG=false \
oracle.install.asm.SYSASMPassword=oracle \
oracle.install.asm.diskGroup.name=DGGRID1 \
oracle.install.asm.diskGroup.redundancy=EXTERNAL \
oracle.install.asm.diskGroup.AUSize=4 \
oracle.install.asm.diskGroup.disks=/dev/asm/asm-ocr \
oracle.install.asm.diskGroup.diskDiscoveryString=/dev/asm/* \
oracle.install.asm.configureAFD=false \
oracle.install.asm.monitorPassword=oracle \
oracle.install.crs.configureRHPS=false \
oracle.install.crs.config.ignoreDownNodes=false \
oracle.install.config.managementOption=NONE \
oracle.install.config.omsPort=0 \
oracle.install.crs.rootconfig.executeRootScript=false 

# output
As a root user, execute the following script(s):
        1. /opt/oracle/oraInventory/orainstRoot.sh
        2. /opt/oracle/app/19c/grid/root.sh

Execute /opt/oracle/oraInventory/orainstRoot.sh on the following nodes: 
[rac1, rac2]
Execute /opt/oracle/app/19c/grid/root.sh on the following nodes: 
[rac1, rac2]

Run the script on the local node first. After successful completion, you can start the script in parallel on all other nodes.

Successfully Setup Software with warning(s).
As install user, execute the following command to complete the configuration.
        /opt/oracle/app/19c/grid/gridSetup.sh -executeConfigTools -responseFile /opt/oracle/app/19c/grid/install/response/gridsetup.rsp [-silent]
```

##### 安装 grid step2（rac1&&rac2）

```
# 先rac1执行完，再rac2
# in root user
/opt/oracle/app/oraInventory/orainstRoot.sh
/opt/oracle/app/19c/grid/root.sh
```

##### 安装 grid step3（rac3）

```
# 与 step1 相比，把 -ignorePrereq 改成 -executeConfigTools 即可

${ORACLE_HOME}/gridSetup.sh -executeConfigTools -waitforcompletion -silent \
 -responseFile ${ORACLE_HOME}/install/response/gridsetup.rsp \
INVENTORY_LOCATION=/opt/oracle/app/oraInventory \
SELECTED_LANGUAGES=en,en_GB \
oracle.install.option=CRS_CONFIG \
ORACLE_BASE=/opt/oracle/app/grid \
oracle.install.asm.OSDBA=asmdba \
oracle.install.asm.OSASM=asmadmin \
oracle.install.asm.OSOPER=asmoper \
oracle.install.crs.config.scanType=LOCAL_SCAN \
oracle.install.crs.config.gpnp.scanName=scanvip \
oracle.install.crs.config.gpnp.scanPort=1521 \
oracle.install.crs.config.ClusterConfiguration=STANDALONE \
oracle.install.crs.config.configureAsExtendedCluster=false \
oracle.install.crs.config.clusterName=gridcluster \
oracle.install.crs.config.gpnp.configureGNS=false \
oracle.install.crs.config.autoConfigureClusterNodeVIP=false \
oracle.install.crs.config.clusterNodes=rac1:rac1vip,rac2:rac2vip \
oracle.install.crs.config.networkInterfaceList=eth0:10.254.76.0:1,eth1:10.254.96.0:5 \
oracle.install.asm.configureGIMRDataDG=false \
oracle.install.crs.config.useIPMI=false \
oracle.install.asm.storageOption=ASM \
oracle.install.asmOnNAS.configureGIMRDataDG=false \
oracle.install.asm.SYSASMPassword=oracle \
oracle.install.asm.diskGroup.name=DGGRID1 \
oracle.install.asm.diskGroup.redundancy=EXTERNAL \
oracle.install.asm.diskGroup.AUSize=4 \
oracle.install.asm.diskGroup.disks=/dev/asm/asm-ocr \
oracle.install.asm.diskGroup.diskDiscoveryString=/dev/asm/* \
oracle.install.asm.configureAFD=false \
oracle.install.asm.monitorPassword=oracle \
oracle.install.crs.configureRHPS=false \
oracle.install.crs.config.ignoreDownNodes=false \
oracle.install.config.managementOption=NONE \
oracle.install.config.omsPort=0 \
oracle.install.crs.rootconfig.executeRootScript=false 
```

##### 验证 grid 成功（rac1&&rac2）

```
su - grid
crsctl stat res -t

# rac1、rac2的 Target 和 State 一致 ONLINE
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.LISTENER.lsnr
               ONLINE  ONLINE       rac1                     STABLE
               ONLINE  ONLINE       rac2                     STABLE
ora.chad
               ONLINE  ONLINE       rac1                     STABLE
               ONLINE  ONLINE       rac2                     STABLE
ora.net1.network
               ONLINE  ONLINE       rac1                     STABLE
               ONLINE  ONLINE       rac2                     STABLE
ora.ons
               ONLINE  ONLINE       rac1                     STABLE
               ONLINE  ONLINE       rac2                     STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.ASMNET1LSNR_ASM.lsnr(ora.asmgroup)
      1        ONLINE  ONLINE       rac1                     STABLE
      2        ONLINE  ONLINE       rac2                     STABLE
      3        ONLINE  OFFLINE                               STABLE
ora.DGDATA01.dg(ora.asmgroup)
      1        ONLINE  ONLINE       rac1                     STABLE
      2        ONLINE  ONLINE       rac2                     STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.DGGRID1.dg(ora.asmgroup)
      1        ONLINE  ONLINE       rac1                     STABLE
      2        ONLINE  ONLINE       rac2                     STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.LISTENER_SCAN1.lsnr
      1        ONLINE  ONLINE       rac2                     STABLE
ora.asm(ora.asmgroup)
      1        ONLINE  ONLINE       rac1                     Started,STABLE
      2        ONLINE  ONLINE       rac2                     Started,STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.asmnet1.asmnetwork(ora.asmgroup)
      1        ONLINE  ONLINE       rac1                     STABLE
      2        ONLINE  ONLINE       rac2                     STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.cvu
      1        ONLINE  ONLINE       rac2                     STABLE
ora.qosmserver
      1        ONLINE  ONLINE       rac2                     STABLE
ora.rac1.vip
      1        ONLINE  ONLINE       rac1                     STABLE
ora.rac2.vip
      1        ONLINE  ONLINE       rac2                     STABLE
ora.scan1.vip
      1        ONLINE  ONLINE       rac2                     STABLE
--------------------------------------------------------------------------------
```

##### 添加 ASM 数据盘 step1（rac1）

```
su - grid

sqlplus "/as sysasm"
desc v$asm_diskgroup;

# 查看磁盘（当下有块仲裁盘）
select NAME,TOTAL_MB,FREE_MB from v$asm_diskgroup;

# 添加数据盘
create diskgroup DGDATA01 external redundancy disk '/dev/asm/asm-data' ATTRIBUTE 'au_size'='1M', 'compatible.asm' = '19.3';

select NAME,TOTAL_MB,FREE_MB from v$asm_diskgroup;
```

##### 添加 ASM 数据盘 step2（rac2）

```
su - grid

sqlplus "/as sysasm"
desc v$asm_diskgroup;
select NAME,TOTAL_MB,FREE_MB from v$asm_diskgroup;

# 刷新数据盘 TOTAL_MB和FREE_MB 字段值
alter diskgroup DGDATA01 mount;

select NAME,TOTAL_MB,FREE_MB from v$asm_diskgroup;


==> output

SQL> select NAME,TOTAL_MB,FREE_MB from v$asm_diskgroup;

NAME                             TOTAL_MB    FREE_MB
------------------------------ ---------- ----------
DGDATA01                           153600     142694
DGGRID1                            102400     101848

至此，grid软件安装结束，可见两块共享盘组成的ASM磁盘组，正常使用
```

##### 安装 oracle step（rac1）

```
# in root user 
mkdir /soft
chown -R oracle:oinstall /soft

# in oracle user
su - oracle
cd $ORACLE_HOME
unzip /soft/LINUX.X64_193000_db_home.zip

# config oracle user ssh auth
./oui/prov/resources/scripts/sshUserSetup.sh -user oracle -hosts "rac1 rac2" -advanced -noPromptPassphrase

# 安装数据库实例对象

${ORACLE_HOME}/runInstaller -ignorePrereq -waitforcompletion -silent \
-responseFile ${ORACLE_HOME}/install/response/db_install.rsp \
oracle.install.option=INSTALL_DB_SWONLY \
ORACLE_HOSTNAME=rac1 \
UNIX_GROUP_NAME=oinstall \
INVENTORY_LOCATION=/opt/oracle/app/oraInventory \
SELECTED_LANGUAGES=en,en_GB \
ORACLE_HOME=/opt/oracle/app/oracle/product/19c/dbhome_1 \
ORACLE_BASE=/opt/oracle/app/oracle \
oracle.install.db.InstallEdition=EE \
oracle.install.db.OSDBA_GROUP=dba \
oracle.install.db.OSOPER_GROUP=oinstall \
oracle.install.db.OSBACKUPDBA_GROUP=dba \
oracle.install.db.OSDGDBA_GROUP=dba \
oracle.install.db.OSKMDBA_GROUP=dba \
oracle.install.db.OSRACDBA_GROUP=dba \
oracle.install.db.CLUSTER_NODES=rac1,rac2 \
oracle.install.db.isRACOneInstall=false \
oracle.install.db.rac.serverpoolCardinality=0 \
oracle.install.db.config.starterdb.type=GENERAL_PURPOSE \
oracle.install.db.ConfigureAsContainerDB=false \
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false \

===> output
As a root user, execute the following script(s):
        1. /opt/oracle/app/oracle/product/19c/dbhome_1/root.sh

Execute /opt/oracle/app/oracle/product/19c/dbhome_1/root.sh on the following nodes: 
[rac1, rac2]

```

##### 安装 oracle step2（rac1&&rac2）

```
# 先rac1执行完，再rac2
# in root user
/opt/oracle/app/oracle/product/19c/dbhome_1/root.sh
```

##### 安装 oracle step3（rac1）

```
# 创建 ORACLE Database

# in grid user
su - grid
crsctl check crs

==> output 
CRS-4638: Oracle High Availability Services is online
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online

# in oracle user
su - oracle
dbca -silent -createDatabase \
 -templateName General_Purpose.dbc \
 -gdbname oracledb -responseFile NO_VALUE \
 -characterSet AL32UTF8 \
 -sysPassword oracle \
 -systemPassword oracle \
 -createAsContainerDatabase false \
 -databaseType MULTIPURPOSE \
 -automaticMemoryManagement false \
 -totalMemory 1024 \
 -redoLogFileSize 50 \
 -emConfiguration NONE \
 -ignorePreReqs \
 -nodelist rac1,rac2 \
 -storageType ASM \
 -diskGroupName +DGDATA01 \
 -asmsnmpPassword oracle \

[FATAL] ORA-03113: end-of-file on communication channel  (没找出是什么原因)
```

