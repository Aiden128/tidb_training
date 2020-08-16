------

------

# [High Performance TiDB] Week 01：TiDB 整体架构

本週心得：

認識 TiDB 已經兩三年了，發現 TiDB 在整體架構上又精進了不少，包括 Tiup、TiCDC、TiFlash 等重要工具都提供了很好的解決方案。今年開始在台灣從事金融領域的工作，需要一組高強度的 DB 作為後端 Support，剛好又看到 High Performance TiDB 課程，希望先藉由學習這個課程並開始在台灣籌組相關社群！



## 題目要求

```reStructuredText
本地下载 TiDB，TiKV，PD 源代码，改写源码并编译部署以下环境：

- 1 TiDB
- 1 PD
- 3 TiKV
  改写后：使得 TiDB 启动事务时，能打印出一个 “hello transaction” 的日志

输出：一篇文章介绍以上过程
```



### 背景與環境準備

先簡述各個元件的功能，

- TiDB：計算層，負責處理 SQL 解譯成對應 TiKV 使用，當然實際上處理沒有這麼簡單，如何將單一 SQL 指令轉譯成分散式 KV 處理架構是需要很多最佳化的，詳細概念可以參考：https://pingcap.com/blog-cn/tidb-internal-2/
- TiKV：儲存層，負責實際進行資料的儲存，是一個分散式的 Key/Value 服務，詳細概念可以參考：https://pingcap.com/blog-cn/tidb-internal-1/
- PD：調度層（我自己稱為保母層），負責對各個元件進行監看、修復、負載平衡等工作，一樣可以參考：https://pingcap.com/blog-cn/tidb-internal-3/

再來進行編譯前的環境準備，我是在 Server 上編譯不是在本地端，用的是 ubuntu 18.04 的環境，首先

1. 安裝 make

```
sudo apt install make cmake gcc unzip -y
```

2. Golang 環境設定，要先安裝 golang 給 tidb 使用：

```
wget https://dl.google.com/go/go1.13.9.linux-amd64.tar.gz 
tar xf go1.13.9.linux-amd64.tar.gz 
sudo mv go /usr/local/go-1.13 
export GOROOT=/usr/local/go-1.13 
export PATH=$GOROOT/bin:$PATH
```

3. 再來是 rust 的部份，可以直接用：

```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

或是可以安裝 cargo。環境都設定好了之後，再找到三個相對應的 repository如下：

```
git clone https://github.com/pingcap/tidb
git clone https://github.com/tikv/tikv
git clone https://github.com/pingcap/pd
```

順利 clone 完之後，會看到三個資料夾分別是 tidb, tikv, pd。



### TiDB

直接執行執行以下命令：

```
cd tidb
make
```

順利的話可以看到在 bin 目錄下會有一個 tidb-server 

### TiKV

```
cd tikv
make build
```

應該可以在 ./tikv/target/debug/ 找到兩個 tikv-server 與 tikv-ctl 文件。

### PD

在安装好TiDB编译所需go环境后，PD无需多余的环境配置即可编译。

在根目录执行以下命令：

```
cd pd
make
```

應該可以在 ./pd/bin/ 找到兩個 pd-server 和 pd-ctl 檔案。



## 開始佈署

### Deploy PD 

TiKV 原生需要是透過 cluster 的方式運作，其中擔任 cluster 的管理者（保母）角色的就是PD，因此要先啓動 PD

```
./pd-server --name=pd-master \
            --data-dir=pd-master \
            --client-urls="http://127.0.0.1:2379" \
            --peer-urls="http://127.0.0.1:2380" \
            --initial-cluster="pd-master=http://127.0.0.1:2380" \
            --log-file=pd-master.log
```



### Deploy TiKV 

啟動三台 TiKV Server（記得要使用不同的 port），並連結到 PD Server 上：

```
./bin/tikv-server --pd-endpoints="127.0.0.1:2379" \
                --addr="127.0.0.1:20161" \
                --data-dir=tikv1 \
                --log-file=tikv1.log
                
./bin/tikv-server --pd-endpoints="127.0.0.1:2379" \
                --addr="127.0.0.1:20162" \
                --data-dir=tikv2 \
                --log-file=tikv2.log 

