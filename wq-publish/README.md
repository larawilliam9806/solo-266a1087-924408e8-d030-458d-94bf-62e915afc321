# 水质实验室 · 批次修订发布台

供水质实验室数据员使用的批次修订发布页（Flask + SQLite）。浏览器上传仪器导出的
CSV、填写批次号，系统逐行校验后预览**新增 / 修改 / 撤回（删除）**差异，确认后
**整批替换**下游采集程序读取的本地 CSV，并在网页登记当前版本与行数。

## 功能与规则

- 上传 CSV 必须包含三列（表头兼容中英文）：`样本号`、`检测项目`、`浓度(mg/L)`
  （亦接受 `sample_no / analyte / concentration` 等常见别名，UTF-8、支持 BOM）。
- 逐行业务校验，错误带行号提示：
  - 同一样本同一检测项目不得重复；
  - 浓度必须是数字且 **≥ 0**（拒绝负数、空值、NaN、Inf、非数字）；
  - 样本号、检测项目不能为空，列数必须与表头一致。
- 上传内容代表修订后的**完整一批**：当前版本中存在、但本次 CSV 未包含的行，
  在预览中列为「撤回（删除）」，确认后该检测结果即被撤回。
- 预览页分类展示新增、修改（旧值→新值）、撤回、不变四类差异，确认后才发布。
- 首页登记当前版本号、批次号、行数、发布时间（UTC）与内容 SHA256，并提供
  已发布 CSV 的在线查看入口和事件日志（含每次恢复记录）。

## 本地启动

环境要求：Python 3.10+（仅依赖 Flask；waitress 用于容器/正式部署）。

```bash
cd wq-publish
./run.sh                 # 自动创建 .venv、安装依赖，监听 http://127.0.0.1:5000
```

手动方式：

```bash
python3 -m venv --without-pip .venv   # 若系统带 ensurepip，直接 python3 -m venv .venv
# 按你的环境引导 pip 后：
.venv/bin/pip install -r requirements.txt
WQ_DATA_DIR=$PWD/data WQ_DOWNSTREAM_DIR=$PWD/downstream \
  .venv/bin/flask --app wsgi:create_app run --host 127.0.0.1 --port 5000
```

## Docker Compose 启动（双实例演示滚动启动）

```bash
cd wq-publish
docker compose up --build -d
```

编排包含 `web1`（http://localhost:5001）与 `web2`（http://localhost:5002）两个
服务进程，挂载**同一组命名卷**（`wq-data` / `wq-downstream`），等价于滚动重启时
新旧进程并存共用数据目录的场景：

- 在 5001 上传并发布一个批次，5002 刷新即可看到同一登记版本与事件日志；
- 发布过程中滚动重启另一台：`docker compose up -d --no-deps --force-recreate web2`，
  已提交成功的发布不会被启动恢复回退，在途发布的临时文件也不会被误删；
- 下游采集程序读取卷内 `published_results.csv`（可用
  `docker compose exec web1 cat /app/downstream/published_results.csv` 查看）。

停止并清空数据：`docker compose down -v`。

环境变量（可选）：

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `WQ_DATA_DIR` | `./data` | SQLite 数据库、flask 密钥、隔离区存放目录 |
| `WQ_DOWNSTREAM_DIR` | `./downstream` | 下游采集程序读取目录，正式文件固定名为 `published_results.csv` |

下游采集程序只需固定读取 `WQ_DOWNSTREAM_DIR/published_results.csv`。

## 操作流程

1. 打开首页，填写批次号并选择仪器导出的 CSV，点击「校验并预览增删改」。
2. 校验失败：回到首页逐条显示带行号的错误，修正后重新上传，不产生任何数据变更。
3. 校验通过：进入预览页核对新增/修改/撤回差异，可「放弃本次修订」或
   「确认无误，整批替换并发布」。
4. 发布成功后回到首页，登记版本号递增、行数更新，下游文件已原子替换。

## 数据一致性设计（如何保证要求）

**下游只会读到完整旧版或新版**

- 发布时先把新 CSV 写入下游目录内的同目录临时文件（`.tmp-publish-*.csv`），
  `flush + fsync` 后调用 `os.replace` 原子改名覆盖正式文件，并 fsync 目录。
  改名瞬间下游要么看到旧文件、要么看到完整新文件，不会读到半写内容；
  临时文件以 `.` 开头且与正式文件名不同，不会被按固定文件名读取的采集程序读到。

