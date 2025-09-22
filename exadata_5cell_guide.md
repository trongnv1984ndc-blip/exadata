# Hướng dẫn cài đặt Exadata Simulation trên VMware với 5 Cell Nodes

## Tổng quan

Tài liệu này hướng dẫn chi tiết cách cài đặt Oracle Exadata Storage Server Simulation trên VMware với cấu hình 5 cell nodes: **exacell01**, **exacell02**, **exacell03**, **exacell04**, **exacell05** sử dụng Oracle Storage Server Software version **12.1.1.1.0**.

## Phần I: Chuẩn bị và yêu cầu hệ thống

### 1. Yêu cầu phần cứng

**Tài nguyên VM tối thiểu:**
- **Memory:** 4GB RAM cho mỗi Storage Server (tổng 20GB cho 5 cell)
- **Storage:** 50GB cho mỗi Cell Server (tổng 250GB)
- **CPU:** 2 vCPU cho mỗi Cell Server (tổng 10 vCPU)

**Yêu cầu bổ sung cho mỗi Cell:**
- 18 Hard Disks: 2GB mỗi disk (cho Cell Disks)
- 6 Flash Disks: 1.5GB mỗi disk (cho Flash Cache)

### 2. Phần mềm cần thiết

- **Oracle Linux:** 6.x hoặc 7.x (khuyến nghị 6.9)
- **Oracle Storage Server Software:** 12.1.1.1.0 (V42777-01.zip)
- **JDK:** 1.7.0_25 (đi kèm với Cell software)
- **VMware:** VirtualBox 6.0+ hoặc VMware Workstation 15+

### 3. Cấu hình mạng cho 5 Cell

**Schema mạng:**

| Cell Name | Public IP | Private IP (Interconnect) | Hostname |
|-----------|-----------|---------------------------|----------|
| exacell01 | 192.168.56.61 | 192.168.2.61 | exacell01.localdomain |
| exacell02 | 192.168.56.62 | 192.168.2.62 | exacell02.localdomain |
| exacell03 | 192.168.56.63 | 192.168.2.63 | exacell03.localdomain |
| exacell04 | 192.168.56.64 | 192.168.2.64 | exacell04.localdomain |
| exacell05 | 192.168.56.65 | 192.168.2.65 | exacell05.localdomain |

**Network Adapters:**
- **eth0:** Public Network (192.168.56.0/24) - Host-only hoặc NAT
- **eth1:** Private Network (192.168.2.0/24) - Internal Network

## Phần II: Cài đặt và cấu hình Cell Node Master (exacell01)

### Bước 1: Cấu hình hệ điều hành cơ bản

#### 1.1 Cấu hình file /etc/hosts

```bash
[root@exacell01 ~]# vi /etc/hosts
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
#::1 localhost localhost.localdomain localhost6 localhost6.localdomain6

# Cell Nodes - Public Network
192.168.56.61 exacell01.localdomain exacell01
192.168.56.62 exacell02.localdomain exacell02
192.168.56.63 exacell03.localdomain exacell03
192.168.56.64 exacell04.localdomain exacell04
192.168.56.65 exacell05.localdomain exacell05

# Cell Nodes - Private Network (Interconnect)
192.168.2.61 exacell01-ib.localdomain exacell01-ib
192.168.2.62 exacell02-ib.localdomain exacell02-ib
192.168.2.63 exacell03-ib.localdomain exacell03-ib
192.168.2.64 exacell04-ib.localdomain exacell04-ib
192.168.2.65 exacell05-ib.localdomain exacell05-ib
```

#### 1.2 Cấu hình kernel parameters cho version 12.1.1.1.0

```bash
[root@exacell01 ~]# vi /etc/sysctl.conf

##### Exadata 12.1.1.1.0 Parameters ################
fs.file-max = 655360
fs.aio-max-nr = 50000000
net.core.rmem_default = 4194304
net.core.rmem_max = 4194304
net.core.wmem_default = 4194304
net.core.wmem_max = 4194304
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.shmall = 2097152
kernel.shmmax = 4398046511104
##### End Exadata Parameters ###########
```

Áp dụng thay đổi:
```bash
[root@exacell01 ~]# sysctl -p
```

#### 1.3 Cấu hình file limits

```bash
[root@exacell01 ~]# vi /etc/security/limits.conf
# Thêm vào cuối file:
* soft nofile 655360
* hard nofile 655360
* soft nproc 16384  
* hard nproc 16384
* soft stack 10240
* hard stack 32768
```

#### 1.4 Cấu hình GRUB và SELinux

```bash
# Thay đổi grub default
[root@exacell01 ~]# vi /etc/grub.conf
default=1

# Tắt SELinux
[root@exacell01 ~]# vi /etc/selinux/config
SELINUX=disabled

# Thêm DISPLAY variable
[root@exacell01 ~]# vi /etc/bashrc
export DISPLAY=:0
```

#### 1.5 Tắt firewall và các services không cần thiết

```bash
[root@exacell01 ~]# chkconfig iptables off
[root@exacell01 ~]# chkconfig ip6tables off
[root@exacell01 ~]# service iptables stop
[root@exacell01 ~]# service ip6tables stop

# Tắt NetworkManager (khuyến nghị)
[root@exacell01 ~]# chkconfig NetworkManager off
[root@exacell01 ~]# service NetworkManager stop
```

### Bước 2: Cấu hình RDS modules

#### 2.1 Load RDS kernel modules

```bash
[root@exacell01 ~]# modprobe rds
[root@exacell01 ~]# modprobe rds_tcp
[root@exacell01 ~]# modprobe rds_rdma

# Kiểm tra
[root@exacell01 ~]# lsmod | grep rds
```

#### 2.2 Cấu hình auto-load RDS

```bash
[root@exacell01 ~]# vi /etc/modprobe.d/rds.conf
install rds /sbin/modprobe --ignore-install rds && /sbin/modprobe rds_tcp && /sbin/modprobe rds_rdma
```

#### 2.3 Xóa conflicting packages nếu có

```bash
[root@exacell01 ~]# rpm -qa | egrep 'rds|rdma'
[root@exacell01 ~]# rpm -e rdma-6.8_4.1-1.el6.noarch  # nếu có
```

### Bước 3: Chuẩn bị và cài đặt phần mềm

#### 3.1 Tạo thư mục cần thiết

```bash
[root@exacell01 ~]# mkdir /var/log/oracle
[root@exacell01 ~]# chmod 775 /var/log/oracle
[root@exacell01 ~]# mkdir /storage_server_software
```

#### 3.2 Upload và giải nén Oracle Cell Software 12.1.1.1.0

```bash
[root@exacell01 ~]# cd /storage_server_software

# Upload file V42777-01.zip và giải nén
[root@exacell01 storage_server_software]# unzip V42777-01.zip
Archive: V42777-01.zip
  inflating: README.txt
  inflating: cellImageMaker_12.1.1.1.0_LINUX.X64_131219-1.x86_64.tar

[root@exacell01 storage_server_software]# tar -xvf cellImageMaker_12.1.1.1.0_LINUX.X64_131219-1.x86_64.tar
```

#### 3.3 Cài đặt JDK 1.7

```bash
[root@exacell01 ~]# cd /storage_server_software/dl180/boot/cellbits
[root@exacell01 cellbits]# unzip cell.bin
Archive: cell.bin
warning [cell.bin]: 20118 extra bytes at beginning or within zipfile
(attempting to process anyway)
  inflating: cell-12.1.1.1.0_LINUX.X64_131219-1.x86_64.rpm

# Extract tất cả RPMs
[root@exacell01 cellbits]# tar -xvf cellrpms.tbz

# Cài đặt JDK
[root@exacell01 cellbits]# rpm -ivh jdk-1.7.0_25-fcs.x86_64.rpm

# Set environment variables
[root@exacell01 ~]# export JAVA_HOME=/usr/java/jdk1.7.0_25
[root@exacell01 ~]# export PATH=$JAVA_HOME/bin:$PATH
[root@exacell01 ~]# java -version
java version "1.7.0_25"
Java(TM) SE Runtime Environment (build 1.7.0_25-b15)
Java HotSpot(TM) 64-Bit Server VM (build 23.25-b01, mixed mode)
```

#### 3.4 Cài đặt required packages cho 12.1.1.1.0

```bash
# Cài đặt net-snmp
[root@exacell01 ~]# yum install net-snmp net-snmp-utils

# Cài đặt perl XML packages
[root@exacell01 ~]# yum install perl-XML-NamespaceSupport
[root@exacell01 ~]# yum install perl-XML-SAX
[root@exacell01 ~]# rpm -ivh perl-XML-SAX-Expat-*.rpm  # từ cellbits directory
[root@exacell01 ~]# rpm -ivh perl-XML-Simple-*.rpm     # từ cellbits directory

# Cài đặt các RPM khác nếu cần
[root@exacell01 cellbits]# rpm -ivh mdadm-*.rpm
[root@exacell01 cellbits]# rpm -ivh python-numeric-*.rpm
```

