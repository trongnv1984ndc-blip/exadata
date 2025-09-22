# Tài liệu tham khảo các câu lệnh Oracle Exadata Storage Server

## Tổng quan

Oracle đã thiết kế các command-line utilities chuyên biệt cho việc quản lý Oracle Exadata Storage Server. Đây là những công cụ mạnh mẽ được tối ưu hóa để quản lý intelligent storage và thực hiện SQL offload processing.

---

## 1. CellCLI - Cell Command Line Interface

### 1.1 Giới thiệu
**CellCLI** là command-line interface chính để quản lý từng Exadata Storage Cell riêng lẻ.

**Cách truy cập**:
```bash
# Đăng nhập vào storage cell
ssh celladmin@cell_server_ip
# hoặc
ssh cellmonitor@cell_server_ip  # read-only
ssh root@cell_server_ip

# Khởi động CellCLI
[celladmin@cell01 ~]$ cellcli
CellCLI: Release 11.2.x.x.x - Production on Date
Copyright (c) 2007, 2009, Oracle. All rights reserved.
Cell Efficiency Ratio: xxx

CellCLI>
```

### 1.2 Cấu trúc lệnh cơ bản
```
CellCLI> <COMMAND> <OBJECT_TYPE> [object_name] [options]
```

**Các lệnh chính**:
- **LIST**: Hiển thị thông tin objects
- **CREATE**: Tạo mới objects
- **ALTER**: Thay đổi cấu hình objects
- **DROP**: Xóa objects
- **DESCRIBE**: Hiển thị các attributes có sẵn
- **HELP**: Trợ giúp

---

## 2. LIST Commands - Lệnh hiển thị thông tin

### 2.1 LIST CELL - Thông tin Cell

```bash
# Hiển thị thông tin cơ bản về cell
CellCLI> list cell
cell01                  online

# Hiển thị thông tin chi tiết
CellCLI> list cell detail

# Ví dụ output:
name:                    cell01
bmcType:                 IPMI
cellVersion:             OSS_11.2.3.2.1_LINUX.X64_111208
cpuCount:                24
fanCount:                12/12
fanStatus:               normal
cellsrvStatus:           running
msStatus:                running
rsStatus:                running
status:                  online
temperatureReading:      24.0
temperatureStatus:       normal
upTime:                  32 days, 4:44
```

### 2.2 LIST PHYSICALDISK - Physical Disks

```bash
# Liệt kê tất cả physical disks
CellCLI> list physicaldisk
20:0            K68DWJ          normal
20:1            K7YXUJ          normal
FLASH_1_0       1030M03RK1      normal
FLASH_1_1       1030M03RJN      normal

# Hiển thị chi tiết một disk cụ thể
CellCLI> list physicaldisk 20:0 detail

# Chỉ hiển thị flash disks
CellCLI> list physicaldisk where diskType='FlashDisk'

# Hiển thị specific attributes
CellCLI> list physicaldisk attributes name,diskType,status,makeModel

# Kiểm tra lỗi disk
CellCLI> list physicaldisk attributes name,errHardReadCount,errHardWriteCount
```

### 2.3 LIST CELLDISK - Cell Disks

```bash
# Liệt kê cell disks
CellCLI> list celldisk
CD_00_cell01            normal
CD_01_cell01            normal
FD_00_cell01            normal

# Chi tiết cell disk
CellCLI> list celldisk CD_00_cell01 detail

# Chỉ hard disk cell disks
CellCLI> list celldisk where diskType='HardDisk'

# Chỉ flash cell disks
CellCLI> list celldisk where diskType='FlashDisk'

# Kiểm tra free space
CellCLI> list celldisk attributes name,freeSpace,size,status
```

### 2.4 LIST GRIDDISK - Grid Disks

```bash
# Liệt kê grid disks
CellCLI> list griddisk
DATA_CD_00_cell01       active
RECO_CD_01_cell01       active

# Chi tiết grid disk
CellCLI> list griddisk DATA_CD_00_cell01 detail

# Lọc theo size
CellCLI> list griddisk attributes name,cellDisk,status where size=476.546875G

# Kiểm tra inactive grid disks
CellCLI> list griddisk where status='inactive'

# Grid disks theo disk type
CellCLI> list griddisk attributes name,cellDisk,diskType,status
```

### 2.5 LIST LUN - Logical Unit Numbers