./bin/tikv-server --pd-endpoints="127.0.0.1:2379" \
                --addr="127.0.0.1:20163" \
                --data-dir=tikv3 \
                --log-file=tikv3.log
```



透過 pd-ctl 確認整體 tikv 狀態：

```
./bin/pd-ctl store -u http://127.0.0.1:2379
```



如果成功會輸出以下訊息：

```
{
  "count": 3,
  "stores": [
    {
      "store": {
        "id": 1,
        "address": "127.0.0.1:20161",
        "version": "4.1.0-alpha",
        "status_address": "127.0.0.1:20180",
        "git_hash": "ae7a6ecee6e3367da016df0293a9ffe9cc2b5705",
        "start_timestamp": 1597577471,
        "deploy_path": "/home/ubuntu",
        "last_heartbeat": 1597577631229153737,
        "state_name": "Up"
      },
      "status": {
        "capacity": "38.6GiB",
        "available": "19.5GiB",
        "used_size": "31.51MiB",
        "leader_count": 1,
        "leader_weight": 1,
        "leader_score": 1,
        "leader_size": 1,
        "region_count": 1,
        "region_weight": 1,
        "region_score": 1,
        "region_size": 1,
        "start_ts": "2020-08-16T11:31:11Z",
        "last_heartbeat_ts": "2020-08-16T11:33:51.229153737Z",
        "uptime": "2m40.229153737s"
      }
    },
    {
      "store": {
        "id": 4,
        "address": "127.0.0.1:20162",
        "version": "4.1.0-alpha",
        "status_address": "127.0.0.1:20180",
        "git_hash": "ae7a6ecee6e3367da016df0293a9ffe9cc2b5705",
        "start_timestamp": 1597577481,
        "deploy_path": "/home/ubuntu",
        "last_heartbeat": 1597577631196926289,
        "state_name": "Up"
      },
      "status": {
        "capacity": "38.6GiB",
        "available": "19.5GiB",
        "used_size": "31.5MiB",
        "leader_count": 0,
        "leader_weight": 1,
        "leader_score": 0,
        "leader_size": 0,
        "region_count": 1,
        "region_weight": 1,
        "region_score": 1,
        "region_size": 1,
        "start_ts": "2020-08-16T11:31:21Z",
        "last_heartbeat_ts": "2020-08-16T11:33:51.196926289Z",
        "uptime": "2m30.196926289s"
      }
    },
    {
      "store": {
        "id": 6,
        "address": "127.0.0.1:20163",
        "version": "4.1.0-alpha",
        "status_address": "127.0.0.1:20180",
        "git_hash": "ae7a6ecee6e3367da016df0293a9ffe9cc2b5705",
        "start_timestamp": 1597577554,
        "deploy_path": "/home/ubuntu",
        "last_heartbeat": 1597577624458576263,
        "state_name": "Up"
      },
      "status": {
        "capacity": "38.6GiB",
        "available": "19.5GiB",
        "used_size": "31.5MiB",
        "leader_count": 0,
        "leader_weight": 1,
        "leader_score": 0,
        "leader_size": 0,
        "region_count": 1,
        "region_weight": 1,
        "region_score": 1,
        "region_size": 1,
        "start_ts": "2020-08-16T11:32:34Z",
        "last_heartbeat_ts": "2020-08-16T11:33:44.458576263Z",
        "uptime": "1m10.458576263s"
      }
    }
  ]
}
```

如果內容正確，表示 TiKV 已經正確連接，也可以看到 state_name 都是 up。



### Deploy TiDB

TiDB 佈署非常簡單，只要指定 PD 位址就好，執行：

（TODO：確認一下 pd 怎麼 HA）

```
./bin/tidb-server --store=tikv \
                  --path="127.0.0.1:2379" \
                  --log-file=tidb.log