#### 3.5 Cài đặt Cell Software RPM

```bash
[root@exacell01 cellbits]# rpm -ivh cell-12.1.1.1.0_LINUX.X64_131219-1.x86_64.rpm
Preparing...                ########################################### [100%]
Pre Installation steps in progress ...
   1:cell                   ########################################### [100%]
Post Installation steps in progress ...
Set cellusers group for /opt/oracle/cell12.1.1.1.0_LINUX.X64_131219/cellsrv/deploy/log directory
Set 775 permissions for /opt/oracle/cell12.1.1.1.0_LINUX.X64_131219/cellsrv/deploy/log directory
Installation SUCCESSFUL.
Starting RS and MS... as user celladmin
Done. Please Login as user celladmin and create cell to startup CELLSRV to complete cell configuration.
```

### Bước 4: Tạo storage disks cho exacell01

#### 4.1 Tạo virtual disks (thực hiện từ host OS)

**Tạo 18 Hard Disks (2048MB mỗi disk):**
```cmd
cd "c:\Program Files\Oracle\VirtualBox"
VBoxManage createhd --filename "C:\VM\exacell01\hd1.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd2.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd3.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd4.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd5.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd6.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd7.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd8.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd9.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd10.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd11.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd12.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd13.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd14.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd15.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd16.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd17.vdi" --size 2048 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\hd18.vdi" --size 2048 --format VDI --variant Fixed
```

**Tạo 6 Flash Disks (1536MB mỗi disk):**
```cmd
VBoxManage createhd --filename "C:\VM\exacell01\flash1.vdi" --size 1536 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\flash2.vdi" --size 1536 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\flash3.vdi" --size 1536 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\flash4.vdi" --size 1536 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\flash5.vdi" --size 1536 --format VDI --variant Fixed
VBoxManage createhd --filename "C:\VM\exacell01\flash6.vdi" --size 1536 --format VDI --variant Fixed
```

#### 4.2 Attach disks vào VM

**Hard Disks:**
```cmd
VBoxManage storageattach exacell01 --storagectl "SATA" --port 1 --device 0 --type hdd --medium "C:\VM\exacell01\hd1.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 2 --device 0 --type hdd --medium "C:\VM\exacell01\hd2.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 3 --device 0 --type hdd --medium "C:\VM\exacell01\hd3.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 4 --device 0 --type hdd --medium "C:\VM\exacell01\hd4.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 5 --device 0 --type hdd --medium "C:\VM\exacell01\hd5.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 6 --device 0 --type hdd --medium "C:\VM\exacell01\hd6.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 7 --device 0 --type hdd --medium "C:\VM\exacell01\hd7.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 8 --device 0 --type hdd --medium "C:\VM\exacell01\hd8.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 9 --device 0 --type hdd --medium "C:\VM\exacell01\hd9.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 10 --device 0 --type hdd --medium "C:\VM\exacell01\hd10.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 11 --device 0 --type hdd --medium "C:\VM\exacell01\hd11.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 12 --device 0 --type hdd --medium "C:\VM\exacell01\hd12.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 13 --device 0 --type hdd --medium "C:\VM\exacell01\hd13.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 14 --device 0 --type hdd --medium "C:\VM\exacell01\hd14.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 15 --device 0 --type hdd --medium "C:\VM\exacell01\hd15.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 16 --device 0 --type hdd --medium "C:\VM\exacell01\hd16.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 17 --device 0 --type hdd --medium "C:\VM\exacell01\hd17.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 18 --device 0 --type hdd --medium "C:\VM\exacell01\hd18.vdi" --mtype normal
```

**Flash Disks:**
```cmd
VBoxManage storageattach exacell01 --storagectl "SATA" --port 19 --device 0 --type hdd --medium "C:\VM\exacell01\flash1.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 20 --device 0 --type hdd --medium "C:\VM\exacell01\flash2.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 21 --device 0 --type hdd --medium "C:\VM\exacell01\flash3.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 22 --device 0 --type hdd --medium "C:\VM\exacell01\flash4.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 23 --device 0 --type hdd --medium "C:\VM\exacell01\flash5.vdi" --mtype normal
VBoxManage storageattach exacell01 --storagectl "SATA" --port 24 --device 0 --type hdd --medium "C:\VM\exacell01\flash6.vdi" --mtype normal
```

### Bước 5: Cấu hình Cell Storage cho exacell01

#### 5.1 Tạo directories và symbolic links

```bash
# Tạo directories
[root@exacell01 ~]# mkdir -p /opt/oracle/cell12.1.1.1.0_LINUX.X64_131219/disks/raw
[root@exacell01 ~]# cd /opt/oracle/cell12.1.1.1.0_LINUX.X64_131219/disks/raw

# Kiểm tra disks
[root@exacell01 raw]# fdisk -l 2>/dev/null | grep 'GB\|MB'

# Tạo symbolic links cho Hard Disks (18 disks)
[root@exacell01 raw]# ln -s /dev/sdb exacell01_DISK01
[root@exacell01 raw]# ln -s /dev/sdc exacell01_DISK02
[root@exacell01 raw]# ln -s /dev/sdd exacell01_DISK03
[root@exacell01 raw]# ln -s /dev/sde exacell01_DISK04
[root@exacell01 raw]# ln -s /dev/sdf exacell01_DISK05
[root@exacell01 raw]# ln -s /dev/sdg exacell01_DISK06
[root@exacell01 raw]# ln -s /dev/sdh exacell01_DISK07
[root@exacell01 raw]# ln -s /dev/sdi exacell01_DISK08
[root@exacell01 raw]# ln -s /dev/sdj exacell01_DISK09
[root@exacell01 raw]# ln -s /dev/sdk exacell01_DISK10
[root@exacell01 raw]# ln -s /dev/sdl exacell01_DISK11
[root@exacell01 raw]# ln -s /dev/sdm exacell01_DISK12
[root@exacell01 raw]# ln -s /dev/sdn exacell01_DISK13
[root@exacell01 raw]# ln -s /dev/sdo exacell01_DISK14
[root@exacell01 raw]# ln -s /dev/sdp exacell01_DISK15
[root@exacell01 raw]# ln -s /dev/sdq exacell01_DISK16
[root@exacell01 raw]# ln -s /dev/sdr exacell01_DISK17
[root@exacell01 raw]# ln -s /dev/sds exacell01_DISK18

# Tạo symbolic links cho Flash Disks (6 disks)
[root@exacell01 raw]# ln -s /dev/sdt exacell01_FLASH01
[root@exacell01 raw]# ln -s /dev/sdu exacell01_FLASH02
[root@exacell01 raw]# ln -s /dev/sdv exacell01_FLASH03
[root@exacell01 raw]# ln -s /dev/sdw exacell01_FLASH04
[root@exacell01 raw]# ln -s /dev/sdx exacell01_FLASH05
[root@exacell01 raw]# ln -s /dev/sdy exacell01_FLASH06

# Kiểm tra symbolic links
[root@exacell01 raw]# ls -ltr
```

#### 5.2 Kiểm tra Cell service status

```bash
[root@exacell01 ~]# service celld status
rsStatus: running
msStatus: running
cellsrvStatus: stopped
```

#### 5.3 Tạo Cell và CellDisks

```bash
# Switch to celladmin user
[root@exacell01 ~]# su - celladmin

# Restart services
[celladmin@exacell01 ~]$ cellcli -e "alter cell restart services all"
Stopping the RS, CELLSRV, and MS services...
The SHUTDOWN of services was successful.
Starting the RS, CELLSRV, and MS services...
Getting the state of RS services... running
Starting CELLSRV services...
The STARTUP of CELLSRV services was successful.
Starting MS services...
The STARTUP of MS services was successful.

# Tạo cell với interconnect
[celladmin@exacell01 ~]$ cellcli -e "create cell exacell01 interconnect1=eth1"
Cell exacell01 successfully created
Starting CELLSRV services...
The STARTUP of CELLSRV services was successful.
Flash cell disks, FlashCache, and FlashLog will be created...
CellDisk FD_00_exacell01 successfully created
CellDisk FD_01_exacell01 successfully created
CellDisk FD_02_exacell01 successfully created
CellDisk FD_03_exacell01 successfully created
CellDisk FD_04_exacell01 successfully created
CellDisk FD_05_exacell01 successfully created
Flash log exacell01_FLASHLOG successfully created
Flash cache exacell01_FLASHCACHE successfully created

# Tạo tất cả cell disks
[celladmin@exacell01 ~]$ cellcli -e "create celldisk all"
CellDisk CD_DISK01_exacell01 successfully created
CellDisk CD_DISK02_exacell01 successfully created
CellDisk CD_DISK03_exacell01 successfully created
CellDisk CD_DISK04_exacell01 successfully created
CellDisk CD_DISK05_exacell01 successfully created
CellDisk CD_DISK06_exacell01 successfully created
CellDisk CD_DISK07_exacell01 successfully created
CellDisk CD_DISK08_exacell01 successfully created
CellDisk CD_DISK09_exacell01 successfully created
CellDisk CD_DISK10_exacell01 successfully created
CellDisk CD_DISK11_exacell01 successfully created
CellDisk CD_DISK12_exacell01 successfully created
CellDisk CD_DISK13_exacell01 successfully created
CellDisk CD_DISK14_exacell01 successfully created
CellDisk CD_DISK15_exacell01 successfully created
CellDisk CD_DISK16_exacell01 successfully created
CellDisk CD_DISK17_exacell01 successfully created
CellDisk CD_DISK18_exacell01 successfully created

# Kiểm tra
[celladmin@exacell01 ~]$ cellcli -e "list celldisk"
```