```bash
# Liệt kê LUNs
CellCLI> list lun
0_0             0_0             normal
0_1             0_1             normal

# Chi tiết LUN
CellCLI> list lun 0_0 detail

# LUN attributes
CellCLI> list lun attributes name,lunSize,raidLevel,status
```

---

## 3. Monitoring Commands - Lệnh giám sát

### 3.1 LIST METRICCURRENT - Metrics hiện tại

```bash
# Tất cả metrics hiện tại
CellCLI> list metriccurrent

# Metrics cho specific object type
CellCLI> list metriccurrent where objectType='GRIDDISK'

# Flash cache metrics
CellCLI> list metriccurrent where objectType='FLASHCACHE'

# Smart scan metrics
CellCLI> list metriccurrent where name like 'N_SMRT_IO_*'

# Database I/O metrics
CellCLI> list metriccurrent where objectType='DATABASE'

# Specific metric với filter
CellCLI> list metriccurrent where name='FC_BY_USED' and metricValue > 0
```

### 3.2 LIST METRICHISTORY - Lịch sử metrics

```bash
# Metric history
CellCLI> list metrichistory

# Flash cache history trong 24h qua
CellCLI> list metrichistory where objectType='FLASHCACHE' and collectionTime > '2023-01-01T00:00:00'

# I/O performance history
CellCLI> list metrichistory where name like 'GD_IO_*' and collectionTime > '2023-01-01T00:00:00'
```

### 3.3 LIST FLASHCACHE - Flash Cache

```bash
# Flash cache information
CellCLI> list flashcache
f3201cel01_FLASHCACHE    normal

# Flash cache detail
CellCLI> list flashcache detail

# Flash cache content
CellCLI> list flashcachecontent

# Objects trong flash cache
CellCLI> list flashcachecontent where objectNumber=161441 detail

# Flash cache usage by database
CellCLI> list flashcachecontent where dbUniqueName like 'PROD*'
```

### 3.4 LIST FLASHLOG - Smart Flash Logging

```bash
# Flash log information
CellCLI> list flashlog

# Flash log detail
CellCLI> list flashlog detail

# Flash log metrics
CellCLI> list metriccurrent where objectType='FLASHLOG'
```

---

## 4. CREATE Commands - Lệnh tạo mới

### 4.1 CREATE GRIDDISK

```bash
# Tạo grid disk từ cell disk cụ thể
CellCLI> create griddisk DATA_CD_00_cell01 celldisk=CD_00_cell01

# Tạo với size cụ thể
CellCLI> create griddisk DATA_CD_00_cell01 celldisk=CD_00_cell01 size=100G

# Tạo grid disks cho tất cả hard disks với prefix
CellCLI> create griddisk all harddisk prefix=DATA

# Tạo grid disks cho flash disks
CellCLI> create griddisk all flashdisk prefix=FLASH

# Tạo với size và prefix
CellCLI> create griddisk all harddisk prefix='RECO', size='150G'
```

### 4.2 CREATE FLASHCACHE

```bash
# Tạo flash cache từ flash cell disks
CellCLI> create flashcache celldisk='FD_00_cell01'

# Tạo từ multiple flash disks
CellCLI> create flashcache celldisk='FD_00_cell01,FD_01_cell01,FD_02_cell01'

# Tạo flash cache với all flash disks
CellCLI> create flashcache all
```

### 4.3 CREATE THRESHOLD

```bash
# Tạo threshold cho monitoring
CellCLI> create threshold cd_io_errs_min.prodb comparison=">", critical=10

# Tạo warning threshold
CellCLI> create threshold CD_IO_ERRS_MIN warning=1, comparison='>=', occurrences=1
```

---

## 5. ALTER Commands - Lệnh thay đổi

### 5.1 ALTER CELL - Quản lý Cell Services

```bash
# Restart tất cả services
CellCLI> alter cell restart services all

# Restart service cụ thể
CellCLI> alter cell restart services cellsrv
CellCLI> alter cell restart services ms
CellCLI> alter cell restart services rs

# Stop services
CellCLI> alter cell shutdown services all
CellCLI> alter cell shutdown services cellsrv

# Start services
CellCLI> alter cell startup services all
CellCLI> alter cell startup services cellsrv

# Bật/tắt LED
CellCLI> alter cell led on
CellCLI> alter cell led off

# Restart BMC
CellCLI> alter cell restart bmc
```

### 5.2 ALTER PHYSICALDISK