```

初次啟動的時候 tidb 會自動做一些 initial ，所以 log 會噴很多 warning ，大概一分鐘之後出現以下內容就是正確的：

```
[2020/08/16 11:41:34.999 +00:00] [INFO] [server.go:235] ["server is running MySQL protocol"] [addr=0.0.0.0:4000]
[2020/08/16 11:41:34.999 +00:00] [INFO] [http_status.go:82] ["for status and metrics report"] ["listening on addr"=0.0.0.0:10080]
```

可以透過 sql client 測試看看：

```
mysql -h 127.0.0.1 -P 4000 -u root -D test
```

如果看到以下表示成功：

```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.7.25-TiDB-v4.0.0-beta.2-960-g5184a0d70-dirty TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible
...
```

確認一下可以正常 Select：

```
select * from information_schema.cluster_info;
```

出現 Content 就表示成功了。

```
mysql> select * from information_schema.cluster_info;
+------+-----------------+-----------------+--------------+------------------------------------------+----------------------+------------------+
| TYPE | INSTANCE        | STATUS_ADDRESS  | VERSION      | GIT_HASH                                 | START_TIME           | UPTIME           |
+------+-----------------+-----------------+--------------+------------------------------------------+----------------------+------------------+
| tidb | 0.0.0.0:4000    | 0.0.0.0:10080   | 4.0.0-beta.2 | 5184a0d7060906e2022d18f11532f119f5df3f39 | 2020-08-16T11:39:20Z | 4m30.898207032s  |
| tidb | 0.0.0.0:4000    | 0.0.0.0:10080   | 4.0.0-beta.2 | 5184a0d7060906e2022d18f11532f119f5df3f39 | 2020-08-16T11:41:32Z | 2m18.898213852s  |
| pd   | 127.0.0.1:2379  | 127.0.0.1:2379  | 4.1.0-alpha  | 865fbd82a028aecfb875a20b932fee3ba4b8c73c | 2020-08-16T11:28:04Z | 15m46.898215934s |
| tikv | 127.0.0.1:20161 | 127.0.0.1:20180 | 4.1.0-alpha  | ae7a6ecee6e3367da016df0293a9ffe9cc2b5705 | 2020-08-16T11:31:11Z | 12m39.898217811s |
| tikv | 127.0.0.1:20162 | 127.0.0.1:20180 | 4.1.0-alpha  | ae7a6ecee6e3367da016df0293a9ffe9cc2b5705 | 2020-08-16T11:31:21Z | 12m29.898219544s |
| tikv | 127.0.0.1:20163 | 127.0.0.1:20180 | 4.1.0-alpha  | ae7a6ecee6e3367da016df0293a9ffe9cc2b5705 | 2020-08-16T11:32:34Z | 11m16.898221347s |
+------+-----------------+-----------------+--------------+------------------------------------------+----------------------+------------------+
6 rows in set (0.01 sec)
```

部署完成！（好興奮阿！！！）



## 開始改寫

這個禮拜的題目要求「使得 TiDB 啓動事務時，能打印出一個 ‘hello transaction’ 的日誌」。個人理解資料庫中所謂的「事務（Transaction）」，應該說的是每一次的 sql 操作流程，對於每條 SQL 操作，資料庫都會自動將其作為一個事務執行。

> A **database transaction** symbolizes a unit of work performed within a [database management system](https://en.wikipedia.org/wiki/Database_management_system) (or similar system) against a database, and treated in a coherent and reliable way independent of other transactions. A transaction generally represents any change in a database.

### Approach

把題目疏理成以下工作，

1. 確認一下在 TiDB 三個角色中那個元件是負責 Transaction。
2. 加上 log 。

根據角色負責的工作，TiDB 負責解譯跟編譯 SQL，也就是每次 SQL 語法的 flow 的開始跟結束點都是要透過 TiDB，因此可以先合理判斷 TiDB 中應該有對應的接口，到官方文件中搜尋一下，可以找到 [TiDB事务概览](https://docs.pingcap.com/zh/tidb/stable/transaction-overview)，應該是八九不離十，再搜尋一下 Code 可以發現 `tidb/kv/kv.go`中有宣告 Transaction 的 interface，主要是定義在 Storage 中：

```
// Storage defines the interface for storage.
// Isolation should be at least SI(SNAPSHOT ISOLATION)
type Storage interface {
    // Begin transaction
    Begin() (Transaction, error)
    // BeginWithStartTS begins transaction with startTS.
    BeginWithStartTS(startTS uint64) (Transaction, error)
    ...
}
```

TiDB 是把實作寫在每個對應支援的儲存，因此可以到  ./store/tikv/kv.go 中再找到實作的方式：

```go
func (s *tikvStore) Begin() (kv.Transaction, error) {
	txn, err := newTiKVTxn(s)
	if err != nil {
		return nil, errors.Trace(err)
	}
	return txn, nil
}

