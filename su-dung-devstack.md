## Giới thiệu

Devstack giúp bạn triển khai nhanh một hệ thống Openstack với bằng 1 câu lệnh chạy script.

## Cài đặt

Devstack nên được chạy với user khác root, vì thế, ta sẽ tạo một username `stack` để chạy Devstack
```sh
sudo useradd -s /bin/bash -d /opt/stack -m stack
```

User `stack` sẽ thao tác nhiều với quyền root nên ta thiết lập cấu hình để không cần nhập password root khi chạy lệnh sudo
```sh
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo su - stack
```

Thực hiện tải một bản devstack về
```sh
git clone https://git.openstack.org/openstack-dev/devstack --branch stable/pike
cd devstack
```

Kiểm tra nhánh của devstack
```sh
git status
```

Nếu là nhánh master thì ta chuyển sang nhánh pike như sau (do tôi cài stable/pike)
```sh
git checkout stable/pike
```

Ta tạo một tập tin `localrc` để thiết lập cấu hình cho devstack cài đặt
```sh
BRANCH=stable/pike

enable_plugin barbican https://review.openstack.org/openstack/barbican $BRANCH
enable_plugin neutron-lbaas https://review.openstack.org/openstack/neutron-lbaas $BRANCH
enable_plugin octavia https://review.openstack.org/openstack/octavia $BRANCH
LIBS_FROM_GIT+=python-neutronclient

KEYSTONE_TOKEN_FORMAT=fernet

DATABASE_PASSWORD=secretdatabase
RABBIT_PASSWORD=secretrabbit
ADMIN_PASSWORD=secretadmin
SERVICE_PASSWORD=secretservice
SERVICE_TOKEN=111222333444

# Enable Logging
LOGFILE=/opt/stack/logs/stack.sh.log
VERBOSE=True
LOG_COLOR=True
SCREEN_LOGDIR=/opt/stack/logs

# Pre-requisite
ENABLED_SERVICES=key,rabbit,mysql

# Nova
NOVA_BRANCH=$BRANCH
ENABLED_SERVICES+=,n-api,n-obj,n-cpu,n-cond,n-sch

# Placement service needed for Nova
ENABLED_SERVICES+=,placement-api,placement-client

# Glance
GLANCE_BRANCH=$BRANCH
ENABLED_SERVICES+=,g-api,g-reg

# Neutron
NEUTRON_BRANCH=$BRANCH
ENABLED_SERVICES+=,q-svc,q-agt,q-dhcp,q-l3,q-meta,neutron

# Horizon
ENABLED_SERVICES+=,horizon

# Enable LBaaS V2
NEUTRON_LBAAS_BRANCH=$BRANCH
ENABLED_SERVICES+=,q-lbaasv2

# Cinder (optional)
CINDER_BRANCH=$BRANCH
ENABLED_SERVICES+=,cinder,c-api,c-vol,c-sch

# Tempest (optional)
#ENABLED_SERVICES+=,tempest

# Octavia
ENABLED_SERVICES+=,octavia,o-api,o-cw,o-hm,o-hk

```

Ở đây, tôi chỉ định cài đặt toàn bộ project trong Openstack là stable/pike.

Chạy script sau để bắt đầu cài đặt
```sh
./stack.sh
```

Quá trình cài đặt sẽ diễn ra khoảng 1h30p. Thông tin sau khi cài đặt xong
```sh
=========================
DevStack Component Timing
=========================
Total runtime    3255

run_process       22
test_with_retry    3
apt-get-update     9
pip_install      567
osc              251
wait_for_service  21
git_timed        230
dbsync           196
apt-get          747
=========================



This is your host IP address: 172.16.68.57
This is your host IPv6 address: ::1
Horizon is now available at http://172.16.68.57/dashboard
Keystone is serving at http://172.16.68.57/identity/
The default users are: admin and demo
The password: secretadmin
```

Thiết lập biến môi trường cho user `stack` như sau:
```sh
echo 'source /opt/stack/devstack/openrc admin admin' >> /opt/stack/.bashrc
```

Sau khi cài xong, bạn chuyển sang user stack để bắt đầu thao tác. hoặc đăng nhập vào horizon.

**NOTE**: Sau khi cài đặt xong, đừng khởi động hay tắt máy. vì sẽ bị mất cấu hình của `cinder, openvswitch`

## Tham khảo

- [https://docs.openstack.org/devstack/latest/](https://docs.openstack.org/devstack/latest/)