## Phần III: Tạo 4 Cell Nodes còn lại (exacell02, exacell03, exacell04, exacell05)

### Phương pháp Clone VM (Khuyến nghị)

#### Bước 1: Clone exacell01 để tạo các cell còn lại

1. **Tắt exacell01 VM**
2. **Clone VM trong VirtualBox:**
   - Right-click exacell01 → Clone
   - Tạo lần lượt: exacell02, exacell03, exacell04, exacell05
   - MAC Address Policy: Generate new MAC addresses
   - Clone type: Full clone

#### Bước 2: Cấu hình từng cell riêng biệt

**Script tự động cấu hình cell:**

```bash
#!/bin/bash
# configure_cell.sh

CELL_NAME=$1
CELL_NUM=$2

if [ -z "$CELL_NAME" ] || [ -z "$CELL_NUM" ]; then
    echo "Usage: $0 <cell_name> <cell_number>"
    echo "Example: $0 exacell02 02"
    exit 1
fi

echo "Configuring $CELL_NAME..."

# Thay đổi hostname
hostnamectl set-hostname ${CELL_NAME}.localdomain
echo "HOSTNAME=${CELL_NAME}.localdomain" > /etc/sysconfig/network

# Cập nhật IP address trong network config files (nếu cần)
PRIVATE_IP="192.168.2.6${CELL_NUM}"
sed -i "s/192\.168\.2\.61/$PRIVATE_IP/g" /etc/sysconfig/network-scripts/ifcfg-eth1

# Xóa symbolic links cũ
cd /opt/oracle/cell12.1.1.1.0_LINUX.X64_131219/disks/raw
rm -f exacell01_*

# Tạo symbolic links mới
for i in {01..18}; do
    ln -s /dev/sd$(printf "%c" $((98+i-1))) ${CELL_NAME}_DISK${i}
done

# Tạo flash disk symbolic links
for i in {01..06}; do
    flash_dev=$(printf "%c" $((115+i)))  # sdt, sdu, sdv, sdw, sdx, sdy
    ln -s /dev/sd${flash_dev} ${CELL_NAME}_FLASH${i}
done

echo "Symbolic links created for $CELL_NAME"
echo "Please reboot and configure cell services manually"
```

#### Bước 3: Cấu hình chi tiết cho từng cell

**Cho exacell02:**
```bash
[root@exacell02 ~]# ./configure_cell.sh exacell02 02

# Sau khi reboot
[root@exacell02 ~]# su - celladmin
[celladmin@exacell02 ~]$ cellcli -e "alter cell restart services all"
[celladmin@exacell02 ~]$ cellcli -e "create cell exacell02 interconnect1=eth1"
[celladmin@exacell02 ~]$ cellcli -e "create celldisk all"
```

**Cho exacell03:**
```bash
[root@exacell03 ~]# ./configure_cell.sh exacell03 03

# Sau khi reboot
[root@exacell03 ~]# su - celladmin
[celladmin@exacell03 ~]$ cellcli -e "alter cell restart services all"
[celladmin@exacell03 ~]$ cellcli -e "create cell exacell03 interconnect1=eth1"
[celladmin@exacell03 ~]$ cellcli -e "create celldisk all"
```

**Cho exacell04:**
```bash
[root@exacell04 ~]# ./configure_cell.sh exacell04 04

# Sau khi reboot
[root@exacell04 ~]# su - celladmin
[celladmin@exacell04 ~]$ cellcli -e "alter cell restart services all"
[celladmin@exacell04 ~]$ cellcli -e "create cell exacell04 interconnect1=eth1"
[celladmin@exacell04 ~]$ cellcli -e "create celldisk all"
```

**Cho exacell05:**
```bash
[root@exacell05 ~]# ./configure_cell.sh exacell05 05

# Sau khi reboot
[root@exacell05 ~]# su - celladmin
[celladmin@exacell05 ~]$ cellcli -e "alter cell restart services all"
[celladmin@exacell05 ~]$ cellcli -e "create cell exacell05 interconnect1=eth1"
[celladmin@exacell05 ~]$ cellcli -e "create celldisk all"
```

## Phần IV: Tạo Grid Disks cho ASM (Tất cả 5 Cells)

### Bước 1: Script tạo Grid Disks tự động cho tất cả cells

```bash
#!/bin/bash
# create_all_griddisks.sh

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

create_griddisks_for_cell() {
    CELL_NAME=$1
    
    echo "Creating Grid Disks for $CELL_NAME..."
    
    # Tạo DATA Grid Disks (80% - 1600MB trên mỗi 2GB disk)
    cat > create_data_griddisk_${CELL_NAME}.cli << EOF
create griddisk DATA_CD_DISK01_${CELL_NAME} celldisk=CD_DISK01_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK02_${CELL_NAME} celldisk=CD_DISK02_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK03_${CELL_NAME} celldisk=CD_DISK03_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK04_${CELL_NAME} celldisk=CD_DISK04_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK05_${CELL_NAME} celldisk=CD_DISK05_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK06_${CELL_NAME} celldisk=CD_DISK06_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK07_${CELL_NAME} celldisk=CD_DISK07_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK08_${CELL_NAME} celldisk=CD_DISK08_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK09_${CELL_NAME} celldisk=CD_DISK09_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK10_${CELL_NAME} celldisk=CD_DISK10_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK11_${CELL_NAME} celldisk=CD_DISK11_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK12_${CELL_NAME} celldisk=CD_DISK12_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK13_${CELL_NAME} celldisk=CD_DISK13_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK14_${CELL_NAME} celldisk=CD_DISK14_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK15_${CELL_NAME} celldisk=CD_DISK15_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK16_${CELL_NAME} celldisk=CD_DISK16_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK17_${CELL_NAME} celldisk=CD_DISK17_${CELL_NAME}, size=1600m
create griddisk DATA_CD_DISK18_${CELL_NAME} celldisk=CD_DISK18_${CELL_NAME}, size=1600m
EOF

    # Tạo FRA Grid Disks (20% - 400MB trên mỗi 2GB disk)
    cat > create_fra_griddisk_${CELL_NAME}.cli << EOF
create griddisk FRA_CD_DISK01_${CELL_NAME} celldisk=CD_DISK01_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK02_${CELL_NAME} celldisk=CD_DISK02_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK03_${CELL_NAME} celldisk=CD_DISK03_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK04_${CELL_NAME} celldisk=CD_DISK04_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK05_${CELL_NAME} celldisk=CD_DISK05_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK06_${CELL_NAME} celldisk=CD_DISK06_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK07_${CELL_NAME} celldisk=CD_DISK07_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK08_${CELL_NAME} celldisk=CD_DISK08_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK09_${CELL_NAME} celldisk=CD_DISK09_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK10_${CELL_NAME} celldisk=CD_DISK10_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK11_${CELL_NAME} celldisk=CD_DISK11_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK12_${CELL_NAME} celldisk=CD_DISK12_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK13_${CELL_NAME} celldisk=CD_DISK13_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK14_${CELL_NAME} celldisk=CD_DISK14_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK15_${CELL_NAME} celldisk=CD_DISK15_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK16_${CELL_NAME} celldisk=CD_DISK16_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK17_${CELL_NAME} celldisk=CD_DISK17_${CELL_NAME}, size=400m
create griddisk FRA_CD_DISK18_${CELL_NAME} celldisk=CD_DISK18_${CELL_NAME}, size=400m
EOF

    # Thực thi scripts
    echo "Executing Grid Disk creation for $CELL_NAME..."
    ssh root@$CELL_NAME "
        su - celladmin -c 'cellcli @create_data_griddisk_${CELL_NAME}.cli'
        su - celladmin -c 'cellcli @create_fra_griddisk_${CELL_NAME}.cli'
    "
    
    echo "Grid Disks creation completed for $CELL_NAME"
}

# Thực thi cho tất cả cells
for cell in "${CELLS[@]}"; do
    create_griddisks_for_cell $cell
    sleep 10  # Nghỉ giữa các cell
done

echo "=== All Grid Disks created for all 5 cells ==="
```