**文件与登记的事务关系、崩溃自动恢复**

- SQLite（`data/wq_publish.db`）是版本登记与全版本结果的唯一真相源，
  发布在 `BEGIN IMMEDIATE` 事务内完成「原子换文件 → 写新版本结果 → 更新登记表」。
- **恢复与发布互斥（多进程滚动启动安全）**：`Storage.recover()` 的整个
  「读登记 → 核对/重建文件 → 清理残留」过程同样在 `BEGIN IMMEDIATE` 写事务内
  执行，与发布共用同一把 SQLite 写锁。滚动启动共用数据目录的多个进程时，
  恢复要么看到发布提交前的旧状态、要么看到提交后的新状态，绝不会把在途或
  刚提交成功的发布回退，也不会与发布中的文件替换交错。
- 每次启动（`Storage.recover()`）自动核对并修复：
  1. 清理崩溃遗留的 `.tmp-publish-*` 临时文件；**修改时间新于 60 秒的临时文件
     视为其他进程可能正在发布，跳过不删**（互斥锁已保证正常情况下看不到在途
     文件，宽限期是对异常写入者的兜底）；
  2. 正式文件缺失或 SHA256 与登记不符时，以数据库登记为准原子重建；
  3. 「文件已改名但事务未提交」的现场，重启后回滚为登记中的完整版本；
  4. 下游目录中未登记的游离文件移入 `data/quarantine/` 隔离（不直接删除；
     正式文件与 `.tmp-publish-*` 临时文件不在隔离范围）；
  5. 超过 24 小时未确认的预览会话自动作废。
  所有恢复动作写入 `event_log` 并展示在首页。
- 多进程共用数据目录时，Flask 会话密钥以 `O_EXCL` 竞争创建、其余进程读取
  同一把密钥，避免滚动启动期间会话（闪存消息）跨进程失效。

**两个页面从同一版本发起修订，只允许一个成功**

- 每次预览生成带随机 token 的修订会话，记录发起时的基线版本 `base_version`。
- 确认发布时在写事务内重新比对当前版本：基线已被其他发布超越时，本会话
  标记为 `obsolete` 并返回 409 过期预览页（仍展示旧基线差异供核对），
  明确要求重新预览，绝不可能覆盖先发布者；重新预览后即可正常发布。
- 同一会话重复发布、已取消/已发布/已过期会话再次确认，一律拒绝。

## 自测

```bash
.venv/bin/python -I tests/selftest.py
```

覆盖 55 项断言：CSV 各类非法输入、上传→预览→发布全链路、增改删分类与撤回语义、
同版本双页面并发冲突（409 + obsolete + 重新预览成功）、文件缺失/损坏/临时文件/
游离文件/改名未提交五类崩溃恢复、会话取消与重复发布防护，以及滚动启动竞态
（发布「改名后、提交前」窗口期内恢复不得回退文件；在途临时文件不得被清理；
交错启动后登记与正式 CSV 一致）。

`examples/` 下提供 `batch_v1.csv`、`batch_v2_revision.csv`（含增改删）与
`batch_invalid.csv`（含重复、负数、空值）可直接在页面上演练。

## 部署提示与已知限制

- `flask run` 为开发服务器；正式环境用容器（见 Dockerfile / docker-compose.yml）
  或 `waitress-serve --call --host 0.0.0.0 --port 8000 wsgi:create_app`。
  并发正确性依赖 SQLite 的 `BEGIN IMMEDIATE` 写锁与同目录 `os.replace`，
  多进程/多实例部署时**必须共享同一个** `WQ_DATA_DIR` 与 `WQ_DOWNSTREAM_DIR`，
  且二者需位于支持 POSIX 文件语义的本地/卷文件系统上（NFS 等网络文件系统的
  SQLite 锁可靠性不在保障范围内）。
- 恢复与发布互斥使用 SQLite 写锁，等待上限为 `busy_timeout`（15 秒）；
  极端情况下若启动恢复等锁超时，进程会启动失败，由编排层（如
  `restart: unless-stopped`）重试即可。
- 全量结果按版本保留在 `results` 表中，当前未提供历史版本回滚的页面入口
  （数据具备，可后续增加）；历史 CSV 文件本身只保留最新一版。
- 上传上限 16MB；认证、权限控制未包含在本版本内，建议仅监听内网或前置反向代理
  加访问控制。
