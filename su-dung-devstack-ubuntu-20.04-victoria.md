## Giới thiệu

Devstack giúp bạn triển khai nhanh một hệ thống Openstack bằng 1 script trên hệ điều hành Ubuntu

Nếu bạn muốn dựng nhanh một hệ thống OpenStack trên hệ điều hành CentOS thì sử dụng PackStack.

## Chuẩn bị

Bạn cần một máy chủ chạy Ubuntu 20.04 64bits.

Cấu hình phụ thuộc vào số lượng project bạn muốn cài. Trong trường hợp của tôi, cấu hình như sau:
```sh
CPU: 8 vCPU
RAM: 12 GB
HDD1: 50GB
HDD2: 50GB
Interface: eth0 ra internet với IP 172.20.0.10
```

## Cài đặt

Trước tiên cần chắc chắn hệ điều hành đã được cập nhật mới nhất. Chạy lệnh cập nhật sau:
```sh
apt update && apt dist-upgrade -y
```

### Cấu hình IP cho ifupdown

Mặc định của Ubuntu 20.04 sử dụng netplan để quản lý các cổng kết nối.

Do tôi chọn sử dụng OpenvSwitch làm switch ảo khi dựng cụm OpenStack. Nếu sử dụng netplan thì sẽ gặp vấn đề với OpenvSwitch. Nên tôi sẽ gỡ bỏ netplan và cài đặt ifupdown để quản lý network trong máy chủ Ubuntu.

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
auto ens160
iface ens160 inet static
address 172.20.0.10
netmask 255.255.255.0
gateway 172.20.0.1
dns-nameservers 1.1.1.1 8.8.8.8
EOF
```

- Thiết lập IP vừa được cấu hình
```sh
ifdown --force ens160 lo && ifup -a
```

- Gỡ netplan:
```sh
systemctl stop networkd-dispatcher
systemctl disable networkd-dispatcher
systemctl mask networkd-dispatcher
apt purge nplan netplan.io -y
```

- Thiết lập lại DNS cho máy chủ do ifupdown ko quản lý DNS.
```sh
apt install resolvconf -y

echo "nameserver 1.1.1.1" >> /etc/resolv.conf
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
echo "nameserver 1.1.1.1" >> /etc/resolvconf/resolv.conf.d/head
echo "nameserver 8.8.8.8" >> /etc/resolvconf/resolv.conf.d/head
sudo service resolvconf restart
```

- Khởi động lại máy chủ trước khi chuyển sang cài đặt OpenStack
```sh
init 6
```

### Chạy script cài đặt

Devstack nên được chạy với user khác root, vì thế, ta sẽ tạo một username `stack` để chạy Devstack
```sh
useradd -s /bin/bash -d /opt/stack -m stack
```

User `stack` sẽ thao tác nhiều với quyền root nên ta thiết lập cấu hình để không cần nhập password root khi chạy lệnh sudo
```sh
echo "stack ALL=(ALL) NOPASSWD: ALL" | tee /etc/sudoers.d/stack
su - stack
```

Bạn thay branch của devstack phù hợp với branch trong file localrc để cài đặt.

Thực hiện tải một bản devstack về
```sh
git clone https://git.openstack.org/openstack-dev/devstack --branch stable/victoria
cd devstack
```

Tôi chỉ cài đặt các project core nên sẽ lấy file cấu hình bằng lệnh sau:
```sh
wget https://raw.githubusercontent.com/TrongTan124/ghi-chep-devstack/master/Local_conf/local-stable-victoria.conf -O ./local.conf
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
wait_for_service      18
pip_install          437
apt-get              443
run_process           48
dbsync                38
git_timed            625
apt-get-update         8
test_with_retry        3
osc                  241
-------------------------
Unaccounted time     414
=========================
Total runtime        2275



This is your host IP address: 172.20.0.10
This is your host IPv6 address: ::1
Horizon is now available at http://172.20.0.10/dashboard
Keystone is serving at http://172.20.0.10/identity/
The default users are: admin and demo
The password: secretadmin

Services are running under systemd unit files.
For more information see:
https://docs.openstack.org/devstack/latest/systemd.html

DevStack Version: victoria
Change: 7a0e578d19d269cfab15a6fe80b4717c0318e76e Workaround for new pip 20.3 behavior 2020-11-30 23:06:46 +0000
OS Version: Ubuntu 20.04 focal
```

Thiết lập biến môi trường cho user `stack` như sau:
```sh
echo 'source /opt/stack/devstack/openrc admin admin' >> /opt/stack/.bashrc
```

Sau khi cài xong, bạn chuyển sang user stack để bắt đầu thao tác. hoặc đăng nhập vào horizon.

**NOTE**: Sau khi cài đặt xong, đừng khởi động hay tắt máy. vì sẽ bị mất cấu hình của `cinder, openvswitch`

## Tham khảo

- [https://docs.openstack.org/devstack/latest/](https://docs.openstack.org/devstack/latest/)