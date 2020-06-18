## Giới thiệu

Devstack giúp bạn triển khai nhanh một hệ thống Openstack bằng 1 script.

## Chuẩn bị

Bạn cần một máy chủ chạy Ubuntu 18.04 64bits.

Cấu hình phụ thuộc vào số lượng project bạn muốn cài. Trong trường hợp của tôi, cấu hình như sau:
```sh
CPU: 4 vCPU
RAM: 8096 GB
Disk: 100GB
Interface: eth0 ra internet với IP 192.168.10.34
```

## Cài đặt

Trước tiên cần chắc chắn hệ điều hành đã được cập nhật mới nhất
```sh
apt update && apt dist-upgrade -y
```

### Cấu hình IP cho ifupdown

Cấu hình ifupdown như sau:

- Cập nhật gói:
```sh
apt update -y && apt dist-upgrade -y
```

- Cài đặt ifupdown:
```sh
apt install ifupdown -y
```

- Cấu hình IP cho interface `ens3`. Đây là interface mặc định khi cài đặt HĐH xong sẽ có
```sh
cat << EOF > /etc/network/interfaces
# loopback network interface
auto lo
iface lo inet loopback

# external network interface
auto ens3
iface ens3 inet static
address 192.168.10.34
netmask 255.255.255.0
gateway 192.168.10.1
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

- Cập nhật HĐH khi thêm repo:
```sh
apt update -y && apt dist-upgrade -y && apt autoremove -y
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

- Chỉnh sửa lại `hosts` của máy chủ. Sử dụng ansible hoặc tự sửa tay trong file `/etc/hosts`

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
git clone https://git.openstack.org/openstack-dev/devstack --branch stable/ussuri
cd devstack
```

Tôi muốn cài `heat`, `magnum`, `octavia` nên sẽ lấy file cấu hình bằng lệnh sau:
```sh
wget https://raw.githubusercontent.com/TrongTan124/ghi-chep-devstack/master/Local_conf/local-stable-ussuri-magnum-heat-octavia.conf -O ./local.conf
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
run_process           37
test_with_retry        5
apt-get-update         3
osc                  502
wait_for_service      24
git_timed            412
dbsync                25
pip_install          567
apt-get              325
-------------------------
Unaccounted time     1770
=========================
Total runtime        3670



This is your host IP address: 192.168.10.34
This is your host IPv6 address: ::1
Horizon is now available at http://192.168.10.34/dashboard
Keystone is serving at http://192.168.10.34/identity/
The default users are: admin and demo
The password: secretadmin

WARNING:
Using lib/neutron-legacy is deprecated, and it will be removed in the future


Services are running under systemd unit files.
For more information see:
https://docs.openstack.org/devstack/latest/systemd.html

DevStack Version: ussuri
Change: 7bfdd381046f85f1d9a33992216fc6bab1ee7666 Fix uwsgi issues 2020-06-16 17:22:32 +0200
OS Version: Ubuntu 18.04 bionic
```

Thiết lập biến môi trường cho user `stack` như sau:
```sh
echo 'source /opt/stack/devstack/openrc admin admin' >> /opt/stack/.bashrc
```

Sau khi cài xong, bạn chuyển sang user stack để bắt đầu thao tác. hoặc đăng nhập vào horizon.

**NOTE**: Sau khi cài đặt xong, đừng khởi động hay tắt máy. vì sẽ bị mất cấu hình của `cinder, openvswitch`

## Tham khảo

- [https://docs.openstack.org/devstack/latest/](https://docs.openstack.org/devstack/latest/)