// BeginWithStartTS begins a transaction with startTS.
func (s *tikvStore) BeginWithStartTS(startTS uint64) (kv.Transaction, error) {
  txn, err := newTikvTxnWithStartTS(s, startTS, s.nextReplicaReadSeed())
  if err != nil {
    return nil, errors.Trace(err)
  }
  return txn, nil
}
```



兩個 fun 都要對應回傳 txn，再看一下 txn 的實作可以看到 /store/tikv/txn.go

```go
func newTiKVTxn(store *tikvStore) (*tikvTxn, error) {
	bo := NewBackofferWithVars(context.Background(), tsoMaxBackoff, nil)
	startTS, err := store.getTimestampWithRetry(bo)
	if err != nil {
		return nil, errors.Trace(err)
	}
	return newTikvTxnWithStartTS(store, startTS, store.nextReplicaReadSeed())
}
// newTikvTxnWithStartTS creates a txn with startTS.
func newTikvTxnWithStartTS(store tikvStore, startTS uint64, replicaReadSeed uint32) (tikvTxn, error) {
  ver := kv.NewVersion(startTS)
  snapshot := newTiKVSnapshot(store, ver, replicaReadSeed)
  return &tikvTxn{
    snapshot:  snapshot,
    us:        kv.NewUnionStore(snapshot),
    store:     store,
    startTS:   startTS,
    startTime: time.Now(),
    valid:     true,
    vars:      kv.DefaultVars,
  }, nil
}
```



可以發現最後都匯到 `func newTikvTxnWithStartTS(store *tikvStore, startTS uint64, replicaReadSeed uint32) (*tikvTxn, error)` 所以直接在這個 fun 加上 log 即可。

改写如下：

```
// newTikvTxnWithStartTS creates a txn with startTS.
func newTikvTxnWithStartTS(store *tikvStore, startTS uint64, replicaReadSeed uint32) (*tikvTxn, error) {
	ver := kv.NewVersion(startTS)
	snapshot := newTiKVSnapshot(store, ver, replicaReadSeed)
	
	// Print a "hello transaction" when begin a transaction
	logutil.BgLogger().Info("hello transaction", zap.Uint64("startTS", startTS))
	
	return &tikvTxn{
		snapshot:  snapshot,
		us:        kv.NewUnionStore(snapshot),
		store:     store,
		startTS:   startTS,
		startTime: time.Now(),
		valid:     true,
		vars:      kv.DefaultVars,
	}, nil
}
```

重新編譯 TiDB 再測試一些查詢，可以在 log 中看到 hello transaction ，比較要注意的是即使沒有執行 sql 也可以看到訊息一直出現，這是因為在 TiDB 中會一直自我進行 ddl checker 跟 gc ，透過開啟 debug mode 會看的比較清楚。

```
[2020/08/16 14:46:18.268 +09:00] [DEBUG] [ddl.go:210] ["[ddl] check whether is the DDL owner"] [isOwner=true] [selfID=9fb0292d-3ce5-4136-93d2-553dfd694c7a]
[2020/08/16 14:46:19.266 +09:00] [DEBUG] [ddl_worker.go:148] ["[ddl] wait to check DDL status again"] [worker="worker 2, tp add index"] [interval=1s]
[2020/08/16 14:46:19.266 +09:00] [DEBUG] [ddl_worker.go:148] ["[ddl] wait to check DDL status again"] [worker="worker 1, tp general"] [interval=1s]
[2020/08/16 14:46:19.267 +09:00] [INFO] [txn.go:99] ["hello transaction"] [startTS=418789924340498433]
```



## 總結

因為我們是在實作層進行更動，所以基本上並不是所有的儲存都支援這個方式，也許可以在比較上層的部份進行修正，之後可以再找同學討論看看。