### Bước 2: Thực thi script

```bash
[root@management_server ~]# chmod +x create_all_griddisks.sh
[root@management_server ~]# ./create_all_griddisks.sh
```

### Bước 3: Kiểm tra Grid Disks trên từng cell

```bash
#!/bin/bash
# check_griddisks.sh

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

for cell in "${CELLS[@]}"; do
    echo "=== Checking Grid Disks on $cell ==="
    ssh root@$cell "
        echo 'Total Grid Disks:'
        su - celladmin -c 'cellcli -e \"list griddisk\"' | wc -l
        
        echo 'DATA Grid Disks:'
        su - celladmin -c 'cellcli -e \"list griddisk where name like DATA.*\"' | wc -l
        
        echo 'FRA Grid Disks:'
        su - celladmin -c 'cellcli -e \"list griddisk where name like FRA.*\"' | wc -l
        
        echo 'Grid Disks Status:'
        su - celladmin -c 'cellcli -e \"list griddisk attributes status\"' | sort | uniq -c
    "
    echo ""
done
```

## Phần V: Validation và kiểm tra hệ thống

### Bước 1: Script kiểm tra trạng thái tất cả Cells

```bash
#!/bin/bash
# validate_all_cells.sh

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

echo "=== EXADATA 5-CELL VALIDATION ==="
echo "Date: $(date)"
echo "Software Version: 12.1.1.1.0"
echo "========================================"

for cell in "${CELLS[@]}"; do
    echo ""
    echo "--- Validating $cell ---"
    
    # Network connectivity test
    if ping -c 2 $cell &> /dev/null; then
        echo "✓ $cell is reachable"
        
        # Detailed validation
        ssh root@$cell "
            echo 'Cell Services Status:'
            su - celladmin -c 'cellcli -e \"list cell attributes name,cellsrvStatus,msStatus,rsStatus\"'
            
            echo ''
            echo 'Cell Version:'
            su - celladmin -c 'cellcli -e \"list cell attributes cellVersion\"'
            
            echo ''
            echo 'Cell Disks Summary:'
            echo 'Total Cell Disks:' \$(su - celladmin -c 'cellcli -e \"list celldisk\"' | wc -l)
            echo 'Hard Disks:' \$(su - celladmin -c 'cellcli -e \"list celldisk where disktype=harddisk\"' | wc -l)
            echo 'Flash Disks:' \$(su - celladmin -c 'cellcli -e \"list celldisk where disktype=flashdisk\"' | wc -l)
            
            echo ''
            echo 'Grid Disks Summary:'
            echo 'Total Grid Disks:' \$(su - celladmin -c 'cellcli -e \"list griddisk\"' | wc -l)
            echo 'DATA Grid Disks:' \$(su - celladmin -c 'cellcli -e \"list griddisk where name like DATA.*\"' | wc -l)
            echo 'FRA Grid Disks:' \$(su - celladmin -c 'cellcli -e \"list griddisk where name like FRA.*\"' | wc -l)
            
            echo ''
            echo 'Flash Cache Status:'
            su - celladmin -c 'cellcli -e \"list flashcache attributes name,status\"'
            
            echo ''
            echo 'Memory Usage:'
            free -m | awk 'NR==2{printf \"Memory Usage: %s/%sMB (%.2f%%)\n\", \$3,\$2,\$3*100/\$2}'
            
            echo ''
            echo 'RDS Modules:'
            lsmod | grep rds | wc -l
            echo 'RDS modules loaded'
        "
    else
        echo "✗ $cell is not reachable"
    fi
    echo "-----------------------------"
done

echo ""
echo "=== VALIDATION SUMMARY ==="
TOTAL_DATA_DISKS=$((18 * 5))  # 18 disks per cell × 5 cells
TOTAL_FRA_DISKS=$((18 * 5))   # 18 disks per cell × 5 cells
TOTAL_CAPACITY_GB=$((2 * 18 * 5))  # 2GB per disk × 18 disks × 5 cells

echo "Expected Configuration:"
echo "- Total Cells: 5"
echo "- Total Hard Disks per Cell: 18"
echo "- Total Flash Disks per Cell: 6"
echo "- Total DATA Grid Disks: $TOTAL_DATA_DISKS"
echo "- Total FRA Grid Disks: $TOTAL_FRA_DISKS"
echo "- Total Raw Capacity: ${TOTAL_CAPACITY_GB}GB"
echo "- Software Version: 12.1.1.1.0"
echo ""
echo "=== VALIDATION COMPLETED ==="
```

### Bước 2: Kiểm tra Flash Cache và Flash Log

```bash
#!/bin/bash
# check_flash_components.sh

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

echo "=== FLASH COMPONENTS STATUS ==="

for cell in "${CELLS[@]}"; do
    echo ""
    echo "--- $cell Flash Components ---"
    ssh root@$cell "
        echo 'Flash Cache:'
        su - celladmin -c 'cellcli -e \"list flashcache detail\"'
        
        echo ''
        echo 'Flash Log:'
        su - celladmin -c 'cellcli -e \"list flashlog detail\"'
        
        echo ''
        echo 'Flash Disks Status:'
        su - celladmin -c 'cellcli -e \"list celldisk where disktype=flashdisk attributes name,status\"'
    "
done
```

## Phần VI: Scripts quản lý hệ thống

### Script startup hệ thống 5-cell

```bash
#!/bin/bash
# startup_5cell_exadata.sh

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

echo "=== Starting 5-Cell Exadata Environment ==="
echo "Software Version: 12.1.1.1.0"

# Bước 1: Khởi động tất cả VMs
echo "Step 1: Starting all VMs..."
for cell in "${CELLS[@]}"; do
    echo "Starting $cell VM..."
    # VBoxManage startvm $cell --type headless
    sleep 15  # Stagger startup to avoid resource contention
done

# Chờ VMs boot up
echo "Step 2: Waiting for VMs to boot up (3 minutes)..."
sleep 180

# Bước 3: Kiểm tra network connectivity
echo "Step 3: Checking network connectivity..."
for cell in "${CELLS[@]}"; do
    while ! ping -c 1 $cell &> /dev/null; do
        echo "Waiting for $cell to be online..."
        sleep 15
    done
    echo "$cell is online ✓"
done

# Bước 4: Load RDS modules trên tất cả cells
echo "Step 4: Loading RDS modules on all cells..."
for cell in "${CELLS[@]}"; do
    echo "Loading RDS modules on $cell..."
    ssh root@$cell "
        modprobe rds 2>/dev/null
        modprobe rds_tcp 2>/dev/null
        modprobe rds_rdma 2>/dev/null
        echo 'RDS modules loaded on $cell'
    " &
done
wait

# Bước 5: Start Cell services
echo "Step 5: Starting Cell services on all nodes..."
for cell in "${CELLS[@]}"; do
    echo "Starting services on $cell..."
    ssh root@$cell "
        service celld start
        sleep 10
        su - celladmin -c 'cellcli -e \"alter cell restart services all\"'
        echo 'Services started on $cell'
    " &
done
wait

# Bước 6: Final verification
echo "Step 6: Verifying all services..."
sleep 30
for cell in "${CELLS[@]}"; do
    echo "Checking $cell..."
    ssh root@$cell "
        su - celladmin -c 'cellcli -e \"list cell attributes name,cellsrvStatus,msStatus,rsStatus\"'
    "
done

echo ""
echo "=== 5-Cell Exadata Startup Completed ==="
echo "All cells should now be online and ready"
```

### Script shutdown hệ thống 5-cell

```bash
#!/bin/bash
# shutdown_5cell_exadata.sh

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

echo "=== Shutting down 5-Cell Exadata Environment ==="

# Bước 1: Stop Cell services gracefully
echo "Step 1: Stopping Cell services on all nodes..."
for cell in "${CELLS[@]}"; do
    echo "Stopping services on $cell..."
    ssh root@$cell "
        su - celladmin -c 'cellcli -e \"alter cell shutdown services all\"' 2>/dev/null
        service celld stop 2>/dev/null
        echo 'Services stopped on $cell'
    " &
done
wait

# Bước 2: Shutdown OS
echo "Step 2: Shutting down operating systems..."
for cell in "${CELLS[@]}"; do
    echo "Shutting down $cell..."
    ssh root@$cell "nohup shutdown -h +1 > /dev/null 2>&1 &" &
    sleep 5
done

# Chờ shutdown hoàn tất
echo "Step 3: Waiting for shutdown to complete (2 minutes)..."
sleep 120

# Bước 4: Force power off VMs if needed
echo "Step 4: Ensuring VMs are powered off..."
for cell in "${CELLS[@]}"; do
    # VBoxManage controlvm $cell poweroff 2>/dev/null
    echo "$cell powered off"
done

echo ""
echo "=== 5-Cell Exadata Shutdown Completed ==="
```

