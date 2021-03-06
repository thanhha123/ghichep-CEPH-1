## Hướng dẫn cài đặt CEPH sử dụng `ceph-deploy`

### Mục tiêu LAB
- Mô hình này sử dụng 2 server, trong đó:
  - Host Ceph1 cài đặt ceph-deploy, ceph-mon, ceph-osd, ceph-mgr
  - Host Ceph2 cài đặt ceph-osd, ceph-mgr
- LAB này chỉ phù hợp với việc nghiên cứu các tính năng và demo thử nghiệm, không áp dụng được trong thực tế.

## Mô hình 
- Sử dụng mô hình
![img](../images/ceph_luminous/luminous-topo.jpg)

## IP Planning
- Phân hoạch IP cho các máy chủ trong mô hình trên
![img](../images/ceph_luminous/luminous-ip-cent72.jpg)

## Chuẩn bị và môi trường LAB
 
- OS
  - CentOS 7.4 - 64 bit
  - 05: HDD, trong đó:
    - `vda`: sử dụng để cài OS
    - `vdb`: sử dụng làm OSD (nơi chứa dữ liệu của client)
  - 02 NICs: 
    - `eth0`: dùng để ssh và tải gói cài đặt, sử dụng dải 172.16.69.0/24
    - `eth1`: dùng để replicate cho CEPH sử dụng, dải 10.10.20.0/24
    - `eth2`: dùng để client (các máy chủ trong OpenStack) sử dụng, dải 10.10.10.0/24
  
- CEPH Luminous

## Thực hiện trên từng host 

- Thực hiện bằng quyền root
  ```sh
  su -
  ```
  
- Đặt hostname
  ```sh
  hostnamectl set-hostname ceph1
  ```

- Sửa file host 
  ```sh
  echo "10.10.10.75 ceph1" >> /etc/hosts
  echo "10.10.10.76 ceph2" >> /etc/hosts
  ```

- Update OS 
  ```sh
  sudo yum update -y
  ```

- Cấu hình các thành phần mạng cơ bản
  ```sh
  sudo systemctl disable firewalld
  sudo systemctl stop firewalld
  sudo systemctl disable NetworkManager
  sudo systemctl stop NetworkManager
  sudo systemctl enable network
  sudo systemctl start network
  ```
- Vô hiệu hóa Selinux
  ```sh
  sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
  ```
- Khởi động lại máy
  ```sh
  init 6
  ```
- Khai báo repos cho CEPH
  ```sh
  sudo yum install -y yum-utils
  sudo yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ 
  sudo yum install --nogpgcheck -y epel-release 
  sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 
  sudo rm /etc/yum.repos.d/dl.fedoraproject.org*
  ```
- Update sau khi khai báo repo
  ```sh
  yum -y update
  ```

- Tạo user `ceph-deploy` để sử dụng cho việc cài đặt cho CEPH.
  ```sh
  sudo useradd -m -s /bin/bash ceph-deploy
  ```

- Đặt mật mẩu cho user `ceph-deploy`  
  ```sh
  sudo passwd ceph-deploy
  ```

- Phân quyền cho user `ceph-deploy`
  ```sh
  echo "ceph-deploy ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ceph-deploy
  sudo chmod 0440 /etc/sudoers.d/ceph-deploy
  ```

  
## Cài đặt CEPH
Thực hiện trên host ceph1 

- Cài đặt công cụ `ceph-deploy`
  ```sh
  sudo yum install -y ceph-deploy
  ```
- Chuyển sang tài khoản `ceph-deploy` để thực hiện cài đặt
  ```sh
  sudo su - ceph-deploy
  ```
