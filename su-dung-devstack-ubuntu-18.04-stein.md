## Giới thiệu

Devstack giúp bạn triển khai nhanh một hệ thống Openstack bằng 1 script.

## Chuẩn bị

Bạn cần một máy chủ chạy Ubuntu 18.04 64bits.

Cấu hình phụ thuộc vào số lượng project bạn muốn cài. Trong trường hợp của tôi, cấu hình như sau:
```sh
CPU: 4 vCPU
RAM: 8096 GB
Disk: 100GB
Interface: eth0 ra internet với IP 172.16.69.225
```

## Cài đặt

Trước tiên cần chắc chắn hệ điều hành đã được cập nhật mới nhất
```sh
apt update && apt dist-upgrade -y
```

### Cấu hình IP cho netplan (nếu muốn)
Sau đó thực hiện cấu hình địa chỉ IP tĩnh cho máy chủ Ubuntu 18.04. Tôi sử dụng trình quản lý network mặc định là netplan chứ không gỡ đi để cài đặt ifupdown.
Kiểm tra file cấu hình network mặc định
```sh
ls /etc/netplan
```

sẽ thấy một file cấu hình network có tên là `50-cloud-init.yaml`. Chúng ta sẽ chỉnh cấu hình trong file này để thiết lập IP tĩnh. lưu ý là nếu có nhiều interface thì sẽ cấu hình tương tự. cần để ý việc giật cấp bằng khoảng trắng.
```sh
vim /etc/netplan/50-cloud-init.yaml
```

Điền các thông tin như sau:
```sh
addresses : [172.16.69.225/24]
gateway4: 172.16.69.1
dhcp4: no
nameservers:
  addresses: [8.8.8.8,8.8.4.4]
```

Áp dụng các cấu hình vừa thiết lập vào server bằng lệnh:
```sh
sudo netplan apply
```

### Cấu hình IP cho ifupdown (tôi xài ở đây)
Chả hiểu sao để netplan ko cài được barbican, nên quyết định xài lại ifupdown.

Cấu hình ifupdown như sau:

- cấu hình repo
```sh
echo 'Acquire::http::Proxy "http://123.30.178.220:3142";' >  /etc/apt/apt.conf
```

- Cập nhật gói:
```sh
apt update -y && apt dist-upgrade -y
```

- Cài đặt ifupdown:
```sh
apt install ifupdown -y
```

- Cấu hình IP:
```sh
cat << EOF > /etc/network/interfaces
# loopback network interface
auto lo
iface lo inet loopback

# external network interface
auto ens3
iface ens3 inet static
address 172.16.69.225
netmask 255.255.255.0
gateway 172.16.69.1
dns-nameservers 8.8.8.8 8.8.4.4

EOF
```

- Thiết lập IP vừa được cấu hình
```sh
ifdown --force ens3 lo && ifup -a
```

- Gỡ netplan:
```sh
systemctl stop networkd-dispatcher
systemctl disable networkd-dispatcher
systemctl mask networkd-dispatcher
apt-get purge nplan netplan.io -y
```

- Cập nhật repo cho máy chủ
```sh
cat  << EOF >> /etc/apt/sources.list
deb http://security.ubuntu.com/ubuntu/ bionic-security main restricted
deb http://security.ubuntu.com/ubuntu/ bionic-security universe
deb http://security.ubuntu.com/ubuntu/ bionic-security multiverse
EOF
```

Thiết lập repo `universe` để HĐH có thể tải gói về cài đặt
```sh
sudo add-apt-repository universe
sudo apt update
```

- Thiết lập lại DNS cho máy chủ do ifupdown ko quản lý DNS.
```sh
sudo apt install resolvconf -y

echo "nameserver 8.8.8.8" >> /etc/resolv.conf
echo "nameserver 8.8.4.4" >> /etc/resolv.conf
echo "nameserver 8.8.8.8" >> /etc/resolvconf/resolv.conf.d/head
echo "nameserver 8.8.4.4" >> /etc/resolvconf/resolv.conf.d/head
sudo service resolvconf restart
```

- Cập nhật HĐH khi thêm repo:
```sh
apt update -y && apt dist-upgrade -y && apt autoremove -y
```

- Khởi động lại máy chủ
```sh
init 6
```

### Chạy script cài đặt

Devstack nên được chạy với user khác root, vì thế, ta sẽ tạo một username `stack` để chạy Devstack
```sh
sudo useradd -s /bin/bash -d /opt/stack -m stack
```

User `stack` sẽ thao tác nhiều với quyền root nên ta thiết lập cấu hình để không cần nhập password root khi chạy lệnh sudo
```sh
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo su - stack
```

Bạn thay branch của devstack phù hợp với branch trong file localrc để cài đặt.

Thực hiện tải một bản devstack về
```sh
git clone https://git.openstack.org/openstack-dev/devstack --branch stable/stein
cd devstack
```

Kiểm tra nhánh của devstack
```sh
git status
```

Nếu là nhánh master thì ta chuyển sang nhánh stein như sau . Do tôi cài stable/stein nên đã chỉ định ngay khi tải về rồi nên bước này bỏ qua
```sh
git checkout stable/stein
```

Sử dụng lệnh sau để lấy tập tin cấu hình chỉ định cho việc cài đặt về máy:
```sh
wget https://raw.githubusercontent.com/TrongTan124/ghi-chep-devstack/master/Local_conf/local-stable-stein.conf -O ./local.conf
```

Tôi muốn cài `heat`, `magnum`, `octavia` nên sẽ lấy file cấu hình bằng lệnh sau:
```sh
wget https://raw.githubusercontent.com/TrongTan124/ghi-chep-devstack/master/Local_conf/local-stable-stein-magnum-heat.conf -O ./local.conf
```

Chạy script sau để bắt đầu cài đặt
```sh
byobu

./stack.sh
```

Quá trình cài đặt sẽ diễn ra khoảng 1h30p. Thông tin sau khi cài đặt xong
```sh
=========================                                                                                                                                                                      
DevStack Component Timing                                                                                                                                                                      
 (times are in seconds)                                                                                                                                                                        
=========================                                                                                                                                                                      
run_process           67                                                                                                                                                                       
test_with_retry        4                                                                                                                                                                       
apt-get-update         5                                                                                                                                                                       
osc                  462                                                                                                                                                                       
wait_for_service      35
git_timed            237
dbsync                37
pip_install          545
apt-get               26
-------------------------
Unaccounted time     2626
=========================
Total runtime        4044



This is your host IP address: 172.16.69.225
This is your host IPv6 address: ::1
Horizon is now available at http://172.16.69.225/dashboard
Keystone is serving at http://172.16.69.225/identity/
The default users are: admin and demo
The password: secretadmin
```

Thiết lập biến môi trường cho user `stack` như sau:
```sh
echo 'source /opt/stack/devstack/openrc admin admin' >> /opt/stack/.bashrc
```

Sau khi cài xong, bạn chuyển sang user stack để bắt đầu thao tác. hoặc đăng nhập vào horizon.

**NOTE**: Sau khi cài đặt xong, đừng khởi động hay tắt máy. vì sẽ bị mất cấu hình của `cinder, openvswitch`

**NOTE**: Cài đặt thêm gói `sudo pip install python-octaviaclient` với user `stack` để sử dụng được tập lệnh `openstack loadbalancer` trong trường hợp bạn sử dụng file localrc.conf có thêm octavia

## Tham khảo

- [https://docs.openstack.org/devstack/latest/](https://docs.openstack.org/devstack/latest/)