### Script monitoring real-time

```bash
#!/bin/bash
# monitor_5cell_realtime.sh

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

monitor_cells() {
    clear
    echo "=== REAL-TIME 5-CELL EXADATA MONITORING ==="
    echo "Time: $(date)"
    echo "Software: Oracle Storage Server 12.1.1.1.0"
    echo "============================================="
    
    for cell in "${CELLS[@]}"; do
        echo ""
        echo "--- $cell ---"
        
        if ping -c 1 $cell &> /dev/null; then
            ssh -o ConnectTimeout=5 root@$cell "
                # System Resources
                echo 'CPU: '\$(top -bn1 | grep 'Cpu(s)' | sed 's/.*, *\([0-9.]*\)%* id.*/\1/' | awk '{print 100 - \$1\"%\"}')
                echo 'Memory: '\$(free -m | awk 'NR==2{printf \"%.1f%%\", \$3*100/\$2}')
                echo 'Load: '\$(uptime | awk -F'load average:' '{print \$2}')
                
                # Cell Services
                echo 'Services: '\$(su - celladmin -c 'cellcli -e \"list cell attributes cellsrvStatus,msStatus,rsStatus\"' 2>/dev/null | tail -n 1 | tr -d '\n')
                
                # Grid Disks
                ACTIVE_GD=\$(su - celladmin -c 'cellcli -e \"list griddisk where status=active\"' 2>/dev/null | wc -l)
                TOTAL_GD=\$(su - celladmin -c 'cellcli -e \"list griddisk\"' 2>/dev/null | wc -l)
                echo \"Grid Disks: \${ACTIVE_GD}/\${TOTAL_GD} active\"
            " 2>/dev/null || echo "Connection failed"
        else
            echo "❌ OFFLINE"
        fi
    done
    
    echo ""
    echo "Press Ctrl+C to stop monitoring"
}

# Continuous monitoring loop
while true; do
    monitor_cells
    sleep 10
done
```

### Script backup configuration

```bash
#!/bin/bash
# backup_5cell_config.sh

BACKUP_BASE="/backup/exadata_5cell"
BACKUP_DIR="${BACKUP_BASE}/config_$(date +%Y%m%d_%H%M%S)"
CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

echo "=== Backing up 5-Cell Exadata Configuration ==="
echo "Backup directory: $BACKUP_DIR"

mkdir -p $BACKUP_DIR

for cell in "${CELLS[@]}"; do
    echo "Backing up $cell configuration..."
    
    # Tạo thư mục backup cho cell
    mkdir -p $BACKUP_DIR/$cell
    
    # Export CellCLI configurations
    ssh root@$cell "
        su - celladmin -c 'cellcli -e \"list cell detail\"' > /tmp/${cell}_cell_detail.txt 2>/dev/null
        su - celladmin -c 'cellcli -e \"list celldisk\"' > /tmp/${cell}_celldisk_list.txt 2>/dev/null
        su - celladmin -c 'cellcli -e \"list griddisk\"' > /tmp/${cell}_griddisk_list.txt 2>/dev/null
        su - celladmin -c 'cellcli -e \"list flashcache detail\"' > /tmp/${cell}_flashcache_detail.txt 2>/dev/null
        su - celladmin -c 'cellcli -e \"list flashlog detail\"' > /tmp/${cell}_flashlog_detail.txt 2>/dev/null
        su - celladmin -c 'cellcli -e \"list physicaldisk\"' > /tmp/${cell}_physicaldisk_list.txt 2>/dev/null
        
        # System info
        uname -a > /tmp/${cell}_system_info.txt
        df -h > /tmp/${cell}_disk_usage.txt
        free -m > /tmp/${cell}_memory_info.txt
        ifconfig > /tmp/${cell}_network_config.txt
        lsmod | grep rds > /tmp/${cell}_rds_modules.txt
    "
    
    # Copy files từ remote
    scp root@$cell:/tmp/${cell}_*.txt $BACKUP_DIR/$cell/ 2>/dev/null
    
    # Backup system configuration files
    scp root@$cell:/etc/hosts $BACKUP_DIR/$cell/ 2>/dev/null
    scp root@$cell:/etc/sysctl.conf $BACKUP_DIR/$cell/ 2>/dev/null
    scp root@$cell:/etc/security/limits.conf $BACKUP_DIR/$cell/ 2>/dev/null
    scp root@$cell:/opt/oracle/cell*/cellsrv/deploy/config/cellinit.ora $BACKUP_DIR/$cell/ 2>/dev/null
    
    # Cleanup temp files
    ssh root@$cell "rm -f /tmp/${cell}_*.txt"
    
    echo "$cell backup completed ✓"
done

# Tạo summary report
cat > $BACKUP_DIR/backup_summary.txt << EOF
=== EXADATA 5-CELL BACKUP SUMMARY ===
Backup Date: $(date)
Software Version: 12.1.1.1.0
Cells: ${CELLS[@]}

Backup Location: $BACKUP_DIR

Files backed up per cell:
- Cell configuration details
- Cell disk listings
- Grid disk listings  
- Flash cache/log details
- Physical disk information
- System information
- Network configuration
- Important config files

Total backup size: $(du -sh $BACKUP_DIR | cut -f1)
EOF

echo ""
echo "=== Backup completed ==="
echo "Summary: $BACKUP_DIR/backup_summary.txt"
echo "Total size: $(du -sh $BACKUP_DIR | cut -f1)"
```

## Phần VII: Kết nối Database Servers với 5-Cell Environment

### Cấu hình trên Database Server

#### File cellip.ora cho 5 cells

```bash
# /etc/oracle/cell/network-config/cellip.ora
cell="192.168.2.61"
cell="192.168.2.62" 
cell="192.168.2.63"
cell="192.168.2.64"
cell="192.168.2.65"
```

#### File cellinit.ora cho Database Server

```bash
# /etc/oracle/cell/network-config/cellinit.ora
# Cell Initialization Parameters for 12.1.1.1.0
ipaddress1=192.168.2.80/24
_cell_print_all_params=true
_skgxp_gen_rpc_timeout_in_sec=90
_skgxp_gen_ant_off_rpc_timeout_in_sec=300
_skgxp_udp_interface_detection_time_secs=15
_skgxp_udp_use_tcb_client=true
_skgxp_udp_use_tcb=false
_reconnect_to_cell_attempts=5
_cell_fast_file_restore_enabled=true
_asm_allow_only_raw_disks=false
```

#### Script kiểm tra kết nối từ DB Server

```bash
#!/bin/bash
# test_db_cell_connectivity.sh

CELLS=("192.168.2.61" "192.168.2.62" "192.168.2.63" "192.168.2.64" "192.168.2.65")

echo "=== Testing DB Server to Cell Connectivity ==="

# Kiểm tra RDS modules trên DB server
echo "Checking RDS modules on DB server..."
if ! lsmod | grep -q rds; then
    echo "Loading RDS modules..."
    modprobe rds
    modprobe rds_tcp
    modprobe rds_rdma
fi

echo "RDS modules status:"
lsmod | grep rds

echo ""
echo "Testing connectivity to each cell:"
for cell_ip in "${CELLS[@]}"; do
    if ping -c 2 $cell_ip &> /dev/null; then
        echo "✓ $cell_ip is reachable"
    else
        echo "✗ $cell_ip is not reachable"
    fi
done

# Test Grid Disk discovery
echo ""
echo "Testing Grid Disk discovery..."
if [ -f /u01/app/grid/stage/ext/bin/kfod ]; then
    export LD_LIBRARY_PATH=/u01/app/grid/stage/ext/lib
    echo "Discovering disks..."
    /u01/app/grid/stage/ext/bin/kfod disks=all op=disks | head -20
    echo "..."
    echo "Total disks found: $(/u01/app/grid/stage/ext/bin/kfod disks=all op=disks | wc -l)"
else
    echo "Grid Infrastructure not found or not installed"
fi

echo ""
echo "=== Connectivity test completed ==="
```

## Phần VIII: Troubleshooting cho 5-Cell Environment

### Lỗi thường gặp với 5 cells

#### 1. Resource contention khi khởi động đồng thời

**Vấn đề:** 5 VMs khởi động cùng lúc gây quá tải host
**Giải pháp:**
```bash
# Stagger startup script
for i in {1..5}; do
    echo "Starting exacell0$i"
    # VBoxManage startvm exacell0$i --type headless
    sleep 30  # Nghỉ 30s giữa mỗi VM
done
```

#### 2. Network conflicts giữa các cells

**Kiểm tra:**
```bash
# Kiểm tra IP conflicts
for i in {1..5}; do
    ping -c 1 192.168.2.6$i
    ping -c 1 192.168.56.6$i
done
```