```bash
# Bật service LED cho disk cụ thể
CellCLI> alter physicaldisk 20:2,20:3 serviceled on
CellCLI> alter physicaldisk 20:6,20:9 serviceled off

# Bật service LED cho tất cả hard disks
CellCLI> alter physicaldisk harddisk serviceled on

# Bật cho tất cả disks
CellCLI> alter physicaldisk all serviceled on
```

### 5.3 ALTER GRIDDISK

```bash
# Thêm comment
CellCLI> alter griddisk RECO_CD_10_cell01 comment='Used for Recovery'

# Inactive grid disk
CellCLI> alter griddisk RECO_CD_11_cell01 inactive

# Inactive với force (nguy hiểm!)
CellCLI> alter griddisk RECO_CD_08_cell01 inactive force

# Inactive mà không chờ
CellCLI> alter griddisk RECO_CD_11_cell01 inactive nowait

# Active grid disk
CellCLI> alter griddisk RECO_CD_11_cell01 active

# Inactive tất cả
CellCLI> alter griddisk all inactive

# Active tất cả
CellCLI> alter griddisk all active
```

### 5.4 ALTER IORMPLAN - I/O Resource Management

```bash
# Activate IORM plan
CellCLI> alter iormplan active

# Set objective
CellCLI> alter iormplan objective='balanced'
CellCLI> alter iormplan objective='low_latency'
CellCLI> alter iormplan objective='high_throughput'

# Database-specific plan
CellCLI> alter iormplan dbplan=((name='PROD',share=8),(name='TEST',share=2))

# Category plan
CellCLI> alter iormplan catplan=((name='dss',share=75),(name='oltp',share=25))

# Flash cache management
CellCLI> alter iormplan flashcache=off
CellCLI> alter iormplan flashlog=off

# Disable IORM
CellCLI> alter iormplan objective=
```

---

## 6. DROP Commands - Lệnh xóa

```bash
# Xóa grid disk
CellCLI> drop griddisk DATA_CD_00_cell01

# Xóa cell disk (cẩn thận!)
CellCLI> drop celldisk CD_00_cell01

# Xóa flash cache
CellCLI> drop flashcache

# Reset cell (factory reset)
CellCLI> drop cell
```

---

## 7. DESCRIBE Commands - Mô tả attributes

```bash
# Xem available attributes
CellCLI> describe cell
CellCLI> describe physicaldisk
CellCLI> describe celldisk
CellCLI> describe griddisk
CellCLI> describe flashcache
```

---

## 8. DCLI - Distributed Command Line Interface

### 8.1 Giới thiệu DCLI
**DCLI** cho phép thực hiện lệnh trên multiple storage cells đồng thời.

### 8.2 Setup DCLI

```bash
# Tạo file chứa list các cells
[celladmin@dbserver ~]$ vi cell_group
cell01
cell02
cell03

# Setup SSH key-based authentication
[celladmin@dbserver ~]$ ssh-keygen -t rsa
[celladmin@dbserver ~]$ dcli -g cell_group -k
```

### 8.3 DCLI Commands

```bash
# Cú pháp cơ bản
dcli [options] [command]

# Các options quan trọng:
# -g <group_file>  : file chứa list servers
# -c <cell_list>   : comma-separated list of cells
# -l <username>    : login username (default: celladmin)
# -k               : setup SSH keys
# -v               : verbose mode
# -n               : abbreviate non-error output
# -r <pattern>     : remove lines matching pattern
# -f <file>        : copy file to remote servers
# -d <directory>   : destination directory for file copy
# --serial         : execute serially instead of parallel
```

### 8.4 DCLI Examples

```bash
# Kiểm tra status tất cả cells
dcli -g cell_group cellcli -e list cell

# Tạo grid disks trên tất cả cells
dcli -g cell_group cellcli -e "create griddisk all harddisk prefix=DATA"

# Restart services trên tất cả cells
dcli -l root -g cell_group cellcli -e "alter cell restart services all"

# Kiểm tra physical disks
dcli -g cell_group cellcli -e "list physicaldisk attributes name,status"

# Flash cache usage
dcli -g cell_group cellcli -e "list metriccurrent where name='FC_BY_USED'"

# Copy file tới multiple cells
dcli -g cell_group -f /tmp/script.sh -d /tmp

# Execute với specific cells
dcli -c cell01,cell02 cellcli -e "list griddisk"

# Serial execution
dcli -g cell_group --serial cellcli -e "list cell detail"

# Filter output
dcli -g cell_group -r "normal" cellcli -e "list physicaldisk"

# Verbose mode
dcli -g cell_group -v cellcli -e "list cell"
```

---

## 9. ExaCLI - Exadata CLI (Cloud/Remote)

