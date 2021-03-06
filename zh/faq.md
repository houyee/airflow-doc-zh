# 常见问题

> 译者：[@zhongjiajie](https://github.com/zhongjiajie)

## 为什么我的任务没有被调度？

您的任务没有被调度的原因有很多。以下是一些常见原因：

* 您的脚本是否“编译”，Airflow 引擎是否可以解析它并找到您的 DAG 对象。如果要对此进行测试，您可以运行`airflow list_dags`并确认您的 DAG 显示在列表中。 您还可以运行`airflow list_tasks foo_dag_id --tree`并确认您的任务按预期显示在列表中。 如果您使用 CeleryExecutor，您需要确认上面的命令在 scheduler 和 worker 中能如期工作。
* DAG 的文件是否在内容的某处有字符串 “airflow” 和 “DAG”？ 在搜索 DAG 目录时，Airflow 忽略不包含 “airflow” 和 “DAG” 的文件，以防止 DagBag 解析导入与用户的 DAG 并置的所有 python 文件。
* 你的`start_date`设置正确吗？ Airflow 会在`start_date + scheduler_interval`时间之后触发任务。
* 您的`schedule_interval`设置正确吗？ 默认`schedule_interval`是一天（`datetime.timedelta(1)`）。 您必须在实例化的 DAG 对象的时候直接指定不同的`schedule_interval` ，而不是通过`default_param`传递参数，因为 task instances 不会覆盖其父 DAG 的`schedule_interval`。
* 您的`start_date`是否超出了在 UI 中可以看到的范围？ 如果将`start_date`设置为3个月之前的某个时间，您将无法在 UI 的主视图中看到它，但您应该能够在`Menu -> Browse -> Task Instances`看到它。
* 是否满足 task 的依赖性。 直接位于 task 上游的任务实例需要处于`success`状态。 此外，如果设置`depends_on_past=True` ，除了满足上游成功之外，前一个调度周期的 task instance 也需要成功（除非该 task 是第一次运行）。 此外，如果设置了`wait_for_downstream=True`，请确保您了解其含义。 您可以从`Task Instance Details`页面查看如何设置这些属性。
* 您需要创建并激活 DagRuns 吗？ DagRun 表示整个 DAG 的特定执行，并具有状态（运行，成功，失败，......）。 scheduler 在向前移动时创建新的 DagRun，但永远不会及时创建新的 DagRun。 scheduler 仅评估`running` 状态的 DagRuns 以查看它可以触发的 task instances。 请注意，清除任务实例（从 UI 或 CLI）会将 DagRun 的状态设置为恢复running。 您可以通过单击 DAG 的计划标记来批量查看 DagRuns 列表并更改状态。
* 是否达到了 DAG 的`concurrency`参数的上限？ `concurrency`定义了允许 DAG 在`running`任务实例的数量，超过这个值，别的就会进入排队队列。
* 是否达到了 DAG 的`max_active_runs`参数的上限？ `max_active_runs`定义允许的 DAG `running`并发实例的数量。

您可能还想阅读文档的 scheduler 部分，并确保完全了解其运行机制。

## 如何根据其他任务的失败触发任务？

查看文档“概念”中的`Trigger Rule`部分

## 安装 airflow[crypto]后，为什么密码在元数据 db 中仍未加密？

查看文档“配置”中的`Connections`部分

## 怎么处理`start_date`？

`start_date`是前 DagRun 时代的部分遗留问题，但它在很多方面仍然具有相关性。 创建新 DAG 时，您可能希望使用`default_args`为任务设置全局`start_date` 。 将基于所有任务的`min(start_date)`创建第一个 DagRun 。 从那时起，scheduler 根据您的 `schedule_interval` 创建新的 DagRuns，并在满足您的依赖项时运行相应的任务实例。 在向 DAG 引入新任务时，您需要特别注意`start_date` ，并且可能希望重新激活非活动的 DagRuns 以正确启用新任务。

我们建议不要使用动态值作为`start_date` ，尤其是`datetime.now()`因为它非常令人疑惑。 周期结束，任务就会被触发，理论上`@hourly` DAG 永远不会达到一小时后，因为`now()`会一直在变化。

以前我们还建议使用与`schedule_interval`相关的`start_date` 。 这意味着`@hourly`将在`00:00`分钟：秒， `@daily`会午夜工作，`@monthly`在每个月的第一天工作。 这不再是必需的。 现在 Airflow 将自动对齐`start_date`和`schedule_interval` ，通过使用`start_date`作为开始查看的时刻。

您可以使用任何传感器或`TimeDeltaSensor`来延迟计划间隔内的任务执行。 虽然`schedule_interval`允许指定`datetime.timedelta`对象，但我们建议使用宏或 cron 表达式作为他的值，因为它强制执行舍入计划的这种想法。

使用`depends_on_past=True`时，必须特别注意`start_date`，因为过去的依赖关系不会仅针对为任务指定的`start_date`的特定计划强制执行。 除非您计划为新任务运行 backfill，否则在引入新的`depends_on_past=True`时及时观察 DagRun 活动状态。

另外需要注意的是，在 backfill CLI 命令的上下文中，任务`start_date`会被 backfill 命令`start_date`覆盖。 这允许对具有`depends_on_past=True`属性任务进行 backfill 操作，如果不是这样涉设计的话，backfill 将无法启动。

## 如何动态创建 DAG？

Airflow 在`DAGS_FOLDER`查找全局命名空间中包含`DAG`对象的模块，并将其添加到`DagBag`中。 在知道这个原理的情况下，我们需要一种方法分配变量到全局命名空间，这可以在 python 中使用标准库中的`globals()`函数轻松完成，就像一个简单的字典。

```py
 for i in range (10):
    dag_id = 'foo_{} '.format(i)
    globals()[dag_id] = DAG(dag_id)
    # 调用一个函数返回 DAG 对象会更好
```

## `airflow run`的所有子命令代表什么？

`airflow run`命令有很多层，这意味着它可以调用自身。

* 基本的`airflow run`：启动 executor，并告诉它运行`airflow run --local`命令。 如果使用 Celery，这意味着它会在队列中放置一个命令，并调用 worker 远程运行。 如果使用 LocalExecutor，则会在子进程池中运行。
* 本地的`airflow run --local`：启动`airflow run --raw`命令（如下所述）作为子进程，负责发出心跳，监听外部杀死进程信号，并确保在子进程失败时进行一些清理工作
* 原始的`airflow run --raw`运行实际 operator 的 execute 方法并执行实际工作

## 怎么使 Airflow dag 运行得更快？

我们可以控制三个变量来改善气流 dag 性能：

* `parallelism`： 此变量控制 Airflow worker 可以同时运行的任务实例的数量。 用户可以通过改变`airflow.cfg`中的 parallelism 调整 并行度变量。
* `concurrency`： Airflow scheduler 在任何时间不会运行超过 `concurrency` 数量的 DAG 实例。 concurrency 在 Airflow DAG 中定义。 如果在 DAG 中没有设置 concurrency，则 scheduler 将使用`airflow.cfg`文件中定义的`dag_concurrency`作为默认值。
* `max_active_runs`： Airflow scheduler 在任何时间不会运行超过 `max_active_runs` DagRuns 数量。 如果在 DAG 中没有设置`max_active_runs` ，则 scheduler 将使用`airflow.cfg`文件中定义的`max_active_runs_per_dag`作为默认值。

## 如何减少 Airflow UI 页面加载时间？

如果你的 dag 需要很长时间才能加载，你可以减小`airflow.cfg`中的`default_dag_run_display_number`的值。 此可配置控制在 UI 中显示的 dag run 的数量，默认值为 25。

## 如何修复异常：Global variable explicit_defaults_for_timestamp needs to be on (1)？

这意味着在 mysql 服务器中禁用了`explicit_defaults_for_timestamp`，您需要通过以下方式启用它：

* 在 my.cnf 文件的 mysqld 部分下设置`explicit_defaults_for_timestamp = 1` 。
* 重启 Mysql 服务器。

## 如何减少生产环境中的 Airflow dag 调度延迟？

* `max_threads`： scheduler 将并行生成多个线程来调度 dags。 这数量是由`max_threads`参数控制，默认值为 2.用户应在生产中将此值增加到更大的值（例如，scheduler 运行机器的 cpus 数量 - 1）。
* `scheduler_heartbeat_sec`： 用户应考虑将`scheduler_heartbeat_sec`配置增加到更高的值（例如 60 秒），该值控制 airflow scheduler 获取心跳和更新作业到数据库中的频率。