#### 3. Storage performance issues

**Vấn đề:** 5 cells với nhiều disks có thể gây chậm I/O
**Giải pháp:**
```bash
# Tối ưu VM settings cho storage
# Trong VirtualBox settings cho mỗi VM:
# - Enable VT-x/AMD-V
# - Enable Nested Paging
# - Increase Video Memory to 128MB
# - Set storage controller to AHCI
# - Enable Host I/O Cache

# Script kiểm tra I/O performance
#!/bin/bash
check_io_performance() {
    CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")
    
    for cell in "${CELLS[@]}"; do
        echo "=== I/O Performance on $cell ==="
        ssh root@$cell "
            echo 'Disk I/O stats:'
            iostat -x 1 3 | tail -20
            
            echo 'Current I/O load:'
            iotop -b -n 1 -o | head -10
        "
    done
}
```

#### 4. Cell services startup failures

**Triển khai script debug:**
```bash
#!/bin/bash
# debug_cell_services.sh

debug_cell() {
    CELL_NAME=$1
    
    echo "=== Debugging $CELL_NAME ==="
    ssh root@$CELL_NAME "
        echo 'Checking celld service:'
        service celld status
        
        echo 'Checking log files:'
        tail -50 /opt/oracle/cell*/cellsrv/deploy/log/cellsrv.log
        
        echo 'Checking cellinit.ora:'
        cat /opt/oracle/cell*/cellsrv/deploy/config/cellinit.ora
        
        echo 'Checking symbolic links:'
        ls -la /opt/oracle/cell*/disks/raw/ | grep -v total
        
        echo 'Checking RDS modules:'
        lsmod | grep rds
        
        echo 'Checking network:'
        ping -c 1 \$(hostname -i)
        
        echo 'Checking disk space:'
        df -h | grep -E '(root|opt)'
    "
}

# Debug all cells
CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")
for cell in "${CELLS[@]}"; do
    debug_cell $cell
    echo "=========================="
done
```

#### 5. Grid Disk creation failures

**Script khắc phục:**
```bash
#!/bin/bash
# fix_griddisk_issues.sh

fix_griddisk() {
    CELL_NAME=$1
    
    echo "Fixing Grid Disk issues on $CELL_NAME..."
    ssh root@$CELL_NAME "
        su - celladmin -c '
            # Xóa tất cả grid disks (nếu có lỗi)
            cellcli -e \"drop griddisk all force\"
            
            # Kiểm tra cell disks
            cellcli -e \"list celldisk\"
            
            # Tạo lại grid disks
            cellcli -e \"create griddisk DATA_CD_DISK01_${CELL_NAME} celldisk=CD_DISK01_${CELL_NAME}, size=1600m\"
            # ... tiếp tục cho tất cả disks
        '
    "
}
```

### Advanced Troubleshooting Scripts

#### Script phân tích log tự động

```bash
#!/bin/bash
# analyze_logs.sh

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")
LOG_DIR="/tmp/cell_logs_$(date +%Y%m%d_%H%M%S)"

mkdir -p $LOG_DIR

echo "=== Collecting logs from all 5 cells ==="

for cell in "${CELLS[@]}"; do
    echo "Collecting logs from $cell..."
    mkdir -p $LOG_DIR/$cell
    
    ssh root@$cell "
        # Cell service logs
        cp /opt/oracle/cell*/cellsrv/deploy/log/cellsrv.log /tmp/${cell}_cellsrv.log 2>/dev/null
        cp /opt/oracle/cell*/cellsrv/deploy/log/ms.log /tmp/${cell}_ms.log 2>/dev/null
        cp /opt/oracle/cell*/cellsrv/deploy/log/rs.log /tmp/${cell}_rs.log 2>/dev/null
        
        # System logs
        cp /var/log/messages /tmp/${cell}_messages.log 2>/dev/null
        dmesg > /tmp/${cell}_dmesg.log
        
        # Performance data
        vmstat 1 5 > /tmp/${cell}_vmstat.log
        iostat -x 1 5 > /tmp/${cell}_iostat.log
    "
    
    # Copy logs to analysis directory
    scp root@$cell:/tmp/${cell}_*.log $LOG_DIR/$cell/ 2>/dev/null
    
    # Cleanup remote temp files
    ssh root@$cell "rm -f /tmp/${cell}_*.log"
done

# Analyze logs
echo ""
echo "=== Log Analysis Summary ==="
echo "Logs collected in: $LOG_DIR"

# Check for common errors
echo ""
echo "Common errors found:"
grep -r "ERROR\|FATAL\|Exception" $LOG_DIR/ | head -10

echo ""
echo "Cell startup issues:"
grep -r "CELL-01547\|startup failed" $LOG_DIR/

echo ""
echo "Network issues:"
grep -r "network\|timeout\|connection" $LOG_DIR/ | head -5
```

## Phần IX: Performance Tuning cho 5-Cell Environment

### Tối ưu hóa VM Settings

```bash
#!/bin/bash
# optimize_vm_settings.sh

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

echo "=== Optimizing VM Settings for 5-Cell Environment ==="

for cell in "${CELLS[@]}"; do
    echo "Optimizing $cell VM settings..."
    
    # VirtualBox optimization commands
    # VBoxManage modifyvm $cell --memory 4096
    # VBoxManage modifyvm $cell --cpus 2
    # VBoxManage modifyvm $cell --vram 128
    # VBoxManage modifyvm $cell --acceleration3d off
    # VBoxManage modifyvm $cell --hwvirtex on
    # VBoxManage modifyvm $cell --nestedpaging on
    # VBoxManage modifyvm $cell --largepages on
    # VBoxManage modifyvm $cell --ioapic on
    # VBoxManage modifyvm $cell --hpet on
    
    # Storage optimization
    # VBoxManage storagectl $cell --name "SATA" --hostiocache on
    # VBoxManage storagectl $cell --name "SATA" --bootable on
    
    echo "$cell VM optimized"
done
```

### Tối ưu hóa OS Settings trên tất cả Cells

```bash
#!/bin/bash
# optimize_os_settings.sh

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

for cell in "${CELLS[@]}"; do
    echo "Optimizing OS settings on $cell..."
    ssh root@$cell "
        # I/O Scheduler optimization
        echo 'deadline' > /sys/block/sda/queue/scheduler
        
        # Network tuning
        echo 'net.core.rmem_max = 134217728' >> /etc/sysctl.conf
        echo 'net.core.wmem_max = 134217728' >> /etc/sysctl.conf
        echo 'net.ipv4.tcp_rmem = 4096 16384 134217728' >> /etc/sysctl.conf
        echo 'net.ipv4.tcp_wmem = 4096 16384 134217728' >> /etc/sysctl.conf
        echo 'net.core.netdev_max_backlog = 30000' >> /etc/sysctl.conf
        
        # Virtual memory tuning
        echo 'vm.dirty_ratio = 15' >> /etc/sysctl.conf
        echo 'vm.dirty_background_ratio = 5' >> /etc/sysctl.conf
        echo 'vm.swappiness = 10' >> /etc/sysctl.conf
        
        # Apply changes
        sysctl -p
        
        # Disable unnecessary services
        chkconfig cups off 2>/dev/null
        chkconfig bluetooth off 2>/dev/null
        chkconfig avahi-daemon off 2>/dev/null
        
        echo 'OS optimization completed on $cell'
    "
done
```

## Phần X: Monitoring và Alerting

### Comprehensive Monitoring Dashboard