- Tạo ssh key cho user ceph-deploy`. Nhấn enter đối với các bước được hỏi trên màn hình.
  ```sh
  ssh-keygen
  ```
- Copy ssh key sang các máy ceph
  ```sh
  ssh-copy-id ceph-deploy@ceph1
  ssh-copy-id ceph-deploy@ceph2
  ```
  - Nhập `Yes` và mật khẩu của user `ceph-deploy` đã tạo ở trước, kết quả như bên dưới
    ```sh
    ceph-deploy@ceph1:~$ ssh-copy-id ceph-deploy@ceph1
    The authenticity of host 'ceph1 (172.16.69.247)' can't be established.
    ECDSA key fingerprint is f2:38:1e:50:44:94:6f:0a:32:a3:23:63:90:7b:53:27.
    Are you sure you want to continue connecting (yes/no)? yes
    /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
    /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
    ceph-deploy@ceph1's password:
    Number of key(s) added: 1

    Now try logging into the machine, with:   "ssh 'ceph-deploy@ceph1'"
    and check to make sure that only the key(s) you wanted were added.
    ```
  
- Tạo các thư mục để công cụ `ceph-deploy` sử dụng để cài đặt CEPH
  ```sh
  cd ~
  mkdir my-cluster
  cd my-cluster
  ```

- Thiết lập các file cấu hình cho CEPH.
  ```sh
  ceph-deploy new ceph1
  ```

- Sau khi thực hiện lệnh trên xong, sẽ thu được 03 file ở dưới (sử dụng lệnh `ll -alh` để xem). Trong đó cần cập nhật file `ceph.conf` để cài đặt CEPH được hoàn chỉnh.
  ```sh
  [ceph-deploy@ceph1 cluster-ceph]$ ls -al
  total 20
  drwxrwxr-x. 2 ceph-deploy ceph-deploy 4096 Sep 15 15:35 .
  drwx------. 4 ceph-deploy ceph-deploy 4096 Sep 15 15:35 ..
  -rw-rw-r--. 1 ceph-deploy ceph-deploy  194 Sep 15 15:35 ceph.conf
  -rw-rw-r--. 1 ceph-deploy ceph-deploy 3028 Sep 15 15:35 ceph-deploy-ceph.log
  -rw-------. 1 ceph-deploy ceph-deploy   73 Sep 15 15:35 ceph.mon.keyring
  ```

- Sửa file ceph.conf như sau:
  ```sh
  [global]
  fsid = 36bb6bc0-5f0d-4418-b403-2d4ca22779a2
  mon_initial_members = ceph1
  mon host = 10.10.10.75
  auth_cluster_required = cephx
  auth_service_required = cephx
  auth_client_required = cephx

  osd pool default size = 3
  osd pool default min size = 1
  osd crush chooseleaf type = 0
  public network = 10.10.10.0/24
  cluster network = 10.10.20.0/24
  bluestore block db size = 5737418240
  bluestore block wal size = 2737418240
  osd objectstore = bluestore
  mon_allow_pool_delete = true
  rbd_cache = false
  osd pool default pg num = 128
  osd pool default pgp num = 128

  [mon.ceph1]
  host = ceph1
  mon addr = 10.10.10.70

  [osd]
  osd crush update on start = false
  bluestore = true
  ```
  
- Cài đặt CEPH, thay `ceph1` bằng tên hostname của máy bạn nếu có thay đổi.
  ```sh
  ceph-deploy install --release luminous ceph1 ceph2
  ```
- Chuyển cấu hình ceph.conf sang các host ceph khác
  ```sh
  ceph-deploy --overwrite-conf config push ceph2
  ```

- Cấu hình `MON` (một thành phần của CEPH)
  ```sh
  ceph-deploy mon create-initial
  ```

- Sau khi thực hiện lệnh để cấu hình `MON` xong, sẽ sinh thêm ra 03 file : `ceph.bootstrap-mds.keyring`, `ceph.bootstrap-osd.keyring` và `ceph.bootstrap-rgw.keyring`. Quan sát bằng lệnh `ll -alh`

  ```sh
  ceph-deploy@ceph1:~/my-cluster$ ls -alh
  total 576K
  drwxrwxr-x 2 ceph-deploy ceph-deploy 4.0K Sep 11 21:12 .
  drwxr-xr-x 6 ceph-deploy ceph-deploy 4.0K Sep 11 21:12 ..
  -rw------- 1 ceph-deploy ceph-deploy   71 Sep  7 18:40 ceph.bootstrap-mds.keyring
  -rw------- 1 ceph-deploy ceph-deploy   71 Sep  7 18:40 ceph.bootstrap-mgr.keyring
  -rw------- 1 ceph-deploy ceph-deploy   71 Sep  7 18:40 ceph.bootstrap-osd.keyring
  -rw------- 1 ceph-deploy ceph-deploy   71 Sep  7 18:40 ceph.bootstrap-rgw.keyring
  -rw------- 1 ceph-deploy ceph-deploy  71 Sep  12 17:20 ceph.client.admin.keyring
  -rw-rw-r-- 1 ceph-deploy ceph-deploy  849 Sep 11 20:58 ceph.conf
  -rw-rw-r-- 1 ceph-deploy ceph-deploy 530K Sep 11 20:58 ceph-deploy-ceph.log
  -rw------- 1 ceph-deploy ceph-deploy   73 Sep  7 18:34 ceph.mon.keyring
  -rw-r--r-- 1 root        root        1.7K Oct 15  2015 release.asc
  ```
- Tạo user admin để quản trị ceph cluster
  ```sh  
  ceph-deploy admin ceph1
  sudo chmod +r /etc/ceph/ceph.client.admin.keyring
  ```

- Copy `ceph.bootstrap-osd.keyring` sang tất cả các host ceph
  ```sh
  scp ceph.bootstrap-osd.keyring ceph1:/var/lib/ceph/bootstrap-osd/ceph.keyring
  scp ceph.bootstrap-osd.keyring ceph2:/var/lib/ceph/bootstrap-osd/ceph.keyring
  ```

Thực hiện trên từng host
- Tạo các OSD cho CEPH, thay `ceph1` bằng tên hostname của máy bạn 
  ```sh
  ceph-deploy disk zap ceph1:/dev/vdb
  ceph-deploy osd prepare ceph1:vdb
  ```
  Sử dụng ceph-disk tại host chứa osd sẽ có thêm các tùy chọn vị trí đặt block.db, block.wal
  ```sh
  ceph-disk prepare  --bluestore --block.db /dev/vdb  --block.wal /dev/vdb /dev/vdb
  ```
  
- Kiểm tra các phân vùng được tạo ra bằng lệnh `sudo lsblk` (nhớ phải có lệnh sudo vì đang dùng user `ceph-deploy`)
```sh
ceph-deploy@ceph1:~/my-cluster$ sudo lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0     11:0    1 1024M  0 rom  
vda    253:0    0   10G  0 disk 
└─vda1 253:1    0   10G  0 part /
vdb    253:16   0   40G  0 disk 
├─vdb1 253:17   0  100M  0 part /var/lib/ceph/osd/ceph-1
├─vdb2 253:18   0   32G  0 part 
├─vdb3 253:19   0  5.4G  0 part 
└─vdb4 253:20   0  2.6G  0 part 
```

Thực hiện trên ceph1
- Copy key về thư mục `/etc/ceph`
  ```sh
  sudo cp ceph.client.admin.keyring /etc/ceph
  ```

- Phân quyền cho file `/etc/ceph/ceph.client.admin.keyring`
  ```sh
  sudo chmod +r /etc/ceph/ceph.client.admin.keyring
  ```

- Kiểm tra trạng thái của CEPH sau khi cài
  ```sh
  ceph -s
  ```

- Kết quả của lệnh trên như sau: 
  ```sh
  ceph-deploy@ceph1:~/my-cluster$ ceph -s
  cluster:
    id:     36bb6bc0-5f0d-4418-b403-2d4ca22779a2
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum ceph1,ceph2,ceph3
    mgr: ceph1(active), standbys: ceph2
    osd: 3 osds: 3 up, 3 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 bytes
    usage:   3180 MB used, 116 GB / 119 GB avail
    pgs:     
  ```

- Nếu có dòng `health HEALTH_OK` thì việc cài đặt đã ok.

- Khi tạo pool, nếu gặp cảnh báo sau khi dùng lênh `ceph -s`
```sh
application not enabled on <N> pool(s)
```
Ở bản Ceph Luminous, các pool được tạo ra phải chỉ định sử dụng cho gateway nào, VD ta chỉ định pool block sử dụng gateway rbd như sau
```sh
ceph osd pool application enable block rbd
```
Lúc này cảnh báo trên sẽ không còn.

- Để add thêm cấu hình trên ceph.conf, cần restart lại mon service:
```sh
systemctl restart ceph-mon.target
```
Để kiểm tra cấu hình trên 1 daemon, vd ở đây là mon daemon, dùng lệnh
```sh
ceph daemon /var/run/ceph/ceph-mon.*.asok config show | grep mon_osd_report_timeout
```

Tham khảo:

[1] - https://chenmiao2016.github.io/2017/07/29/ceph-luminous%E6%90%AD%E5%BB%BA/

[2] - https://www.zybuluo.com/dyj2017/note/924784








