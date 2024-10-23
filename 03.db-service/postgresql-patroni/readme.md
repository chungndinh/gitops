# Link tham khảo:
https://serverstadium.com/knowledge-base/set-up-high-availability-postgresql-cluster-using-patroni-on-serverstadium/
# Mô hình
- OS: Ubuntu 22.04
- Patroni:
    - postgres-01: 192.16.8.2.241
    - postgres-02: 192.16.8.2.242
    - postgres-03: 192.16.8.2.243
- Etcd & Haproxy:
    - postgresql-etcd-01: 192.168.2.244
# Cài đặt PostgreSQL 14
-   Cài đặt trên các node của Patroni
    ```
    sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
    apt update
    apt -y install postgresql-14 postgresql-server-dev-14
    systemctl stop postgresql && systemctl disable postgresql
    ```
# Cài đặt Patroni
-   Cài đặt trên các node của Patroni
    ```
    ln -s /usr/lib/postgresql/14/bin/* /usr/sbin/ 
    apt -y install python3 python3-pip
    pip install --upgrade setuptools
    pip install psycopg2
    pip install patroni
    pip install python-etcd
    ```


# Cấu hình Patroni
-   Cài đặt trên các node của Patroni
    ```
    sudo nano /etc/patroni.yml
    ```
- Copy theo file từng patroni tương ứng theo từng node Patroni. Trong folder configs/patroni.

    ```
    mkdir -p /data/patroni
    sudo chown -R postgres:postgres /data/patroni
    sudo chmod -R 700 /data/patroni
    ```

    ```
    sudo nano /etc/systemd/system/patroni.service
    ```
-  Thêm config như bên dưới:

    ```
    [Unit]
    Description=High availability PostgreSQL Cluster
    After=syslog.target network.target
    [Service]
    Type=simple
    User=postgres
    Group=postgres
    ExecStart=/usr/local/bin/patroni /etc/patroni.yml
    KillMode=process
    TimeoutSec=30
    Restart=no

    [Install]
    WantedBy=multi-user.target
    ```
- Reload và enable patroni:
    ```
    sudo systemctl daemon-reload
    sudo systemctl enable patroni
    ```
- Cấu hình watchdog:

    ```
    modprobe softdog
    chown postgres /dev/watchdog
    ```
- Start patroni và kiểm tra status:    
    ```
    systemctl start patroni
    systemctl status patroni
    ```
- Check patroni list
    ```
    patronictl -c /etc/patroni.yml list   
    ``` 
---
# Cài đặt Etcd và Haproxy
## Cài đặt Etcd:
-   Cài đặt trên node postgresql-etcd-01:
    ```
    sudo apt -y install etcd
    sudo nano /etc/default/etcd
    ```
-   Copy cấu hình như bên dưới
    ```
    ETCD_LISTEN_PEER_URLS="http://192.168.2.244:2380"
    ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://192.168.2.244:2379"
    ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.2.244:2380"
    ETCD_INITIAL_CLUSTER="default="http://192.168.2.244:2380"
    ETCD_ADVERTISE_CLIENT_URLS="http://192.168.2.244:2379"
    ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
    ETCD_INITIAL_CLUSTER_STATE=new
    ```
-   Enable và start etcd
    ```
    sudo systemctl enable etcd
    sudo systemctl start etcd
    ```


## Cài đặt HAProxy
-   Cài đặt trên node postgresql-etcd-01:
    ```
    sudo apt -y install haproxy
    mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.org
    nano /etc/haproxy/haproxy.cfg
    ```
- Copy theo file configs/haproxy

- Validate lại file cấu hình và start haproxy:
    ```
    sudo /usr/sbin/haproxy -c -V -f /etc/haproxy/haproxy.cfg 
    sudo systemctl enable haproxy
    sudo systemctl start haproxy
    ```
# Kết nối
- Kết nối tới 192.168.2.244:5000 với user posgre và password trong patroni
- Sử dụng trang stats của haproxy để theo dõi node nào đang làm lead và replica
# Patroni
- Chuyển lead:
    - Chuyển lead từ node này sang node khác:
        ```
        patronictl -c /etc/patroni.yml switchover --master <current-leader> --candidate <new-leader>
        ```
    - Chuyển lead force:
        ```
        patronictl -c /etc/patroni.yml failover --candidate <new-leader>
        ```
## Update mới:
# Mô hình:
- 3 Node (ETCD + Patroni) + 1 Haproxy (Read | Read/Write)
- pgBackRest (backup tool used to perform PostgreSQL database backup, archiving, restoration, and point-in-time recovery)
- Link tham khảo: https://docs.percona.com/postgresql/14/solutions/high-availability.html#key-benefits-of-patroni
# Cấu hình:
- Link tham khảo: https://docs.percona.com/postgresql/14/solutions/ha-setup-apt.html
- Cài etcd: https://docs.percona.com/postgresql/14/how-to.html