```bash
#!/bin/bash
# monitoring_dashboard.sh

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

create_dashboard() {
    while true; do
        clear
        echo "╔══════════════════════════════════════════════════════════════════════════════════════════╗"
        echo "║                            EXADATA 5-CELL MONITORING DASHBOARD                          ║"
        echo "║                              Version: 12.1.1.1.0                                       ║"
        echo "╠══════════════════════════════════════════════════════════════════════════════════════════╣"
        echo "║  Time: $(date '+%Y-%m-%d %H:%M:%S')                                                              ║"
        echo "╚══════════════════════════════════════════════════════════════════════════════════════════╝"
        
        echo ""
        printf "%-12s %-10s %-8s %-8s %-12s %-15s %-10s\n" "CELL" "STATUS" "CPU%" "MEM%" "SERVICES" "GRIDDISKS" "FLASHCACHE"
        echo "────────────────────────────────────────────────────────────────────────────────────────────"
        
        for cell in "${CELLS[@]}"; do
            if ping -c 1 -W 2 $cell &> /dev/null; then
                # Get system stats
                STATS=$(ssh -o ConnectTimeout=3 root@$cell "
                    # CPU Usage
                    CPU=\$(top -bn1 | grep 'Cpu(s)' | sed 's/.*, *\([0-9.]*\)%* id.*/\1/' | awk '{print 100 - \$1}')
                    
                    # Memory Usage  
                    MEM=\$(free -m | awk 'NR==2{printf \"%.0f\", \$3*100/\$2}')
                    
                    # Services Status
                    SERVICES=\$(su - celladmin -c 'cellcli -e \"list cell attributes cellsrvStatus\"' 2>/dev/null | tail -1 | cut -d' ' -f2)
                    
                    # Grid Disks Count
                    GD_ACTIVE=\$(su - celladmin -c 'cellcli -e \"list griddisk where status=active\"' 2>/dev/null | wc -l)
                    GD_TOTAL=\$(su - celladmin -c 'cellcli -e \"list griddisk\"' 2>/dev/null | wc -l)
                    
                    # Flash Cache Status
                    FC_STATUS=\$(su - celladmin -c 'cellcli -e \"list flashcache attributes status\"' 2>/dev/null | tail -1 | cut -d' ' -f2)
                    
                    echo \"\$CPU|\$MEM|\$SERVICES|\${GD_ACTIVE}/\${GD_TOTAL}|\$FC_STATUS\"
                " 2>/dev/null)
                
                if [ -n "$STATS" ]; then
                    IFS='|' read -r cpu mem services griddisks flashcache <<< "$STATS"
                    
                    # Color coding based on status
                    if [[ "$services" == "running" ]]; then
                        status="🟢 ONLINE"
                    else
                        status="🔴 OFFLINE"
                    fi
                    
                    printf "%-12s %-10s %-8s %-8s %-12s %-15s %-10s\n" \
                        "$cell" "$status" "${cpu}%" "${mem}%" "$services" "$griddisks" "$flashcache"
                else
                    printf "%-12s %-10s %-8s %-8s %-12s %-15s %-10s\n" \
                        "$cell" "🟡 LIMITED" "N/A" "N/A" "Unknown" "N/A" "N/A"
                fi
            else
                printf "%-12s %-10s %-8s %-8s %-12s %-15s %-10s\n" \
                    "$cell" "🔴 DOWN" "N/A" "N/A" "N/A" "N/A" "N/A"
            fi
        done
        
        echo ""
        echo "╔════════════════════════════════════════════════════════════════════════════════════════╗"
        echo "║                                   SUMMARY STATISTICS                                  ║"
        echo "╚════════════════════════════════════════════════════════════════════════════════════════╝"
        
        # Calculate totals
        ONLINE_CELLS=0
        TOTAL_GD=0
        
        for cell in "${CELLS[@]}"; do
            if ping -c 1 -W 1 $cell &> /dev/null; then
                ((ONLINE_CELLS++))
                GD_COUNT=$(ssh -o ConnectTimeout=2 root@$cell "su - celladmin -c 'cellcli -e \"list griddisk\"' 2>/dev/null | wc -l" 2>/dev/null)
                if [[ "$GD_COUNT" =~ ^[0-9]+$ ]]; then
                    ((TOTAL_GD += GD_COUNT))
                fi
            fi
        done
        
        echo "Online Cells: $ONLINE_CELLS/5"
        echo "Total Grid Disks: $TOTAL_GD"
        echo "Expected Grid Disks: $((36 * 5)) (36 per cell × 5 cells)"
        echo ""
        echo "Press Ctrl+C to exit | Refresh every 10 seconds"
        
        sleep 10
    done
}

# Start dashboard
create_dashboard
```

### Alert System

```bash
#!/bin/bash
# alert_system.sh

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")
ALERT_LOG="/var/log/exadata_alerts.log"
EMAIL_ALERTS="admin@company.com"  # Configure email if needed

check_alerts() {
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    for cell in "${CELLS[@]}"; do
        # Check if cell is reachable
        if ! ping -c 2 -W 5 $cell &> /dev/null; then
            echo "[$timestamp] CRITICAL: $cell is not reachable" | tee -a $ALERT_LOG
            continue
        fi
        
        # Check cell services
        SERVICES_STATUS=$(ssh -o ConnectTimeout=5 root@$cell \
            "su - celladmin -c 'cellcli -e \"list cell attributes cellsrvStatus,msStatus,rsStatus\"' 2>/dev/null" \
            | tail -1)
        
        if [[ ! "$SERVICES_STATUS" =~ "running.*running.*running" ]]; then
            echo "[$timestamp] WARNING: $cell services not all running: $SERVICES_STATUS" | tee -a $ALERT_LOG
        fi
        
        # Check memory usage
        MEM_USAGE=$(ssh -o ConnectTimeout=5 root@$cell \
            "free -m | awk 'NR==2{printf \"%.0f\", \$3*100/\$2}'" 2>/dev/null)
        
        if [[ "$MEM_USAGE" =~ ^[0-9]+$ ]] && [ "$MEM_USAGE" -gt 90 ]; then
            echo "[$timestamp] WARNING: $cell high memory usage: ${MEM_USAGE}%" | tee -a $ALERT_LOG
        fi
        
        # Check CPU usage
        CPU_USAGE=$(ssh -o ConnectTimeout=5 root@$cell \
            "top -bn1 | grep 'Cpu(s)' | sed 's/.*, *\([0-9.]*\)%* id.*/\1/' | awk '{print 100 - \$1}'" 2>/dev/null)
        
        if [[ "$CPU_USAGE" =~ ^[0-9.]+$ ]] && [ "${CPU_USAGE%.*}" -gt 80 ]; then
            echo "[$timestamp] WARNING: $cell high CPU usage: ${CPU_USAGE}%" | tee -a $ALERT_LOG
        fi
        
        # Check Grid Disks
        GD_OFFLINE=$(ssh -o ConnectTimeout=5 root@$cell \
            "su - celladmin -c 'cellcli -e \"list griddisk where status!=active\"' 2>/dev/null | wc -l" 2>/dev/null)
        
        if [[ "$GD_OFFLINE" =~ ^[0-9]+$ ]] && [ "$GD_OFFLINE" -gt 0 ]; then
            echo "[$timestamp] CRITICAL: $cell has $GD_OFFLINE offline grid disks" | tee -a $ALERT_LOG
        fi
    done
}

# Continuous monitoring
echo "Starting Exadata 5-Cell Alert System..."
echo "Alert log: $ALERT_LOG"

while true; do
    check_alerts
    sleep 60  # Check every minute
done
```

## Phần XI: Backup và Disaster Recovery

### Automated Backup Strategy

