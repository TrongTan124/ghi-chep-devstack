## Giới thiệu

Tùy thuộc vào file cấu hình chỉ định để devstack cài đặt mà ta đặt tên cho phù hợp.

Nếu bạn đặt tên file là `local.conf` thì trong tập tin cấu hình ta có dòng đầu tin sẽ như sau
```sh
[[local|localrc]]
```

Còn nếu đặt tên file là `localrc` thì trong tập tin sẽ không cần dòng `[[local|localrc]]` ở đầu nữa.

Tại sao có sự khác biệt như vậy. Vì:

- Khi bạn đặt tin tập tin là `local.conf` thì bạn có thể tùy chỉnh thông tin cấu hình. Ví dụ,  bằng cách đặt vào trong tập tin cấu hình đoạn sau:
```sh
[[post-config|$NOVA_CONF]]
[DEFAULT]
use_syslog = True
[osapi_v3]
enabled = False
```

bạn có thể chỉnh các thông tin cấu hình theo cách bạn muốn với nova.

Bạn muốn biết tên biến chỉ tới file cấu hình nào thì đọc trong devstack để biết. Vào `/opt/stack/devstack/lib` và đọc thông tin file tương ứng.
```sh
root@devstack:~# cd /opt/stack/devstack/lib
root@devstack:/opt/stack/devstack/lib# grep "CONF" nova
NOVA_CONF_DIR=/etc/nova
NOVA_CONF=$NOVA_CONF_DIR/nova.conf
```

## Tham khảo

- [https://docs.openstack.org/devstack/latest/](https://docs.openstack.org/devstack/latest/)