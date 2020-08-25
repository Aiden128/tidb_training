# # [High Performance TiDB]_Week 02：对 TiDB 进行基准测试

本週心得：

## 題目要求

```reStructuredText
使用 sysbench、go-ycsb 和 go-tpc 分别对 TiDB 进行测试并且产出测试报告。  
测试报告需要包括以下内容：  
  
* 部署环境的机器配置(CPU、内存、磁盘规格型号)，拓扑结构(TiDB、TiKV 各部署于哪些节点)  
* 调整过后的 TiDB 和 TiKV 配置  
* 测试输出结果  
* 关键指标的监控截图  
* TiDB Query Summary 中的 qps 与 duration  
* TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标  
* TiKV Details 面板中 grpc 的 qps 以及 duration  
  
输出：写出你对该配置与拓扑环境和 workload 下 TiDB 集群负载的分析，提出你认为的 TiDB 的性能的瓶颈所在(能提出大致在哪个模块即 可)  
  
截止时间：下周二（8.25）24:00:00(逾期提交不给分)
```

## 背景與環境準備

### 環境準備
這次使用的是 multipass 來準備虛擬環境（可以參考 [Multipass 介紹](https://sysadmins.co.za/getting-started-with-multipass-vms/)），因為 tiup 進行主機操作時可選擇用 key 或是密碼登入，同時該帳號要具備 root 權限，因此先準備 ssh private key，再用 cloud-init 的方式設定 multipass VM。

- 產生給 cloud-init 用的 ssh public key：
```
    ssh-keygen -b 2048 -f ~/.ssh/multipass -t rsa -q -N ""
```
這邊要注意一下，因為自己在建置過程中有使用 tiup destroy 重新建立 VM，執行 tiup destroy 的時候會順便把 ssh key 刪除（？！？！），所以記得要把這個 key 保留好。

- 把產生的 pubkey 到 cloud-init.yaml
```
$ cat ./cloud-init.yaml
#cloud-config
ssh_authorized_keys:
  - ssh-rsa AAAAB3.......hh32R ruan@mbp
```

- 生成 VM，依照規格用 script 產生，配置如下。

TODO

- 使用 tiup 佈署，不得不說，使用 tiup 實在沒什麼技術難點，只要將 topology.yaml 準備好進行 tiup 安裝很簡單就可以完成。先測試最小架構：

```
global:
  user: "tidb"
  ssh_port: 22
  deploy_dir: "/tidb-deploy"
  data_dir: "/tidb-data"

pd_servers:
  - host: 10.149.251.115
  - host: 10.149.251.137
  - host: 10.149.251.239

tidb_servers:
  - host: 10.149.251.121
  - host: 10.149.251.223
  - host: 10.149.251.140

tikv_servers:
  - host: 10.149.251.15
  - host: 10.149.251.214
  - host: 10.149.251.173

monitoring_servers:
  - host: 10.149.251.56

grafana_servers:
  - host: 10.149.251.56

alertmanager_servers:
  - host: 10.149.251.56
```
- 執行安裝

```shell
tiup cluster deploy tidb-test v4.0.0 ./topology.yaml --user ubuntu -i ~/.ssh/id_rsa
```
畫面跳出確認 topology 是否正確，執行 y 

- 檢查安裝結果

```
:~ $  tiup cluster display cluster-test

Starting component `cluster`:  display tidb-test
tidb Cluster: tidb-test
tidb Version: v4.0.0
ID                    Role          Host            Ports        OS/Arch       Status  Data Dir                      Deploy Dir
--                    ----          ----            -----        -------       ------  --------                      ----------
10.149.251.56:9093    alertmanager  10.149.251.56   9093/9094    linux/x86_64  Up      /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
10.149.251.56:3000    grafana       10.149.251.56   3000         linux/x86_64  Up      -                             /tidb-deploy/grafana-3000
10.149.251.115:2379   pd            10.149.251.115  2379/2380    linux/x86_64  Up      /tidb-data/pd-2379            /tidb-deploy/pd-2379
10.149.251.137:2379   pd            10.149.251.137  2379/2380    linux/x86_64  Up|L    /tidb-data/pd-2379            /tidb-deploy/pd-2379
10.149.251.239:2379   pd            10.149.251.239  2379/2380    linux/x86_64  Up|UI   /tidb-data/pd-2379            /tidb-deploy/pd-2379
10.149.251.56:9090    prometheus    10.149.251.56   9090         linux/x86_64  Up      /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
10.149.251.121:4000   tidb          10.149.251.121  4000/10080   linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
10.149.251.140:4000   tidb          10.149.251.140  4000/10080   linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
10.149.251.223:4000   tidb          10.149.251.223  4000/10080   linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
10.149.251.15:20160   tikv          10.149.251.15   20160/20180  linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
10.149.251.173:20160  tikv          10.149.251.173  20160/20180  linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
10.149.251.214:20160  tikv          10.149.251.214  20160/20180  linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160

```

- 確認狀態正常後可以連到 TiDB 進行
```
mysql -h 127.0.0.1 -p 4000 -u root

mysql> create database sbtest;
Query OK, 0 rows affected (2.95 sec)
```

### 測試工具安裝

#### go-tpc
1. 用 script 方式安裝
```
curl --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/pingcap/go-tpc/master/install.sh | sh
```
安裝後記得要用 source 啟動 .bashrc

## 作業：測試報告
#### 部署环境的机器配置(CPU、内存、磁盘规格型号)，拓扑结构(TiDB、TiKV 各部署于哪些节点)  

|實例 | 個數 | 物理機配置 | IP
| :-- | :-- | :-- | :-- | :-- |
| TiDB |3 | 16 VCore 24GB * 1 | 10.149.251.103 <br/> 10.149.251.81 <br/> 10.149.251.67
| PD | 3 | 4 VCore 8GB * 1 |10.149.251.27 <br/> 10.149.251.46 <br/> 10.149.251.241
| TiKV | 3 | 16 VCore 24 GB | 10.149.251.142 <br/> 10.149.251.123 <br/> 10.149.251.53 
| Monitoring & Grafana | 1 | 4 VCore 8GB * 1  | 10.149.251.168 

####  调整过后的 TiDB 和 TiKV 配置

| 項目   |    |
|:----|:----|
| 操作系統     | Ubuntu 18.04|
| TiDB 版本     | TiDB-v4.0.0|
| TiDB 關鍵參數 | performance: <br> max-procs: 24 <br> |
| TiKV 關鍵參數 | readpool: <br> coprocessor: <br> high-concurrency: 8 <br> normal-concurrency: 8 <br> low-concurrency: 8 <br> storage: <br> block-cache: <br> capacity: "32GB" <br> |
 
#### sysbench
#####   TiDB Query Summary 中的 qps 与 duration 
![enter image description here](https://lh3.googleusercontent.com/e0w092l5ZtFt262-yklnQw9zIA2yVyACHBncgWCOPEMXcg0t2HtRhSP5mKUICEmv1aglG6QyOxz842Ho1TO7XWhHFe8VPB_cM50ne_wWE2_kqsimP-d8mZVSVP-yUMvpIPw302SC9BvpQPHIfLSIVxbFqGTj2yDOLlrB5A9XyKv8tklZ1VCBiaI5mlnNetf5vSahVSLhObUPgmSDuYRkKERFd-S7PBaxYvhKrjtUfWOywtBpfkEdM27wr6LSxKNP5JWw2xsGQjtnQT9FW4KpXiN8cUiZCDUxFP8qP-bmQcvxoOqV-p1M4a0GJ4IA9Gf7DQik9mKADaE8NZTOu6rHp3uerWx28Pg7p0i6htxHGSOV-a9fAqZpy6cRZ6OBwG05KclDgb4hBo8C135SpRqxXya2a6W6qQKjo1lz6AxcrDsfFujQc0-zjMYnxFlpNwk-bG5F_N6FLehB3I3H67GHPTMx1mKkc6T7g1BL_Awc8MVoffQF37vzAhlC7BvTqATcjmWG5xEGWIikk137817rKGzDEMn7LiVvX1rpYfpZgEVp9q0hSiI9WiyjbreVwUYTnDTvrUai7qORstApQSMg37qoOS78oJJdNQmECvWpMtXJY5Y9APgAFFcvu3CsFYctgTBUo30tVJfcnJom7_WRkG52hDVsWsmpUUfij4CYMb_4cFp56TT7XW2CwRLKyQ=w1020-h284-no?authuser=0)


#####   TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标 
![enter image description here](https://lh3.googleusercontent.com/cRVE1L43K4L-4ccJVX0d3j37dpWbnU3dyc1ne-vKMQzP6o297QC2gQk-oE2neJoEa16zmOz-TdV8OfL1yB0SurQ9uQn6tRNLye2HR5wOBQmBi_BIZqVngYL1op2yDjHEqTHvvIAuN_dW1p1lZViC4SVIunV5DM0Tcr-hHpTxwe9yQHIALfXwnwZuRjMlvLVoT8wPBw2kqJ5M-l70g374JMWxt4lB_lTGb7PE-QLnhICdyocNFSpiIwW-mJ-DUviu-V7cJiGGIWagKLfnKmozoTEKguK671FKNp4yGqkvX0AMK0vJ60Vty0scv9OxCq4u-xKfwxppI0LpzIjk6hPReTshTfhDDZOiNd-fIT2Ubw3uBvzSbE48FGEM2zigJK7Fw7GHSHGjV_M-GB2vKtTQDd0rspbDiSDPqXF5rWlUlGEIZcSn9b6O4cJdav14onN2b9pw8t2p7Q0kuhpWb23HFv0RWIjEjS7p6FMLphl63DFULHqWC0LQcwhlKXIvIuh2p9YqKaOt5Y73n29dO7HYktzbZEnB4whWyKhU3Jnqu5WPG4OACVVC8y6HufbEa9BlzX03mYczUfiYn0w3r-PJcsoFcIT1XQI3BRKn4bk2CNXziJ7_lD-Kcffix_4SJOfo_0-wYkrLcZWwvDa2IYYxMNv2uW8xxRhh-6IIFGBBZQ8Xo3JwJDiE8rVRP9d6_g=w516-h970-no?authuser=0)

#####   TiKV Details 面板中 grpc 的 qps 以及 duration
![enter image description here](https://lh3.googleusercontent.com/KJU0PYw__xrT1IrXGtt_cYjLI3ev2X2rt9GC-rcGrnUgnM3Euri1Tm5z2Pisko1f8NNpVv9NCb8f7npGVUsepPxPsyNoCOn7_vr6Q9PO60imbGqy_1nGSLTDec-6of4oHLpPuFfGKr-KFK4_6NFO99A0uNHvhVR3KtPRM0y8OGll0Azhk75wm4awCYoF8y3vKSsm56HBzLVlX7ePnrBZQ2R9E10adbr_IEDiX0a4gh9Zahb7CxSQSimATX4Ta4qmaFJxiRrwHpqAxqYJ37_BCJ7Y6BkowqNcizV2FzCf5FoE5NYHG2r1UAyfQ109X-cFx_IEB6-CyOJWVYy-5Uq_CPrDU1whDRJ3WfAsUxdgI4ilvZ12OyIlKRvj_Ai6reBlckJKWmWXB-Sxzf-6FH8Klq288EXvSPgF06MqFyFkVsWm7CUUr_GbEchfTXzlortnBBvWat66mcQSxZngUFvwoYv3S7KGoc6Y8QPV5FLwvhwHUAKsEaaiaqk6wCcDapT7-dw5432xOsRVp8QHuQIuIsD6TJD1D1DL6y1JX6lxT42j3O5JYlHEVC98hrjcPnQsoxioIPk979GUtqqLyZYot1uS7Ou9l8xubHhcj732nJESANWfWdOXVU8WDuxnaaaoxfQZ_inOsqftcy_uMUDThYm5RKKadSgTSAuFux6uu3S0-_dnW28KiUefZHBRRQ=w507-h655-no?authuser=0)

#### go-ycsb
測試方法：
```
./bin/go-ycsb load mysql -P workloads/workloada -p recordcount=10000 -p mysql.host=10.149.251.104 -p mysql.port=4000 --threads 16
./bin/go-ycsb run mysql -P workloads/workloada -p recordcount=10000 -p mysql.host=10.149.251.104 -p mysql.port=4000 --threads 16
```
測試結果：
```
Run finished, takes 3.164680955s READ - Takes(s): 3.2, Count: 484, OPS: 153.2, Avg(us): 2628, Min(us): 328, Max(us): 174433, 99th(us): 15000, 99.9th(us): 175000, 99.99th(us): 175000 UPDATE - Takes(s): 3.0, Count: 508, OPS: 172.0, Avg(us): 83150, Min(us): 12893, Max(us): 217830, 99th(us): 206000, 99.9th(us): 218000, 99.99th(us): 218000
```
#####   TiDB Query Summary 中的 qps 与 duration 
![enter image description here](https://lh3.googleusercontent.com/o0vW0BZoHO8X3JkMJwvsIg_NdJd9vg7qBKcZV2V-cXznTasDizVn72SMSXlViqSNYT3PT3ypA_Vm-XxPqjEO20QO_YVKlgAMfgWeyjrP13iR_IR_KeFUVlEI-aOsg3aV8Y75Yyi12iJfEpnyUgzb4_ZNcv8_41ZFnIh8q5P7N_Qq3lPVi4dYAcgvSDH80V3oGTk1AFO2EEzvjNjAQdy_CTwWOjv0tAuPTUzTe-XYZLC50XKkue_ujXMOfQJYxsww65Uf2hQaXHBO7StkLyDa6PuqB53c9xxZfi1bMgSDEfN_ZAx2UDVDzEVM-WgXBOINJVrmNStJWpyewN-pkZcl3ebD7yRL-s-Iz9YFaK1ub9AsgUO31RYpunUyZlyn-dQdGV0DARHu7Pl5Jgncq07yLj9fyHx7XNvKRAvLUJB-7GVFAki4NyvXlzaPfBD6hWUA1SILRXes6-FmFsAtLYG-er5oMJFMsvECNwcG2iYI3wfEsc6USNDIOFB9KOASGo5vG9x24y7hrQxKhg9j7d45za-3ErwpZRgSec81BhncwTuyq6kU3Mqz6IfttCi17OGI0igUrClwYfFfGAKiwKKoeu7K6nocY58TXR0VD51ERoFIBQaH6pbUgaFUNic120F8hqasJBhvpxc6mMMsM3k-pw1-iyIH7it-8OCq9SbPi6B9NoLCxCgKMWjoCxR51w=w1013-h256-no?authuser=0)
#####   TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标 
![enter image description here](https://lh3.googleusercontent.com/uc8ewFe8cBwk_kA79z4QZW5vwFo1qztoFAReA6DL-knz2ufN-4Ujxa3miKTIXSouxVSMUCVFTpo96-69ySlWEE1V08Z3CLwpPC_GutdFQ9TVOe2WEmu6dQk-LXdmJFeBQXoGdwdTsbzb3bKY7fFZOv7dnwnjeQDiJzcPqli1Cdk5aDLSGh8LpyPgq13Krqz23e76mkcn-WEyd5rYCR3e3Gk5Qm9TL8ocFnULQAjveLBt--T3ZIxcGIbH40KcIoaWbzDE0KcUr_hHrRGLbM2L_Qgshv1L6jXMVo78pARe698IPIzpQH4sZ4A3GvY1VLVjzrdFmQMYTqV_R-vJKeSdShXxZCKZnEck4Pb0umdeOaVbvbxY27C18nZwCQ3v19hK7vIaDXBw-Dju_d4euZje4PZlMs0aOxzb4TxgVuEzSWTKfUbQrM26KjoW1NObPbj2whWSsaPndTwjMZ48V6Y-e5IDUjt2S1EGDlFQ3NWH4hjUv1Vug0Og0_89Cbe1PD8VOyJAH0672gu8ON5c8lOCei5sgzIBjZ2fl0kZq8GGPkl3EfSvGvcGY6iaYKIx_AV7itIpKJIQVgZz6twLRzwKS9ROlFTcPU9G8SNzpQxYhnRM7_rJiAT35W4Uwv6-zfFBvtx4CJ2tKEQgRroiPvUuO2LaSfR6ljNLFOqqBClg3dqlbkC89c0Ud1i6a6ec8g=w499-h965-no?authuser=0)
#####   TiKV Details 面板中 grpc 的 qps 以及 duration
![enter image description here](https://lh3.googleusercontent.com/Rc_ibGfEB76uL3AaaR0aVaoIZFVnybLdZRUI3ydPNzW1QAVYuEeNCReMjdYXXUUzjYGvQL3-na-5vt9krRU2V0spL_n23GINEDanOMZHdDe2eaSIVGaQyMdLYAVtxHuY1QRqRcisiBfLr-PUaIEliXVBmo6imVfYsTk83X70DbWAfBBZGAuCsQQFxt8raQ5_jFp1APdfSVONNWmjWgToKZBw6uuFvzQIbFz9JuMmlCgIdyJcQw_MDmxky7C78hUCibol3jF_QXykudOre-daDTJf4iMzW3Lv-OhNBrsQQUp9oWTL-MNIx29uIjBBc41SGJuLKZQfgv_IeiT6J6wLqOXhnc5f9gf0klN-3H3MCW3kmm-UE223np9cCDcaCUWg1rEG6cxokotmHRXPRggz-UY-azX_BDiS5PoV9CmNnUUOrhDu13qBsFRdoLnIlKjwtak-KjscIFK0KzxN2G-7uvljTBt3cTKhq1lFqezYKidALta4o3O7S5BlRgKr03H8RC4ZQ9zwS03lXgrmFxa227j0r1Krg-qehsHU9ecVMUN-AJFGzHRaGAoZ2wA2CTQo2koJz8378J1GNjxtM2F_5FqB52FRw0cQ8wFFtK80zi8LXD-CIXGjki4D5HqW4VZpatXhMTGFHiZPKf-MQtMvnArXQnSPsTtb92FnQuRBrvaBR-AjlcAfnfbNcW2YlQ=w499-h647-no?authuser=0)



#### go-tpc
測試方法：
```
資料準備
aiden@k8s-01:~$ go-tpc tpcc prepare --warehouses 10 -D tpcc -H 10.149.251.104 -P 4000 -T 10
測試
aiden@k8s-01:~$ go-tpc tpcc run --warehouses 10 -D tpcc -H 10.149.251.104 -P 4000 -T 10 --time 5m
```

測試結果：
```
```
#####   TiDB Query Summary 中的 qps 与 duration 
![enter image description here](https://lh3.googleusercontent.com/RklgNKcnOVa8HwyOv0MierDj2H4kwPiyFWukI3E1nTGymAYGoLtsaLVs8a78B3Awur5jjfdwsc8IVbkT5gy9voNstCBkYIy0b1pHmUkOWFLBmetw81nSwtbSs6Bwu1k6LaH7JyMos5clQI-46hOw9nSTZCMhkdit0O4vVFbjvTgR8iMMci3ZkWEAvSAFZoNZu4LChNbrcN3ZBwjhNbhRAOAoCV2wj370i0p4dwvNo-posbM3CxbUuEXj_yvUOcY3fOW_MNpx8Y4_1qk1xK7h19cAZ3-euq4NYFP1b7teWEgxeQpDulsyG9QhwoXhIcsqeNuI7rGUtQVDpn0AjBOcDxBAkNGX_G1LCb38bso-LNh59LxP8I-5e6--NsPli_Ysdz14_UTwBtw9Gbo8u_pFx9eHpAbDlQgB72Xs6QoT6Rl84RCWEFHcWVf4SiJPshbKIzMLkpLQbpYHOHns5u_cac4zo1CXV6TDLc4sxgequEO5L2ptgWa1YX84Nz-KgaZYFn71ucvzcroamEbsymxvWSKKsvrUBtTOCluZk2KH7VElgbeo6GWj23Pvf0dECneIlIhfxF6SqsTrChUgpWjjxeLC9m-6ixCFMqZJyGUB4lnXgfISl7nCSb0NxsAr4SEDZLY9ivaIC1WzDAwX9KTcbannVtxR3i-ll95QsBB8d9BTXUQntCqXwjpSNq3RBg=w1008-h276-no?authuser=0)
#####   TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标 
![enter image description here](https://lh3.googleusercontent.com/6qZ0v3u1ileAtioy4ZmWkuf-5rPBR32bqMYsZ1zwqWrSsIGnU9iaK2CHhrY7HgtUFo8I0Q3bWPmgkMTYRg0dS0Yv5uxfesL9XT1hVXOCjUJwRvPpp-m7xgUNQThhKeKnjPWZBgauNllvzok2IlnF7KYebVoTds2Lb03baNGqRWclAqudrUo8jkeZMY1djS-ib5FjU0zqsmesL6e6fb-mAiXrHbSx88N_XrPM9f_9YQsBRWTbR84jPO1IVi7EUcemKvOmuH-6dQdyN9SzBB6tbp5RuyDENMDNrw40YDsVfW-pz4Ak-NvPi4_Fpzp5-pm2lzylsrZsV8AF1hjWIyqejaV1eQXJFG0rWCy2ygKTXAz1Pgv4OA4sVkBzdaiB6K8Mt_wh8cPiMySVwd-QfKPaMozjfbL9uFSHhf9B62GvMyO9gfGvJcW05zf6UP07io3ki6cManDiipjeW64--GIw-tiBAephsFaGKSgFPc7IJSfzXU8XF0ZyOfxExuDKQ34Wa105YtzvnRrNQIzJ_oS8suxf-lcANQW4HbIrbpbgaQVXNTwQI2Yf_8pdxLhb7pCEXCQXK7Vnyt3oMRsump9vnpd5wlvDjcjbym1lCsebKtGeB29m8zQa21cpcBONE1NKyHAxnBUCSdxzts-SZ1KWbdSeR7NlPfO_8ozrfE8JS_VeJnpMe9_bZiXs6rQZLA=w502-h964-no?authuser=0)
#####   TiKV Details 面板中 grpc 的 qps 以及 duration
![enter image description here](https://lh3.googleusercontent.com/fp-PYWa95KOXwRj3IWJluYZ99k5K_LdQ4mhJg0iSff2Ro8K2R7t6EJapfuD63QSPJIhpf66qpmKHWidCeerobYtZOKqNTU4EWI-HiNMG1SEJVIfUVylZF28EfjK8Q08A1fn2lvEAwUA8F2kLElkgnFFQ60GdmyEEp5iD7ZY4AbfoNsj9EqCD5UHnYLgYpcQqEGqXF7hkYdp9UM8sXW-X-HvcOday_OPXLGKYPiR_Pkc2vx9jGkpOnpbEIb63nd7zbUnd0K2bGD1TUoVY-hXGMbkQLyX4yeFHgRlrr5x_T_14DuH59jRtFL6Cieu_tupLnTmYqAAQl1GOlNhzxV1ogr6EiaSH4-I71SbvsSxNUy86UTN8tzkBAWe-5hmyCTBq_9SFPnEdI2O5uRrJVgzl9_sN7iMXQr-eAwO7BvJsuY-ExiUKN8I_H1bfjjBGEbCNkRSynnfs8yOhTY8WzVhVsgyqLHRvp9b0kvTtz7hXymYGLbhD__N3D7_cwwDCz47RPW6E48j2UOfgS_IsrAgSJ-BDh265NgT3Z2LpGE4VVSMhF_lrBG2we1nbrg53iJWljccRIvFq5EdrhWZDSWuSnszPkoaYYX9wW7JGaXKwpQmp0i0bTsyyUC_bOe5JIJ9LKYw76bcNr1-pEj6uS9BbNjLkehYizBtVzHJW1ufsytyOULb0XG6Y2281kr_lxg=w508-h645-no?authuser=0)



## 延伸學習

## 課前學習資料筆記

 1. 使用 TiUP 部署 TiDB 集群(https://docs.pingcap.com/zh/tidb/stable/production-deployment-using-tiup)
- 文件中有說明不同的拓譜架構，科普一下幾個提到元件：
	- **TiFlash**：列存副本通过 Raft Learner 协议异步复制，但是在读取的时候通过 Raft 校对索引配合 MVCC 的方式获得 Snapshot Isolation 的一致性隔离级别
	- **TiCDC**：[TiCDC](https://github.com/pingcap/ticdc) 是一款通过拉取 TiKV 变更日志实现的 TiDB 增量数据同步工具，具有将数据还原到与上游任意 TSO 一致状态的能力，同时提供[开放数据协议](https://docs.pingcap.com/zh/tidb/dev/ticdc-open-protocol) (TiCDC Open Protocol)，支持其他系统订阅数据变更。
	- **TiDB Binlog**：TiDB Binlog 是一个用于收集 TiDB 的 binlog，并提供准实时备份和同步功能的商业工具。
	- **TiSpark**：TiSpark 是 PingCAP 为解决用户复杂 OLAP 需求而推出的产品。它借助 Spark 平台，同时融合 TiKV 分布式集群的优势，和 TiDB 一起为用户一站式解决 HTAP (Hybrid Transactional/Analytical Processing) 的需求。

 2. TiKV 线程池优化(https://github.com/pingcap-incubator/tidb-in-action/blob/master/session4/chapter8/threadpool-optimize.md)
 3. PD Dashboard 说明(https://docs.pingcap.com/zh/tidb/stable/dashboard-intro)
 4. TPCC 背景知识](https://github.com/pingcap-incubator/tidb-in-action/blob/master/session4/chapter3/tpc-c.md)
 5. ycsb,sysbench](https://github.com/pingcap-incubator/tidb-in-action/blob/master/session4/chapter3/sysbench.md)

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTIyMzkwMTMwMCwtNTk3MTU0MzQ2LC01Nj
QxMTg3MjAsLTE1ODEwMjQ1NjYsLTExMDgzMzk5NzIsMzM0Nzc5
MTczLDIxMTUxNjM2MTEsLTU2MTQyMDEyNCwtNTE0Mzg2NjcxLD
E0MjAzMDQ3MzEsLTEwNjcwMDk5NzcsODAzMzI0MjYyLC0xMDk2
NjMyNjc5LC0xNDE4Nzg5Mjg0LDIxNDQ0MTEzMjMsLTE3NjE5MT
kxODEsMTQyMTM3MDY2OCwtMTY2NTY1MDU0NSw5MTkzNjY1ODQs
LTM4Nzc3NjQ1Ml19
-->