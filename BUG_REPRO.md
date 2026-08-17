# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

巡护 App 的离线补传功能对不上账，麻烦帮我修一下。

复现：
1. 巡护员在无信号区暂存了 2 个打卡点（w1 在 08:10、w2 在 08:20，各带自己的 idempotency_key）
2. 回到有网区，App 一次性 POST /api/checkins/batch 把这 2 条推上来
3. 接口返回 200，但响应体里有 4 条记录：
   前 2 条是 {"id":"","shift_id":"","officer_id":"","waypoint_id":"","timestamp":"0001-01-01T00:00:00Z",...} 这种空记录，
   真正的 w1、w2 排在后面

App 是按这个响应的条数给巡护员显示「本次补传 N 个轨迹点」并据此累计当班打卡次数的，
现在推 2 条显示补传 4 个，打卡计次直接翻倍；空记录在轨迹列表里也是几行空白。
查 GET 出来的当班轨迹条数是对的，问题只在补传接口的返回值上。

期望：批量补传的返回值就是这次推上来的那些记录本身 —— 条数一致、每条都是真实记录、按原始时间戳从早到晚排列；
重复推同一批时仍然按幂等键收敛，不重复计次。

请修复，修复后 go test ./... -count=1 要全绿。

## 含 Bug 版本

- 仓库：11DingKing/go-ecfc00-t012-01
- 仓库地址：https://github.com/11DingKing/go-ecfc00-t012-01.git
- parent SHA：4ccbbf3a8ec890b94cbab2d48e760d95cfc8f361

## 复现步骤

```bash
git clone -- https://github.com/11DingKing/go-ecfc00-t012-01.git bug-repro
cd bug-repro
git checkout --detach 4ccbbf3a8ec890b94cbab2d48e760d95cfc8f361
go test ./internal/app/ -run "^TestCheckInOfflineReplayAndIdempotency$|^TestCheckInSingleIdempotency$" -count=1 -v
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/app/ -run "^TestCheckInOfflineReplayAndIdempotency$|^TestCheckInSingleIdempotency$" -count=1 -v
=== RUN   TestCheckInOfflineReplayAndIdempotency
    patrol_service_test.go:26: expected 3 results, got 6
--- FAIL: TestCheckInOfflineReplayAndIdempotency (0.00s)
=== RUN   TestCheckInSingleIdempotency
--- PASS: TestCheckInSingleIdempotency (0.00s)
FAIL
FAIL	patrol-platform/internal/app	0.072s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/app/ -run "^TestCheckInOfflineReplayAndIdempotency$|^TestCheckInSingleIdempotency$" -count=1 -v
=== RUN   TestCheckInOfflineReplayAndIdempotency
    patrol_service_test.go:26: expected 3 results, got 6
--- FAIL: TestCheckInOfflineReplayAndIdempotency (0.00s)
=== RUN   TestCheckInSingleIdempotency
--- PASS: TestCheckInSingleIdempotency (0.00s)
FAIL
FAIL	patrol-platform/internal/app	0.001s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

定向复现命令在修复前失败、修复后通过（exit 0）。
POST /api/checkins/batch 的返回条数等于推入条数，每条都是真实记录（无空 id / 空 waypoint_id），且按原始时间戳升序。
重复推同一批仍按幂等键收敛，当班轨迹不重复计次。
go test ./... -count=1 -timeout=300s 与 go test -race ./... -count=1 -timeout=600s 全绿。
go build ./...、go vet ./... 通过，gofmt -l . 无输出。
只修改生产代码；不得新增、删除、改写或跳过任何 *_test.go 中的测试与断言。
linux/amd64 与 linux/arm64 两个架构上结果一致。