```bash
#!/bin/bash
# comprehensive_backup.sh

BACKUP_BASE="/backup/exadata_5cell"
RETENTION_DAYS=30
CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

echo "=== Comprehensive 5-Cell Backup Strategy ==="

# Create timestamped backup directory
BACKUP_DIR="${BACKUP_BASE}/full_backup_$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

# Function to backup single cell
backup_cell() {
    local cell=$1
    local cell_backup_dir="$BACKUP_DIR/$cell"
    
    echo "Backing up $cell..."
    mkdir -p $cell_backup_dir
    
    # 1. CellCLI Configuration Export
    ssh root@$cell "
        su - celladmin -c '
            cellcli -e \"export celldisk all\" > /tmp/celldisk_export.xml 2>/dev/null
            cellcli -e \"export griddisk all\" > /tmp/griddisk_export.xml 2>/dev/null
            cellcli -e \"list cell detail\" > /tmp/cell_detail.txt 2>/dev/null
            cellcli -e \"list physicaldisk\" > /tmp/physicaldisk_list.txt 2>/dev/null
            cellcli -e \"list flashcache detail\" > /tmp/flashcache_detail.txt 2>/dev/null
            cellcli -e \"list flashlog detail\" > /tmp/flashlog_detail.txt 2>/dev/null
            cellcli -e \"list alerthistory\" > /tmp/alert_history.txt 2>/dev/null
        '
        
        # System configuration backup
        tar czf /tmp/system_config.tar.gz \
            /etc/hosts \
            /etc/sysctl.conf \
            /etc/security/limits.conf \
            /etc/modprobe.d/rds.conf \
            /opt/oracle/cell*/cellsrv/deploy/config/ \
            /opt/oracle/cell*/disks/raw/ 2>/dev/null
            
        # Network configuration
        tar czf /tmp/network_config.tar.gz \
            /etc/sysconfig/network* \
            /etc/resolv.conf 2>/dev/null
    "
    
    # Copy all backup files
    scp root@$cell:/tmp/*.xml $cell_backup_dir/ 2>/dev/null
    scp root@$cell:/tmp/*.txt $cell_backup_dir/ 2>/dev/null
    scp root@$cell:/tmp/*.tar.gz $cell_backup_dir/ 2>/dev/null
    
    # Cleanup remote temp files
    ssh root@$cell "rm -f /tmp/*.xml /tmp/*.txt /tmp/*.tar.gz"
    
    # Create cell-specific restore script
    cat > $cell_backup_dir/restore_$cell.sh << EOF
#!/bin/bash
# Restore script for $cell
# Generated: $(date)

echo "Restoring $cell configuration..."

# Upload and extract system configs
scp system_config.tar.gz root@$cell:/tmp/
scp network_config.tar.gz root@$cell:/tmp/

ssh root@$cell "
    cd /
    tar xzf /tmp/system_config.tar.gz
    tar xzf /tmp/network_config.tar.gz
    
    # Restart network
    service network restart
    
    # Load RDS modules
    modprobe rds
    modprobe rds_tcp
    modprobe rds_rdma
    
    # Restart cell services
    service celld restart
    sleep 10
    su - celladmin -c 'cellcli -e \"alter cell restart services all\"'
"

echo "$cell restore completed"
EOF
    
    chmod +x $cell_backup_dir/restore_$cell.sh
    echo "$cell backup completed ✓"
}

# Backup all cells in parallel
echo "Starting parallel backup of all 5 cells..."
for cell in "${CELLS[@]}"; do
    backup_cell $cell &
done
wait

# Create master restore script
cat > $BACKUP_DIR/restore_all_cells.sh << EOF
#!/bin/bash
# Master restore script for all 5 cells
# Generated: $(date)
# Backup location: $BACKUP_DIR

CELLS=("${CELLS[@]}")

echo "=== Restoring All 5 Cells ==="
echo "Backup date: $(basename $BACKUP_DIR | cut -d'_' -f3-4)"

for cell in "\${CELLS[@]}"; do
    echo "Restoring \$cell..."
    cd $BACKUP_DIR/\$cell
    ./restore_\$cell.sh
    sleep 30
done

echo "All cells restored. Please verify services manually."
EOF

chmod +x $BACKUP_DIR/restore_all_cells.sh

# Create backup summary
cat > $BACKUP_DIR/backup_summary.html << EOF
<!DOCTYPE html>
<html>
<head>
    <title>Exadata 5-Cell Backup Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .success { color: green; }
        .error { color: red; }
    </style>
</head>
<body>
    <h1>Exadata 5-Cell Backup Report</h1>
    <p><strong>Backup Date:</strong> $(date)</p>
    <p><strong>Software Version:</strong> 12.1.1.1.0</p>
    <p><strong>Backup Location:</strong> $BACKUP_DIR</p>
    
    <h2>Backup Summary</h2>
    <table>
        <tr><th>Cell</th><th>Status</th><th>Files</th><th>Size</th></tr>
EOF

# Add each cell to summary
for cell in "${CELLS[@]}"; do
    if [ -d "$BACKUP_DIR/$cell" ]; then
        file_count=$(ls -1 $BACKUP_DIR/$cell | wc -l)
        size=$(du -sh $BACKUP_DIR/$cell | cut -f1)
        echo "        <tr><td>$cell</td><td class=\"success\">Success</td><td>$file_count</td><td>$size</td></tr>" >> $BACKUP_DIR/backup_summary.html
    else
        echo "        <tr><td>$cell</td><td class=\"error\">Failed</td><td>0</td><td>0</td></tr>" >> $BACKUP_DIR/backup_summary.html
    fi
done

cat >> $BACKUP_DIR/backup_summary.html << EOF
    </table>
    
    <h2>Backup Contents</h2>
    <ul>
        <li>Cell configurations (CellCLI exports)</li>
        <li>Grid disk definitions</li>
        <li>Physical disk layouts</li>
        <li>Flash cache/log configurations</li>
        <li>System configuration files</li>
        <li>Network configuration</li>
        <li>Alert history</li>
        <li>Restore scripts</li>
    </ul>
    
    <h2>Restore Instructions</h2>
    <ol>
        <li>Ensure all VMs are powered on and accessible</li>
        <li>Run: <code>$BACKUP_DIR/restore_all_cells.sh</code></li>
        <li>Verify cell services: <code>cellcli -e "list cell detail"</code></li>
        <li>Check grid disks: <code>cellcli -e "list griddisk"</code></li>
    </ol>
    
    <p><strong>Total Backup Size:</strong> $(du -sh $BACKUP_DIR | cut -f1)</p>
    <p><em>Generated by Exadata 5-Cell Management System</em></p>
</body>
</html>
EOF

# Cleanup old backups
echo "Cleaning up backups older than $RETENTION_DAYS days..."
find $BACKUP_BASE -type d -name "full_backup_*" -mtime +$RETENTION_DAYS -exec rm -rf {} \; 2>/dev/null

echo ""
echo "=== Backup Completed ==="
echo "Location: $BACKUP_DIR"
echo "Size: $(du -sh $BACKUP_DIR | cut -f1)"
echo "Summary: $BACKUP_DIR/backup_summary.html"
echo "Restore: $BACKUP_DIR/restore_all_cells.sh"
```

## Phần XII: Kết luận và Best Practices

### Summary của môi trường 5-Cell

**🎯 Cấu hình hoàn chỉnh:**
- **5 Cell Nodes:** exacell01 → exacell05
- **Software Version:** Oracle Storage Server 12.1.1.1.0
- **Total Storage:** 90 Hard Disks (18 per cell × 5 cells)
- **Flash Storage:** 30 Flash Disks (6 per cell × 5 cells)
- **Total Grid Disks:** 180 (90 DATA + 90 FRA)
- **Total Capacity:** ~180GB raw storage

### Best Practices cho 5-Cell Environment

#### 1. Resource Management
```bash
# Recommended host system requirements:
- RAM: 32GB+ (20GB for VMs + 12GB for host OS)
- CPU: 16+ cores (10 for VMs + 6 for host)
- Storage: 300GB+ fast SSD
- Network: Gigabit Ethernet minimum
```

#### 2. Startup/Shutdown Sequences
```bash
# Always use staggered startup (30s intervals)
# Always shutdown services gracefully before OS shutdown
# Monitor resource usage during operations
```

#### 3. Maintenance Windows
```bash
#!/bin/bash
# maintenance_mode.sh
# Put all cells in maintenance mode

CELLS=("exacell01" "exacell02" "exacell03" "exacell04" "exacell05")

echo "Entering maintenance mode for all cells..."
for cell in "${CELLS[@]}"; do
    ssh root@$cell "
        su - celladmin -c 'cellcli -e \"alter cell shutdown services CELLSRV\"'
        echo '$cell CELLSRV stopped for maintenance'
    "
done

echo "All cells in maintenance mode"
echo "To exit maintenance mode, run startup script"
```

#### 4. Health Check Automation
```bash
# Schedule regular health checks
# 0 */6 * * * /opt/scripts/validate_all_cells.sh >> /var/log/exadata_health.log 2>&1
# 0 2 * * 0 /opt/scripts/comprehensive_backup.sh >> /var/log/exadata_backup.log 2>&1
```

### Final Validation Checklist

**✅ Pre-Production Checklist:**
- [ ] All 5 cells online and services running
- [ ] All 180 Grid Disks active
- [ ] Flash Cache operational on all cells
- [ ] Network connectivity between all cells
- [ ] Database server can discover all Grid Disks
- [ ] Backup procedures tested and working
- [ ] Monitoring systems operational
- [ ] Documentation updated and accessible

**🔧 Recommended Next Steps:**
1. **Install Database Servers** - Configure 2-node RAC
2. **Create ASM Disk Groups** - Use discovered Grid Disks
3. **Install Oracle Database** - 12c or higher
4. **Configure Exadata Features** - Smart Scan, Storage Indexes
5. **Performance Testing** - Validate I/O performance
6. **Production Migration** - Migrate real workloads

### Support và Resources

**📚 Useful Commands Reference:**
```sql
-- Quick status check on all cells
CellCLI> list cell attributes name,status,cellsrvStatus

-- List all Grid Disks across environment  
CellCLI> list griddisk attributes name,cellDisk,status,size

-- Check Flash Cache effectiveness
CellCLI> list metriccurrent where objecttype='FLASHCACHE'

-- Monitor I/O patterns
CellCLI> list metriccurrent where objecttype='GRIDDISK' and name like 'GD_.*_IO_.*'
```

**🎓 Learning Path:**
1. Master CellCLI commands and navigation
2. Understand Smart Scan and predicate pushdown
3. Learn ASM disk group management
4. Practice backup and recovery procedures
5. Explore advanced features (IORM, Smart Flash Cache)

**📧 Troubleshooting Contacts:**
- Oracle Support: Use MOS (My Oracle Support)
- Community: Oracle-L mailing list, Stack Overflow
- Documentation: Oracle Database Administrator's Guide

---

**Chúc mừng! Bạn đã hoàn thành việc xây dựng môi trường Exadata Simulation 5-Cell hoàn chỉnh với Oracle Storage Server Software 12.1.1.1.0. Môi trường này sẽ cung cấp nền tảng mạnh mẽ để học tập và thực hành các tính năng tiên tiến của Oracle Exadata.**