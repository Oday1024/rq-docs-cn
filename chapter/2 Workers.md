# Workers

worker是一个Python进程，通常在后台运行，并且仅作为子进程（work horse）存在，以执行你不希望在Web进程内执行的冗长或阻塞任务。

## 启动workers

开始计算工作，只需从项目目录的根目录启动一个worker：

```sh
$ rq worker high default low
*** Listening for work on high, default, low
Got send_newsletter('me@nvie.com') from default
Job ended normally without result
*** Listening for work on high, default, low
...
```

Workers将在无限循环中读取给定队列中的任务（顺序很重要），当所有任务完成后等待新工作到达。

每个worker一次只能处理一份工作。在一个worker中，没有并发处理。如果你想同时执行任务，只需启动更多workers。

你可以使用[Supervisor](https://python-rq.org/patterns/supervisor/)或 [systemd](https://python-rq.org/patterns/systemd/)等流程管理器在生产中运行RQ worker。

### Burst模式

默认情况下，workers将立即开始工作，在他们完成所有任务时阻止并等待新任务。workers也可以以***突发模式***启动以完成所有当前可用的任务，在所有给定队列为空后立即退出。

```sh
$ rq worker --burst high default low
*** Listening for work on high, default, low
Got send_newsletter('me@nvie.com') from default
Job ended normally without result
No more work, burst finished.
Registering death.
```

这对于需要定期处理的批处理任务非常有用，或者只是在高峰期临时扩展workers。

### worker参数

除此`--burst`之外，`rq worker`还接受以下参数：

- `--url`或`-u`：描述Redis连接详细信息的URL（例如`rq worker --url redis://:secrets@example.com:1234/9`）
- `--path`或`-P`：支持多个导入路径（例如`rq worker --path foo --path bar`）
- `--config`或`-c`：包含RQ设置的模块的路径。
- `--results-ttl`：任务结果将保留此秒数（默认为500）。
- `--worker-class`或`-w`：使用的RQ Worker类（例如`rq worker --worker-class 'foo.bar.MyWorker'`）
- `--job-class`或`-j`：使用的RQ Job类。
- `--queue-class`：使用的RQ队列类。
- `--connection-class`：使用的Redis连接类，默认为`redis.StrictRedis`。
- `--log-format`：工作日志的格式，默认为 `'%(asctime)s %(message)s'`
- `--date-format`：工作日志的日期时间格式，默认为 `'%H:%M:%S'`
- `--disable-job-desc-logging`：关闭任务说明日志记录。
- `--max-jobs`：执行的最大任务数。

## worker内部

### worker生命周期

worker的生命周期包括几个阶段：

1. **启动**：加载Python环境。
2. **注册**：worker将自己注册到系统，以便知道该worker。
3. **监听**：从任何给定的Redis队列中获取任务。如果所有队列都为空且worker以突发模式运行，则立即退出。否则，等待任务到来。
4. **准备执行任务**：worker告诉系统它将通过设置其状态为`busy`，并在`StartedJobRegistry`中注册任务来开始工作。
5. **fork子进程**：复制一个子进程（work horse）以在故障安全上下文中执行实际任务。
6. **处理工作**：在子进程（work horse）中执行实际的工作。
7. **清理任务执行**。worker将其状态设置为`idle`，并设置任务及其结果的过期时间（基于`result_ttl`）。任务成功后，会从`StartedJobRegistry`中删除任务并且添加到`FailedJobRegistry`中，任务失败后，添加到`FailedJobRegistry`中。
8. **循环**：从第3步开始重复。

### 性能说明

基本上`rq worker`shell脚本是一个简单的fetch-fork-execute循环。当你的很多工作都进行冗长的设置，或者它们都依赖于同一组模块时，每次运行任务时都要支付开销（因为你在fork之后进行了导入）。这很干净，因为RQ不会以这种方式泄漏内存，但也会很慢。

可用于提高这类任务的吞吐量性能的模式可以是在fork之前导入必要的模块。无法告诉RQ worker为你执行此设置，但你可以在开始工作循环之前自己完成此操作。

为此，请提供你自己的任务脚本（而不是使用`rq worker`）。一个简单的实现示例：

```python
#!/usr/bin/env python
import sys
from rq import Connection, Worker

# 预先导入模块
import library_that_you_want_preloaded

# 提供队列名称，类似于rq worker
with Connection():
    qs = sys.argv[1:] or ['default']

    w = Worker(qs)
    w.work()
```

### worker名称

worker以其名称注册到系统，这些名称在实例化期间随机生成（请参阅[监控](https://python-rq.org/docs/monitoring/)）。要覆盖此默认值，请在启动worker时指定名称，或使用`--name`客户端选项。

```python
from redis import Redis
from rq import Queue, Worker

redis = Redis()
queue = Queue('queue_name')

# Start a worker with a custom name
worker = Worker([queue], connection=redis, name='foo')
```

### 检索worker信息

*在版本0.10.0中更新。*

`Worker`实例将其运行时信息存储在Redis中。以下是检索它们的方法：

```python
from redis import Redis
from rq import Queue, Worker

# Returns all workers registered in this connection
redis = Redis()
workers = Worker.all(connection=redis)

# Returns all workers in this queue (new in version 0.10.0)
queue = Queue('queue_name')
workers = Worker.all(queue=queue)
worker = workers[0]
print(worker.name)
```

除`worker.name`之外，worker人还具有以下属性：

- `hostname` - 运行此工作程序的主机
- `pid` - worker的进程ID
- `queues` - worker正在侦听工作的队列
- `state`-可能的状态是`suspended`，`started`，`busy`和`idle`
- `current_job` - 正在执行的任务（如果有的话）
- `last_heartbeat` - worker最后的更新时间
- `birth_date` - worker实例化的时间
- `successful_job_count` - 成功完成的任务数量
- `failed_job_count` - 处理的失败任务数
- `total_working_time` - 执行任务所花费的时间，以秒为单位

*版本0.10.0中的新功能。*

如果您只想知道用于监控目的的worker数量， `Worker.count()`那么性能要高得多。

```python
from redis import Redis
from rq import Worker

redis = Redis()

# 计算redis中的worker连接数
workers = Worker.count(connection=redis)

# 计算指定队列中的worker数
queue = Queue('queue_name', connection=redis)
workers = Worker.all(queue=queue)
```

### worker统计

*0.9.0版中的新功能。*

如果要检查队列的利用率，`Worker`实例会存储一些有用的信息：

```python
from rq.worker import Worker
worker = Worker.find_by_key('rq:worker:name')

worker.successful_job_count  # 成功的任务书
worker.failed_job_count # 失败的任务书
worker.total_working_time  # 执行任务所花费的时间 (单位：秒)
```

## 更好的worker处理标题

安装第三方软件包`setproctitle`后，工作进程将具有更好的标题（由ps和top等系统工具显示）：

```
pip install setproctitle
```

## 取下worker

如果工作人员在任何时候接收到`SIGINT`（通过Ctrl + C）或`SIGTERM`（通过 `kill`），worker等待当前正在运行的任务完成，停止工作循环并优雅地注册自己的死亡。

如果，在删除阶段，再次收到`SIGINT`或`SIGTERM`，worker将强行终止子进程（发送它`SIGKILL`），但仍将尝试注册自己的死亡。

## 使用配置文件

如果您想通过配置文件而不是通过命令行参数进行配置`rq worker`，可以通过创建Python文件来完成此操作 `settings.py`：

```python
REDIS_URL = 'redis://localhost:6379/1'

# 你也可以指定使用的redis数据库
# REDIS_HOST = 'redis.example.com'
# REDIS_PORT = 6380
# REDIS_DB = 3
# REDIS_PASSWORD = 'very secret'

# 监听的队列
QUEUES = ['high', 'default', 'low']

# 如果您使用Sentry收集运行时异常，则可以使用此方法去配置RQ
# The 'sync+' prefix is required for raven: https://github.com/nvie/rq/issues/350#issuecomment-43592410
SENTRY_DSN = 'sync+http://public:secret@example.com/1'

# 自定义worker名称
# NAME = 'worker-1024'
```

上面的示例展示了当前支持的所有选项。

*注意：* `QUEUES` *和* `REDIS_PASSWORD` *设置是0.3.3以后的新特性。*

指定从哪个模块读取设置，请使用以下`-c`选项：

```sh
$ rq worker -c settings
```

## 自定义Worker类

有时您想要自定义worker的行为。到目前为止，一些更常见的请求是：

1. 在运行任务之前管理数据库连接。
2. 使用不需要的任务执行模型`os.fork`。
3. 能够使用不同的并发模型，如 `multiprocessing`或`gevent`。

您可以使用该`-w`选项指定要使用的其他工作类：

```sh
$ rq worker -w 'path.to.GeventWorker'
```

## 自定义任务和队列类

你可以告诉worker使用自定义类来处理任务和队列`--job-class`或`--queue-class`。

```sh
$ rq worker --job-class 'custom.JobClass' --queue-class 'custom.QueueClass'
```

在添加队列任务时不要忘记使用相同的类。

例如：

```python
from rq import Queue
from rq.job import Job

class CustomJob(Job):
    pass

class CustomQueue(Queue):
    job_class = CustomJob

queue = CustomQueue('default', connection=redis_conn)
queue.enqueue(some_func)
```

## 自定义DeathPenalty类

当任务超时时，worker将尝试使用提供的方法`death_penalty_class`（默认值: `UnixSignalDeathPenalty`）将其终止。如果你希望尝试以特定于应用程序或“更清洁”的方式杀死任务，则可以覆盖此项。

DeathPenalty类使用以下参数构造 `BaseDeathPenalty(timeout, JobTimeoutException, job_id=job.id)`

## 自定义异常处理

如果你需要针对不同类型的任务以不同方式处理错误，或者只是想要自定义RQ的默认错误处理行为，使用`--exception-handler`选项运行`rq worker`：

```sh
$ rq worker --exception-handler 'path.to.my.ErrorHandler'

# Multiple exception handlers is also supported
$ rq worker --exception-handler 'path.to.my.ErrorHandler' --exception-handler 'another.ErrorHandler'
```

如果要禁用RQ的默认异常处理程序，请使用`--disable-default-exception-handler`选项：

```python
$  rq worker --exception-handler 'path.to.my.ErrorHandler' --disable-default-exception-handler
```