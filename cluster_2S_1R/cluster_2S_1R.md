# Doc
## 1. Tạo thư mục
```cmd
mkdir cluster_2S_1R
cd cluster_2S_1R

# Create clickhouse-keeper directories
for i in {01..03}; do
  mkdir -p fs/volumes/clickhouse-keeper-${i}/etc/clickhouse-keeper
done

# Create clickhouse-server directories
for i in {01..02}; do
  mkdir -p fs/volumes/clickhouse-${i}/etc/clickhouse-server
done
```

```cmd
for i in {01..02}; do
  mkdir -p fs/volumes/clickhouse-${i}/etc/clickhouse-server/config.d
  mkdir -p fs/volumes/clickhouse-${i}/etc/clickhouse-server/users.d
  touch fs/volumes/clickhouse-${i}/etc/clickhouse-server/config.d/config.xml
  touch fs/volumes/clickhouse-${i}/etc/clickhouse-server/users.d/users.xml
done
```

## 2. Configure node (config.xml)
### Server setup 
- Mở cổng lắng nghe các external network (docker, máy trong mạng LAN)

```xml
<listen_host>0.0.0.0</listen_host>
```

- API dành cho curl,...
```xml
<http_port>8123</http_port>
```

- Port cho các server ClickHouse (ClickHouse CLI)
```xml
<tcp_port>9000</tcp_port>
```

- Log
```xml
<logger>
    <level>debug</level>
    <log>/var/log/clickhouse-server/clickhouse-server.log</log>          
    <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
    <size>1000M</size>
    <count>3</count>
</logger>
```

| level   | ý nghĩa               |
| ------- | --------------------- |
| trace   | chi tiết cực nhiều    |
| debug   | debug                 |
| info    | thông tin bình thường |
| warning | cảnh báo              |
| error   | lỗi                   |

- `<log>/var/log`: file log chính
- `<errorlog>/var/log`: file log chỉ chứa lỗi
- Khi log đạt 1000MB → tạo file mới
- Giữ tối đa 3 file log

### Cluster Configuration
Nó nói cho ClickHouse biết:
- Cluster có bao nhiêu node
- Node nào thuộc shard nào
- Replica nằm ở đâu

```xml
<remote_servers>
    <cluster_2S_1R>
        <shard>
            <replica>
                <host>clickhouse-01</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <replica>
                <host>clickhouse-02</host>
                <port>9000</port>
            </replica>
        </shard>
    </cluster_2S_1R>
</remote_servers>
```

- Nếu muốn thêm replicate thì khai lặp lại nhiều lần

### Zookeeper
```xml
<zookeeper>
        <node>
            <host>clickhouse-keeper-01</host>
            <port>9181</port>
        </node>
        <node>
            <host>clickhouse-keeper-02</host>
            <port>9181</port>
        </node>
        <node>
            <host>clickhouse-keeper-03</host>
            <port>9181</port>
        </node>
    </zookeeper>
```

### Macros configuration
Định nghĩa biến sẵn đỡ nhớ để type

```xml
<macros>
    <shard>01</shard>
    <replica>01</replica>
</macros>
```

### User configuration (trong config.xml) & quyền
```xml
<user_directories>
    <users_xml>
        <path>users.xml</path>
    </users_xml>
    <local_directory>
        <path>/var/lib/clickhouse/access/</path>
    </local_directory>
</user_directories>
```

### distributed_ddl
ClickHouse sẽ lưu task DDL vào path này trong Keepe

```xml
<distributed_ddl>
    <path>/clickhouse/task_queue/ddl</path>
</distributed_ddl>
```

## 3. Configure keeper server (keeper_config.xml)
Thay toàn bộ cấu hình cũ bằng cái này khi merge với config chính

```xml
<clickhouse replace="true">
```


```xml
<tcp_port>9181</tcp_port>   -- Cổng cho client kết nối
<server_id>1</server_id>    -- id node này trong cluster
<log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>       -- Lưu raft log (metadata replication)
<snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>       -- Lưu snapshot raft để khôi phục nhanh
```

```xml
<raft_configuration>
    <server>
        <id>1</id>
        <hostname>clickhouse-keeper-01</hostname>
        <port>9234</port>
    </server>
    <server>
        <id>2</id>
        <hostname>clickhouse-keeper-02</hostname>
        <port>9234</port>
    </server>
    <server>
        <id>3</id>
        <hostname>clickhouse-keeper-03</hostname>
        <port>9234</port>
    </server>
</raft_configuration>
```
* Đây là **danh sách tất cả node trong cluster Raft**
* Mỗi node có `id`, `hostname`, `port` → các node Raft dùng để đồng bộ metadata
* Nhờ đây, cluster **tự bầu leader + replicate state giữa các Keeper nodes**

## 4. users.xml
### `<profiles>`

```xml
<profiles>
    <default>
        <max_memory_usage>10000000000</max_memory_usage>
        <use_uncompressed_cache>0</use_uncompressed_cache>
        <load_balancing>in_order</load_balancing>
        <log_queries>1</log_queries>
    </default>
</profiles>
```

* **Profile** = một bộ **cấu hình cho user**, bao gồm các giới hạn và cách hoạt động.
* Ở đây có profile `default` dùng chung cho user default.

Các thuộc tính quan trọng:

| Thuộc tính               | Ý nghĩa                                                                        |
| ------------------------ | ------------------------------------------------------------------------------ |
| `max_memory_usage`       | Giới hạn bộ nhớ tối đa cho query (10 GB)                                       |
| `use_uncompressed_cache` | Có dùng cache uncompressed không (0 = không)                                   |
| `load_balancing`         | Khi query trên cluster → cách phân phối query (in_order = theo thứ tự replica) |
| `log_queries`            | Có log query không (1 = có)                                                    |