### 9.1 Giới thiệu ExaCLI
**ExaCLI** cho phép remote management của Exadata Storage Servers, đặc biệt trong cloud environments.

### 9.2 ExaCLI Usage

```bash
# Connect to storage server
exacli -l <username> -c <storage_server_ip>

# Interactive session
exacli cloud_user_cluster@192.168.136.7

# Single command execution
exacli -l cloud_user_cluster -c 192.168.136.7 "list cell"

# Available objects trong ExaCLI (subset của CellCLI):
# CELL, CELLDISK, DATABASE, FLASHCACHE, FLASHCACHECONTENT,
# GRIDDISK, PHYSICALDISK, PLUGGABLEDATABASE, etc.
```

---

## 10. Specialized Commands

### 10.1 CALIBRATE - Performance Testing

```bash
# Test storage performance
CellCLI> calibrate force
```

### 10.2 Alert Management

```bash
# List alert definitions
CellCLI> list alertdefinition

# List alert history
CellCLI> list alerthistory

# List specific alerts
CellCLI> list alerthistory where examinedBy is null
```

### 10.3 Quarantine Management

```bash
# List quarantined SQL
CellCLI> list quarantine

# Create quarantine rule
CellCLI> create quarantine 'select * from large_table'
```

---

## 11. Monitoring và Diagnostics Commands

### 11.1 Active Requests

```bash
# Show active requests
CellCLI> list activerequest

# Active requests với detail
CellCLI> list activerequest detail
```

### 11.2 InfiniBand Ports

```bash
# List IB ports
CellCLI> list ibport

# IB port detail
CellCLI> list ibport ib0 detail

# Reset IB counters
CellCLI> alter ibport ib0 reset counters
```

### 11.3 System Information

```bash
# List metric definitions
CellCLI> list metricdefinition

# Available metrics
CellCLI> list metricdefinition where objectType='GRIDDISK'

# Key management
CellCLI> list key
```

---

## 12. Best Practices và Security

### 12.1 User Accounts
- **celladmin**: Full administrative access
- **cellmonitor**: Read-only access (recommended for monitoring)
- **root**: Full system access (use sparingly)

### 12.2 Safety Guidelines

```bash
# Always check before making changes
CellCLI> list griddisk where status != 'active'

# Use describe to understand attributes
CellCLI> describe griddisk

# Backup important configurations
dcli -g cell_group cellcli -e "list cell detail" > cell_config_backup.txt

# Never use 'force' option unless absolutely necessary
# CellCLI> alter griddisk ... force  # DANGEROUS!
```

### 12.3 Common Troubleshooting Commands

```bash
# Check cell health
CellCLI> list cell detail

# Service status
CellCLI> list cell attributes cellsrvStatus,msStatus,rsStatus

# Disk errors
CellCLI> list physicaldisk attributes name,errHardReadCount,errHardWriteCount where errHardReadCount > 0

# Flash cache issues
CellCLI> list metriccurrent where objectType='FLASHCACHE' and name like '*ERR*'

# Grid disk problems
CellCLI> list griddisk where status != 'active'

# IORM status
CellCLI> list iormplan detail
```

---

## 13. Quick Reference Commands

### 13.1 Essential Monitoring Commands
```bash
# Cell overview
cellcli -e list cell detail

# Storage overview
cellcli -e list physicaldisk
cellcli -e list celldisk
cellcli -e list griddisk

# Performance monitoring
cellcli -e list metriccurrent where name like 'FC_*'
cellcli -e list metriccurrent where objectType='DATABASE'

# Flash cache usage
cellcli -e list flashcache detail
cellcli -e list flashcachecontent
```

### 13.2 Essential Management Commands
```bash
# Service management
cellcli -e alter cell restart services all

# IORM management
cellcli -e list iormplan detail
cellcli -e alter iormplan active

# Grid disk management
cellcli -e create griddisk all harddisk prefix=DATA
cellcli -e alter griddisk all inactive
```

---

**Lưu ý quan trọng**: 
- Luôn backup cấu hình trước khi thay đổi
- Test commands trên development environment trước
- Sử dụng Oracle Support khi gặp vấn đề phức tạp
- Các lệnh này yêu cầu privilege phù hợp và có thể ảnh hưởng tới production system

Tài liệu này cung cấp foundation commands cho việc quản lý Oracle Exadata Storage Server. Để biết thêm chi tiết về từng lệnh cụ thể, sử dụng `HELP <command>` trong CellCLI.