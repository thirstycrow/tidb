### Download the projects

```sh
projects=~/Projects
data=~/Data/tidb_dev

cd $projects

git clone https://github.com/pingcap/pd.git
git clone https://github.com/tikv/tikv.git
git clone https://github.com/pingcap/tidb.git
```

### Build the projects
cd $projects/pd && make
cd $projects/tikv && make build
cd $projects/tidb && make

### Start the servers
```
cd $data

# 1 PD
$projects/pd/bin/pd-server \
    --name=pd \
    --data-dir=pd \
    --client-urls="http://127.0.0.1:2379" \
    --peer-urls="http://127.0.0.1:2380" \
    --initial-cluster="pd=http://127.0.0.1:2380" \
    --log-file=pd.log &

# 3 TiKV
tikv_cfg=debug

$projects/tikv/target/$tikv_cfg/tikv-server \
    --pd-endpoints="127.0.0.1:2379" \
    --addr="127.0.0.1:20160" \
    --status-addr="127.0.0.1:20181" \
    --data-dir=tikv1 \
    --log-file=tikv1.log &

$projects/tikv/target/$tikv_cfg/tikv-server \
    --pd-endpoints="127.0.0.1:2379" \
    --addr="127.0.0.1:20161" \
    --status-addr="127.0.0.1:20182" \
    --data-dir=tikv2 \
    --log-file=tikv2.log &

$projects/tikv/target/$tikv_cfg/tikv-server \
    --pd-endpoints="127.0.0.1:2379" \
    --addr="127.0.0.1:20162" \
    --status-addr="127.0.0.1:20182" \
    --data-dir=tikv3 \
    --log-file=tikv3.log &

# and 1 TiDB
cat > tidb.toml <<EOF
[prepared-plan-cache]
enabled = true
mem-quota-query = $[2<<30]
EOF

$projects/tidb/bin/tidb-server --store=tikv \
    --path="127.0.0.1:2379" \
    --config=tidb.toml \
    --log-file=tidb.log \
    -L=info &
```

### Locate where transactions get started in the source code
1. Grep some transaction related keyword such as 'rollback' to find where TiDB abstracts transactions:
```sh
$ grep -P ' Rollback\('
kv/interface_mock_test.go:func (t *mockTxn) Rollback() error {
session/txn.go:func (st *TxnState) Rollback() error {
store/mockstore/mocktikv/mvcc_leveldb.go:func (mvcc *MVCCLevelDB) Rollback(keys [][]byte, startTS uint64) error {
store/tikv/txn.go:func (txn *tikvTxn) Rollback() error {
```
Obviously it's in `session/txn.go`. After reading the source code, we find that `TxnState` is the primary abstraction of transaction in TiDB. And it's the `session.txn` member that holds the transaction state. So I'm going to find where the `session.txn` member is assigned or initialized.

2. There are 3 metheds where `sessoin.txn` gets assigned: `NewTxn`, `InitTxnWithStartTS` and `ExecRestrictedSQLWithSnapshot`. In the first 2 methods, `session.txn` is assigned the results of the corresponding `store.Begin...` methods, so if there were not an error, a transaction should have been stared. So we add a log statement after the error checks.

3. As in the `ExecRestrictedSQLWithSnapshot` method, `session.txn` is assigned the result of `s.Txn(false)`. Follow the source code of this method, we find that fact that transactions are lazy initialized, so the log statement should be where the invocation of `s.txn.changePendingToValid()` returns successfully.

4. The final changes can be found here: https://github.com/thirstycrow/tidb/commit/46d5deb2ae1dd6f76797c16f487c770aedd79e10

5. After rebuild and restart tidb-server, I did some tests with the mysql client, and it turned out the `NewTxn` method is get invoked each time a DDL statement is executed, and `s.Txn(true)` each time a DML statement is executed. However the log in `InitTxnWithStartTS` never appeared during the testing.

6. Even without any operations, the log in `s.Txn(true)` gets printed several times in each second. I guess there are some tasks gets scheduled in every seconds in the background.