---

### `<users>`

```xml
<users>
    <default>
        <access_management>1</access_management>
        <profile>default</profile>
        <networks>
            <ip>::/0</ip>
        </networks>
        <quota>default</quota>
        <named_collection_control>1</named_collection_control>
        <show_named_collections>1</show_named_collections>
        <show_named_collections_secrets>1</show_named_collections_secrets>
    </default>
</users>
```

* Đây định nghĩa **user `default`**.

Các thuộc tính:

| Thuộc tính                       | Ý nghĩa                                                            |
| -------------------------------- | ------------------------------------------------------------------ |
| `access_management`              | Cho phép user tạo/drop user, role (1 = bật)                        |
| `profile`                        | Gán profile đã định nghĩa → ở đây dùng `default`                   |
| `networks`                       | Quyền truy cập từ IP nào → `::/0` = tất cả IP (IPv6 + IPv4 mapped) |
| `quota`                          | Gán quota → ở đây là `default`                                     |
| `named_collection_control`       | Cho phép user quản lý named collections (1 = bật)                  |
| `show_named_collections`         | Hiển thị named collections (1 = bật)                               |
| `show_named_collections_secrets` | Hiển thị các secret trong named collections (1 = bật)              |

---

### `<quotas>`

```xml
<quotas>
    <default>
        <interval>
            <duration>3600</duration>
            <queries>0</queries>
            <errors>0</errors>
            <result_rows>0</result_rows>
            <read_rows>0</read_rows>
            <execution_time>0</execution_time>
        </interval>
    </default>
</quotas>
```

* **Quota** = giới hạn tài nguyên của user trong khoảng thời gian
* `duration=3600` → tính trong 1 giờ
* Các thông số `queries`, `errors`, `result_rows`, `read_rows`, `execution_time` = 0 → **không giới hạn**

> Nói cách khác, user `default` được phép query thoải mái, không bị limit.

---

### Tóm tắt

* `profiles` → định nghĩa **cách user chạy query**
* `users` → định nghĩa **user, quyền, profile, IP, quota**
* `quotas` → định nghĩa **giới hạn tài nguyên**, ở đây không giới hạn

-----

## **5. Check**
- Vô CLI
```cmd
docker exec -it clickhouse-01 clickhouse-client
```

---

### Xem thông tin cluster
```sql
SELECT 
    cluster,
    shard_num,
    replica_num,
    host_name,
    port
FROM system.clusters;
```

### Xem keeper
- Cách 1: SQL
```sql
SELECT *
FROM system.zookeeper
WHERE path IN ('/', '/clickhouse');
```

- Cách 2: `mntr`, đổi keeper chạy là biết ai là leader ngay

```cmd
docker exec -it clickhouse-keeper-01  /bin/sh -c 'echo mntr | nc 127.0.0.1 9181'
```

## 6. Demo
```sql
CREATE DATABASE IF NOT EXISTS uk ON CLUSTER cluster_2S_1R;
```

```sql
SHOW DATABASES;
```

```sql
CREATE TABLE IF NOT EXISTS uk.uk_price_paid_local
ON CLUSTER cluster_2S_1R
(
    price UInt32,
    date Date,
    postcode1 LowCardinality(String),
    postcode2 LowCardinality(String),
    type Enum8('terraced' = 1, 'semi-detached' = 2, 'detached' = 3, 'flat' = 4, 'other' = 0),
    is_new UInt8,
    duration Enum8('freehold' = 1, 'leasehold' = 2, 'unknown' = 0),
    addr1 String,
    addr2 String,
    street LowCardinality(String),
    locality LowCardinality(String),
    town LowCardinality(String),
    district LowCardinality(String),
    county LowCardinality(String)
)
ENGINE = MergeTree
ORDER BY (postcode1, postcode2, addr1, addr2);

CREATE TABLE uk.uk_price_paid
ON CLUSTER cluster_2S_1R
AS uk.uk_price_paid_local
ENGINE = Distributed(
    cluster_2S_1R,
    uk,
    uk_price_paid_local,
    rand()
);
```
--- 

- **LowCardinality**: kiểu extend của string. Nếu 1 trường có ít giá trị hơn rất nhiều so bảng ghi, set kiểu này nó sẽ tạo Dictionary. Tối ưu truy vấn.

- **UInt8/UInt32**: U là số dương -> Tiết kiệm nếu biết gtri ko âm

- **Enum8/Enum16**: Giới hạn giá trị map số cho ClickHouse query nhanh hơn.

- rand(): trường shard nghĩa là bắn vào 5 triệu data nó ngẫu nhiên mỗi row 1 shard.
---

create table on cluster thì nó tạo cho 2 server nhưng khi query phải có distributed nếu ko mỗi server là 1 db riêng lẻ.
- Tạo bảng local -> Tạo bảng lưu router/query proxy.

```text
              (Distributed)
                     │
         ┌───────────┴───────────┐
         │                       │
       local                    local
    (MergeTree)              (MergeTree)
      node1                    node2

```
---

`Lưu ý`:
- XML hay yaml chỉ là set môi trường muốn nó được hay ko phải dùng thêm engine.

Ví dụ: node 1 replicate shard 1, node 2 replicate shard 1. ko có nghĩa 2 node này sync data nhau phải gắn engine cho 1 table. Nếu muốn đồng bộ tạo bảng với engine ReplicateMergeTree

--- 

[Coi đầy đủ scale và repli](https://clickhouse.com/docs/architecture/cluster-deployment)