# Cài đặt mới Oracle Exadata Storage Server 20.1.0.0.0 từ đầu

## Mục lục
1. [Chuẩn bị phần cứng và môi trường](#i-chuẩn-bị-phần-cứng-và-môi-trường)
2. [Cài đặt Operating System cơ bản](#ii-cài-đặt-operating-system-cơ-bản)
3. [Chuẩn bị hệ thống cho Exadata](#iii-chuẩn-bị-hệ-thống-cho-exadata)
4. [Cài đặt Exadata Storage Server Software](#iv-cài-đặt-exadata-storage-server-software)
5. [Cấu hình InfiniBand Network](#v-cấu-hình-infiniband-network)
6. [Khởi động và cấu hình Cell Services](#vi-khởi-động-và-cấu-hình-cell-services)
7. [Cấu hình Management Network](#vii-cấu-hình-management-network)
8. [Security và Monitoring](#viii-security-và-monitoring)
9. [Calibration và Testing](#ix-calibration-và-testing)
10. [Final Cleanup](#x-final-cleanup)
11. [Post-Installation Verification](#xi-post-installation-verification)
12. [Troubleshooting](#xii-troubleshooting)

---

## I. Chuẩn bị phần cứng và môi trường

### 1. Yêu cầu phần cứng tối thiểu
- **Server**: Oracle Exadata Storage Server (X8M, X9M hoặc tương thích)
- **RAM**: Tối thiểu 64GB
- **CPU**: Intel Xeon (multi-core)
- **Storage**: SSD và HDD drives
- **Network**: InfiniBand + Ethernet interfaces

### 2. Chuẩn bị môi trường cài đặt
```bash
# Tạo bootable USB hoặc sử dụng IPMI virtual media
# Download Oracle Linux 7.9 hoặc 8.x minimal ISO
```

### 3. Files cần thiết
- `cell_20.1.0.0.0_LINUX.X64_200616-1.x86_64.iso`
- Oracle Linux 7.9 hoặc 8.x ISO
- License keys (nếu có)

---

## II. Cài đặt Operating System cơ bản

### 1. Boot từ Oracle Linux ISO
- Khởi động server từ Oracle Linux installation media
- Chọn "Install Oracle Linux" hoặc "Install Red Hat Enterprise Linux"

### 2. Cấu hình cài đặt OS

**Partition layout khuyến nghị:**
```
/boot     - 1GB   (ext4)
/         - 50GB  (ext4) 
/opt      - 100GB (ext4)
/u01      - 50GB  (ext4)
swap      - 16GB
/tmp      - 20GB  (ext4)
```

### 3. Network configuration cơ bản
```bash
# Cấu hình management network tạm thời
# Ví dụ: eth0 = 192.168.1.100/24
```

---

## III. Chuẩn bị hệ thống cho Exadata

### 1. Update và cài đặt packages cần thiết
```bash
# Update system
yum update -y

# Cài đặt required packages
yum install -y wget curl unzip tar
yum groupinstall -y "Development Tools"
yum install -y kernel-devel kernel-headers
yum install -y infiniband-utils rdma-core

# Disable SELinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
setenforce 0

# Disable firewall
systemctl disable firewalld
systemctl stop firewalld
```

### 2. Cấu hình kernel parameters
```bash
# Chỉnh sửa /etc/sysctl.conf
cat >> /etc/sysctl.conf << 'EOF'
# Exadata Storage Server parameters
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
fs.aio-max-nr = 1048576
fs.file-max = 6815744
EOF

# Apply settings
sysctl -p
```

### 3. Cấu hình users và groups
```bash
# Tạo oracle user và group
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 oper

useradd -u 54321 -g oinstall -G dba,oper oracle
echo "oracle:oracle123" | chpasswd

# Tạo thư mục cần thiết
mkdir -p /opt/oracle
mkdir -p /u01/app/oracle
chown -R oracle:oinstall /opt/oracle /u01/app/oracle
chmod -R 755 /opt/oracle /u01/app/oracle
```

---

## IV. Cài đặt Exadata Storage Server Software

### 1. Mount ISO file
```bash
# Copy ISO file vào server (qua SCP, USB, hoặc virtual media)
mkdir -p /mnt/exadata_iso
mount -o loop cell_20.1.0.0.0_LINUX.X64_200616-1.x86_64.iso /mnt/exadata_iso

# Kiểm tra nội dung
ls -la /mnt/exadata_iso/
```

### 2. Chạy imagemaker để cài đặt
```bash
# Copy installation files
mkdir -p /tmp/exadata_install
cp -r /mnt/exadata_iso/* /tmp/exadata_install/
cd /tmp/exadata_install

# Chạy imagemaker cho fresh installation
./imagemaker --install --target /opt/oracle/cell

# Hoặc sử dụng với response file
./imagemaker --install --target /opt/oracle/cell --silent --responsefile install.rsp
```

### 3. Alternative: Manual RPM installation
```bash
# Nếu imagemaker không hoạt động, cài manual
cd /mnt/exadata_iso

# Cài đặt kernel packages trước
rpm -ivh kernel/*.rpm

# Cài đặt Exadata packages
rpm -ivh exadata-*.rpm
rpm -ivh cell-*.rpm

# Verify installation
rpm -qa | grep -i exadata
rpm -qa | grep -i cell
```

---

## V. Cấu hình InfiniBand Network

### 1. Load InfiniBand drivers
```bash
# Load IB modules
modprobe ib_core
modprobe ib_mad
modprobe ib_sa
modprobe ib_cm
modprobe ib_ipoib

# Verify IB interfaces
ibstat
ibstatus
```

### 2. Cấu hình IB interfaces
```bash
# Cấu hình /etc/sysconfig/network-scripts/ifcfg-ib0
cat > /etc/sysconfig/network-scripts/ifcfg-ib0 << 'EOF'
DEVICE=ib0
BOOTPROTO=static
IPADDR=10.0.0.10
NETMASK=255.255.255.0
ONBOOT=yes
TYPE=InfiniBand
EOF

# Cấu hình /etc/sysconfig/network-scripts/ifcfg-ib1
cat > /etc/sysconfig/network-scripts/ifcfg-ib1 << 'EOF'
DEVICE=ib1
BOOTPROTO=static
IPADDR=10.0.1.10
NETMASK=255.255.255.0
ONBOOT=yes
TYPE=InfiniBand
EOF

# Restart network
systemctl restart network
```

---

## VI. Khởi động và cấu hình Cell Services

### 1. Start cell services lần đầu
```bash
# Start celld daemon
systemctl start celld
systemctl enable celld

# Verify celld is running
systemctl status celld
ps -ef | grep celld
```

### 2. Initial cell configuration
```bash
# Connect to cellcli
/opt/oracle/cell/cellsrv/bin/cellcli

# Trong cellcli, thực hiện:
CellCLI> ALTER CELL startup services all
CellCLI> LIST CELL

# Set cell name
CellCLI> ALTER CELL name='exadata-cell01'

# Configure interconnect
CellCLI> ALTER CELL interconnect1='10.0.0.10/24'
CellCLI> ALTER CELL interconnect2='10.0.1.10/24'
```

### 3. Discover và configure storage
```bash
# Trong cellcli:
CellCLI> LIST PHYSICALDISK

# Create cell disks for all flash cache drives
CellCLI> CREATE CELLDISK ALL FLASHDISK

# Create cell disks for hard drives  
CellCLI> CREATE CELLDISK ALL HARDDISK

# Verify cell disks
CellCLI> LIST CELLDISK

# Create grid disks
CellCLI> CREATE GRIDDISK ALL HARDDISK prefix='DATA'
CellCLI> CREATE GRIDDISK ALL FLASHDISK prefix='RECO'

# List grid disks
CellCLI> LIST GRIDDISK
```

---

## VII. Cấu hình Management Network

### 1. Cấu hình Ethernet management
```bash
# Cấu hình eth0 cho management
cat > /etc/sysconfig/network-scripts/ifcfg-eth0 << 'EOF'
DEVICE=eth0
BOOTPROTO=static
IPADDR=192.168.1.100
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=8.8.8.8
ONBOOT=yes
EOF

# Restart network
systemctl restart network
```

### 2. Cấu hình trong cellcli
```bash
CellCLI> ALTER CELL managementNetwork='192.168.1.100/24'
CellCLI> ALTER CELL defaultGateway='192.168.1.1'
CellCLI> ALTER CELL dnsServer1='8.8.8.8'
```

---

## VIII. Security và Monitoring

### 1. Configure HTTPS
```bash
CellCLI> ALTER CELL httpsAccess=TRUE
CellCLI> ALTER CELL httpPort=8888
CellCLI> ALTER CELL httpsPort=8889
```

### 2. Configure SNMP
```bash
CellCLI> ALTER CELL snmpSubscriber=(host=monitoring.domain.com,port=162,community=public)
```

### 3. Configure email alerts
```bash
CellCLI> ALTER CELL smtpServer='mail.domain.com'
CellCLI> ALTER CELL smtpFrom='exadata@domain.com'
CellCLI> ALTER CELL smtpTo='admin@domain.com'
```

---

## IX. Calibration và Testing

### 1. Run storage calibration
```bash
CellCLI> CALIBRATE
# Chờ quá trình calibration hoàn thành (có thể mất vài giờ)

CellCLI> LIST CALIBRATIONHISTORY
```

### 2. Verify installation
```bash
# Check cell status
CellCLI> LIST CELL DETAIL

# Check all components
CellCLI> LIST PHYSICALDISK
CellCLI> LIST CELLDISK  
CellCLI> LIST GRIDDISK
CellCLI> LIST ALERTHISTORY

# Check version
imageinfo
```

---

## X. Final Cleanup

```bash
# Unmount ISO
umount /mnt/exadata_iso

# Remove temp files
rm -rf /tmp/exadata_install

# Enable services for auto-start
systemctl enable celld
systemctl enable cellsrv

# Reboot để verify everything starts correctly
reboot
```

---

## XI. Post-Installation Verification

```bash
# Sau khi reboot, verify services
systemctl status celld
systemctl status cellsrv

# Connect và check cell
cellcli -e "LIST CELL"
cellcli -e "LIST GRIDDISK"
cellcli -e "LIST METRICCURRENT"
```

### Checklist sau cài đặt:
- [ ] Cell services running
- [ ] InfiniBand interfaces up
- [ ] Management network configured  
- [ ] Storage disks discovered
- [ ] Grid disks created
- [ ] Calibration completed
- [ ] Monitoring configured

---

## XII. Troubleshooting

### Lỗi thường gặp

#### 1. Mount ISO failed
```bash
# Kiểm tra file integrity
md5sum cell_20.1.0.0.0_LINUX.X64_200616-1.x86_64.iso

# Try different mount options
mount -o loop,ro cell_20.1.0.0.0_LINUX.X64_200616-1.x86_64.iso /mnt/exadata_iso
```

#### 2. Service start failed
```bash
# Check logs
tail -f /opt/oracle/cell/log/celld.log
tail -f /opt/oracle/cell/log/cellsrv.log

# Reset configuration nếu cần
cellcli -e "ALTER CELL shutdown services all"
```

#### 3. Network issues
```bash
# Verify network interfaces
ip addr show
ibstatus

# Check routing
ip route show
```

#### 4. Storage not discovered
```bash
# Check physical disks
ls -la /dev/sd*
lsblk

# Verify disk permissions
ls -la /dev/disk/by-id/
```

#### 5. Calibration failed
```bash
# Check for hardware issues
cellcli -e "LIST PHYSICALDISK ATTRIBUTES status WHERE status != normal"

# Review calibration logs
cellcli -e "LIST CALIBRATIONHISTORY DETAIL"
```

---

## Lưu ý quan trọng

⚠️ **Cảnh báo:**
- Quá trình này tạo một Exadata Storage Server hoàn toàn mới
- Calibration có thể mất 2-8 giờ tùy theo số lượng disk
- Cần coordinate với team database để connect ASM instances
- Document tất cả IP addresses và configurations
- Test connectivity với compute nodes trước khi production

📝 **Best Practices:**
- Luôn backup configuration trước khi thay đổi
- Test trên development environment trước
- Coordinate với Oracle Support nếu cần
- Thực hiện trong maintenance window
- Monitor system resources trong quá trình cài đặt

---

## Appendix

### A. Sample Response File
```ini
# install.rsp
ORACLE_BASE=/opt/oracle
ORACLE_HOME=/opt/oracle/cell
CELL_NAME=exadata-cell01
MANAGEMENT_IP=192.168.1.100
INTERCONNECT1_IP=10.0.0.10
INTERCONNECT2_IP=10.0.1.10
```

### B. Useful Commands
```bash
# Check Exadata version
imageinfo

# Cell status overview
cellcli -e "LIST CELL DETAIL"

# Storage summary
cellcli -e "LIST GRIDDISK ATTRIBUTES name,size,status"

# Network status
cellcli -e "LIST IBPORT DETAIL"

# Performance metrics
cellcli -e "LIST METRICCURRENT"
```

### C. Log Files Location
```
/opt/oracle/cell/log/celld.log
/opt/oracle/cell/log/cellsrv.log
/opt/oracle/cell/log/ms_odl.log
/var/log/messages
/var/log/dmesg
```

---

**Document Version:** 1.0  
**Last Updated:** September 2025  
**Compatibility:** Exadata Storage Server 20.1.0.0.0