---
title:   "[Airflow] - Airflow executor comparison"
excerpt: "[Airflow] - Airflow executor comparison"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - airflow
  - HELM
last_modified_at: 2023-09-16T12:06:00+09:00
---

# Background

airflow에서 task를 생성하고 실행하는데 크게 `scheduler`와 `executor` component가 영향을 줍니다.
`scheduler`는 dag 상태 update, task를 trigger 및  `executor`에게 넘기는 역할을 하고, 
`executor`는 task의 실행 process 에 관여를 합니다. 

오늘은 airflow 소스코드를 통해 scheduler 및 executor의 작동원리를 살펴보겠습니다.
executor로는 `local executor` 와 production 환경에 주로 쓰이는 `celery executor`,  `k8s executor` 을 비교해보겠습니다.

# Contents

## scheduler

---

scheduler의 main step에서는 dag parsing 결과를 바탕으로 `task instance(TI)`를 생성한후 executor에게 전달해주는 역할을 합니다. 이후 executor의 heartbeat 함수를 호출해 queued_task를 비동기로 처리합니다.

**`airflow/airflow/jobs/scheduler_job_runner.py`**

```python
def _run_scheduler_loop(self) -> None:
  """
  Harvest DAG parsing results, queue tasks, and perform executor heartbeat; the actual scheduler loop.

  """
  ...	
  with create_session() as session:
      num_queued_tis = self._do_scheduling(session)

      self.job.executor.heartbeat()
  ...
	 
```

_do_scheduling 함수와 _critical_section_enqueue_task_instances 함수를 통해 task_instance를 enqueue 합니다. `scheduler`를 ha로 구성하였을때 오직 하나의 `scheduler` process만 이 함수를 사용할수 있도록 critical section으로 구성하였습니다.

```python
def _do_scheduling(self, session: Session) -> int:
  """
  Make the main scheduling decisions.

  """
  with prohibit_commit(session) as guard:
	...
      # Find anything TIs in state SCHEDULED, try to QUEUE it (send it to the executor)
      num_queued_tis = self._critical_section_enqueue_task_instances(session=session)
	...
      guard.commit()

def _critical_section_enqueue_task_instances(self, session: Session) -> int:
  """
  Enqueues TaskInstances for execution.

  """
  ...
  queued_tis = self._executable_task_instances_to_queued(max_tis, session=session) # list[TI]

  self._enqueue_task_instances_with_queued_state(queued_tis, session=session)
  ...

```

_enqueue_task_instances_with_queued_state 함수 내부에서 `executor`의 queue_command 함수를 통해 ti를 저장합니다.

```python
def _enqueue_task_instances_with_queued_state(self, task_instances: list[TI], session: Session) -> None:
  """
  Enqueue task_instances which should have been set to queued with the executor.
	
  :param task_instances: TaskInstances to enqueue
  """
  # actually enqueue them
  for ti in task_instances:
    ...
    self.job.executor.queue_command(
      ti,
      command,
      priority=priority,
      queue=queue,
    )
```

base_executor의 `queue_command` 함수에서 executor의 queued_tasks에 ti를 저장합니다.

**`airflow/airflow/executors/base_executor.py`**

```python
def queue_command(
  self,
  task_instance: TaskInstance,
  command: CommandType,
  priority: int = 1,
  queue: str | None = None,
  ):
  ...
  """Queues command to task."""
  if task_instance.key not in self.queued_tasks:
    self.queued_tasks[task_instance.key] = (command, priority, queue, task_instance)
```

## base executor

---

모든 executor들은 base_executor class를 상속받습니다. 이후 executor의 실행방식에 맞게 함수들을 override합니다. 기본적으로 `scheduler`에 의해 `executor`의 `heartbeat` 함수가 주기적으로 실행되고 open slots 갯수만큼 task가 실행됩니다.

**`airflow/airflow/executors/base_executor.py`**

```python
def heartbeat(self) -> None:
  """Heartbeat sent to trigger new jobs."""

  open_slots = self.parallelism - len(self.running)
	
  num_running_tasks = len(self.running)
  num_queued_tasks = len(self.queued_tasks)
	
  ...
  self.trigger_tasks(open_slots)
  ...

  self.sync()
```

`trigger_tasks`함수에서 실행가능한 slot 갯수 만큼 task_instance를 queued_task에서 가져와 실행합니다

```python
def trigger_tasks(self, open_slots: int) -> None:
  """
  Initiate async execution of the queued tasks, up to the number of available slots.
  """
  for _ in range(min((open_slots, len(self.queued_tasks)))):
    ...  
    task_tuples.append((key, command, queue, ti.executor_config))

  if task_tuples:
    self._process_tasks(task_tuples)
```

`_process_task` 에서 `execute_async` 함수를 호출해 비동기로 task를 처리합니다

```python
def _process_tasks(self, task_tuples: list[TaskTuple]) -> None:
  for key, command, queue, executor_config in task_tuples:
    del self.queued_tasks[key]
    self.**execute_async**(key=key, command=command, queue=queue, executor_config=executor_config)
    self.running.add(key)
```

이렇듯 `scheduler`가 task instance를 `executor`의 task_queue에 넣어주면 executor의 heartbeat 함수를 통해 task instance를 `execute_async` 함수 또는`_process_task` 함수에서 처리를 합니다.
앞으로 살펴볼 `local executor` 와  `kubernetes executor`는 `execute_async` 함수를,
`celery executor`는  `_process_task`를 override 합니다.

## Local executor

---

local executor에는 parallelism 설정에 따라 크게 2가지 작동방식이 있습니다.

### 1. **`self.parallelism == 0`**

`self.parallelism == 0` 일때는 `class UnlimitedParallelism` 가 local executor의 실행을 담당합니다.  `execute_async`가 called 될때마다  process를 spawn 하고 `class LocalWorker` 를 통해 이 전략이 실행됩니다.  코드로 보면 다음과 같습니다.

```python
class LocalExecutor(BaseExecutor):
  ...
  class UnlimitedParallelism:
    ...
    def start(self) -> None:
      self.executor.workers_used = 0
      self.executor.workers_active = 0

    def execute_async(
      self,
      key: TaskInstanceKey,
      command: CommandType,
      queue: str | None = None,
      executor_config: Any | None = None,
    ) -> None:
      ...
      local_worker = **LocalWorker**(self.executor.result_queue, key=key, command=command)
      ...
      local_worker.start()
```

`UnlimitedParallelism`에서 `execute_async`가 호출 됬을때를 살펴보겠습니다. TI가 전달되었을때 함수내에서 `LocalWorker.start` 가 호출되고, `LocalWorker` 부모 class인 `Process`의 start가 호출되어 
target function으로 지정된 `LocalWorker.do_work`  이 호출됩니다

```python
class LocalWorkerBase(**Process**, LoggingMixin):
  def __init__(self, result_queue: Queue[TaskInstanceStateType]):
    super().__init__(target=**self.do_work**)
    ...
	
class LocalWorker(LocalWorkerBase):
  ...
  def do_work(self) -> None:
    self.execute_work(key=self.key, command=self.command)
```

`localworkerbase`의 `executor_work`가 실행되고 `subprocess/fork` 방식으로 process를 생성하여 command를 실행합니다.

```python
class LocalWorkerBase(Process, LoggingMixin):
  ...
  def execute_work(self, key: TaskInstanceKey, command: CommandType) -> None:
  """
  Execute command received and stores result state in queue.

  """

  if settings.EXECUTE_TASKS_NEW_PYTHON_INTERPRETER:
    state = self._execute_work_in_subprocess(command)
  else:
    state = self._execute_work_in_fork(command)

    self.result_queue.put((key, state))

```

### 2. **`self.parallelism > 0`**

`self.parallelism > 0` 일때는 `self.parallelism` 만큼 worker process를 미리 생성해놓습니다. 
그리고 task queue를 이용해서 task ingestion을 처리합니다. `class QueuedLocalWorker` 를 통해 이 전략이 실행됩니다. 코드로 보면 다음과 같습니다.

`executor` 생성시 `start` 함수에서 task 처리 및 공유를 위한 준비를 합니다.
command를 실행하기위한 `QueuedLocalWorker` 와 
task_instance 공유를 위한 `manager.Queue` 를 생성합니다.

```python
class LocalExecutor(BaseExecutor):
  ...
  class LimitedParallelism:
    ...
    def start(self) -> None:
      """Start limited parallelism implementation."""
		
      self.queue = self.executor.**manager.Queue()**
      self.executor.workers = [
        **QueuedLocalWorker**(self.queue, self.executor.result_queue)
      for _ in range(self.executor.parallelism)
      ]
		
      self.executor.workers_used = len(self.executor.workers)
		
      for worker in self.executor.workers:
        worker.**start**()
```

각 worker에 대해서 start시 위의 경우와 마찬가지로 `Process` class의 상속으로 인해 `target function`으로 설정된 `do_work`가 실행됩니다. `LocalWorker`와는 다르게 `QueuedLocalWorker`는 do_work 실행시 task_queue에서 ti 를 받을때까지 대기하고 ti가 준비되면 실행합니다. 

```python
class QueuedLocalWorker(LocalWorkerBase):
  ...
  def do_work(self) -> None:
    while True:
      try:
        key, command = **self.task_queue.get**()
        ...
        self.execute_work(key=key, command=command)
      finally:
        self.task_queue.task_done()

```

`LimitedParallelism`에서 `execute_async` 가 호출되면 task_queue에 put을 함으로써 `QueuedLocalworker`에게 ti를 전달해줍니다. `do_work`로 대기하고 있던 `QueuedLocalworker`는 queue에서 ti를 전달받아 처리합니다.

```python
class LocalExecutor(BaseExecutor):
  ...
  class LimitedParallelism:
    ...
      def execute_async(self, key, command, queue, executor_config) -> None:
        ...
        self.queue.put((key, command))
```

## Kubernetes executor

---

kubernetes는 각 task instance별로 pod를 생성하여 실행합니다.

task instance key를 받으면 `PodGenerator`가 `from_obj` 함수로 `k8s.V1Pod` object를 반환합니다

**`airflow/airflow/providers/cncf/kubernetes/executors/kubernetes_executor.py`**

```python
def execute_async(
        self,
        key: TaskInstanceKey,
        command: CommandType,
        queue: str | None = None,
        executor_config: Any | None = None,
    ) -> None:
    """Executes task asynchronously."""

    kube_executor_config = PodGenerator.from_obj(executor_config)
   
    if executor_config:
        pod_template_file = executor_config.get("pod_template_file", None)
    
        self.task_queue.put((key, command, kube_executor_config, pod_template_file))
    
```

from_obj 함수에서는 pod_override key값을 이용해 Pod를 get 또는 create 해서 반환합니다.

**`airflow/airflow/providers/cncf/kubernetes/pod_generator.py`**

```python
def from_obj(obj) -> dict | k8s.V1Pod | None:
    """Converts to pod from obj."""
    if obj is None:
        return None
		
    k8s_legacy_object = obj.get("KubernetesExecutor", None)
    k8s_object = obj.get("pod_override", None)
		
    ...
    
    if isinstance(k8s_object, k8s.V1Pod):
        return k8s_object
    elif isinstance(k8s_legacy_object, dict):
        return PodGenerator.from_legacy_obj(obj)
```

## Celery executor

---

celery executor는  `list[TaskTuple]`를 인자로 받는 `_process_tasks` 를 override 합니다.
task 정보와 executor_command 함수 객체를 tuple로 만들어 `_send_tasks_to_celery` 의 인자로 넘겨줍니다.

**`airflow/airflow/providers/celery/executor/celery_executor.py`**

```python
def _process_tasks(self, task_tuples: list[TaskTuple]) -> None:
    from airflow.providers.celery.executors.celery_executor_utils import execute_command
		
    task_tuples_to_send = [task_tuple[:3] + (**execute_command**,) for task_tuple in task_tuples]
    ...
    key_and_async_results = self.**_send_tasks_to_celery**(task_tuples_to_send)
		
```

해당 task_tuple들을 send_task_to_executor를 통해 celery worker로 전달됩니다.

```python
def _send_tasks_to_celery(self, task_tuples_to_send: list[TaskInstanceInCelery]):
    from airflow.providers.celery.executors.celery_executor_utils import send_task_to_executor
		
    if len(task_tuples_to_send) == 1 or self._sync_parallelism == 1:
        # One tuple, or max one process -> send it in the main thread.
        return list(map(**send_task_to_executor**, task_tuples_to_send))
    ...
    with ProcessPoolExecutor(max_workers=num_processes) as send_pool:
        key_and_async_results = list(
        send_pool.map(**send_task_to_executor**, task_tuples_to_send, chunksize=chunksize)
        )
        return key_and_async_results
```

이후 `task_to_run.apply_async`를 통해 task를 celery worker에서 비동기로 처리합니다.
`task_to_run`은 위에서 인자로 넘겨진 `execute_command`로 celery task 로써 호출됩니다.

**`airflow/airflow/providers/celery/executor/celery_executor_utils.py`**

```python
def send_task_to_executor(task_tuple: TaskInstanceInCelery,) -> tuple[TaskInstanceKey, CommandType, AsyncResult | ExceptionWithTraceback]:
    """Sends task to executor."""
    key, command, queue, task_to_run = task_tuple
    with timeout(seconds=OPERATION_TIMEOUT):
        result = **task_to_run.apply_async**(args=[command], queue=queue)
  
    return key, command, result
```

celery task로써 `execute_command`가 호출되면 worker내에서 subprocess/fork 형태로 task가 실행됩니다.

```python
app = Celery(celery_app_name, config_source=celery_configuration)

**@app.task**
def **execute_command**(command_to_exec: CommandType) -> None:
    """Executes command."""
    ...
    try:
        if settings.EXECUTE_TASKS_NEW_PYTHON_INTERPRETER:
            _execute_in_subprocess(command_to_exec, celery_task_id)
        else:
            _execute_in_fork(command_to_exec, celery_task_id)
```

## Summary

---

소스코드를 보면서 local executor, kubernetes executor, celery executor 실행 방식의 차이를 보았습니다.

local executor는 parallelism 설정에 따라 process 생성방식에 차이가 있지만,
공통적으로 내부 resource 내에서 process들을 통해 task들을 처리합니다. 
실무에서 주로 쓰이는 remote executor 인 kubernetes executor는 task 마다 pod를 실행시키고,
celery executor는 queue를 통해 ingestion 이후 celery worker에서 subprocess를 통한 실시간 처리를 지향합니다. 실행 방식에 따라 장단점이 있기에 실제 업무에서 사용하게 되는 경우도 다릅니다.

kubernetes의 경우 scale out이 용이하고 pod의 image들을 customize 할수 있다는 장점이 있습니다.
하지만 task 마다 pod를 실행하기에 수초정도의 overhead가 생겨서 실시간성을 요하는 task 처리에는 적합하지 않습니다.

celery의 경우 scale out이 가능하고 task가 거의 실시간으로 처리된다는 장점이 있습니다. 또한 task queue를 통해 cluster의 resource 보다 더 큰 ingestion을 할수 있습니다. 하지만 task들이 실행되는 환경(image, resource requirments)이 celery worker process의 airflow image 에 종속적이라는 단점이 있고 celery worker 및 db_backend에 대한 관리 포인트 증가 및 기본적으로 사용되는 resource가 존재한다는 단점도 있습니다. **하지만 helm chart version에서 hpa 기반 keda autoscaler가 지원되면서 celery worker에 대한 리소스 관리(대기 리소스 감소 및 burst workload 처리) 가 많이 용이해졌습니다.** 또한 k8s환경위에서 celery resource들을 관리한다면 관리포인트에대한 부담도 일정부분 줄어듭니다.

지금까지 airflow의 executor별 내부 처리 방식을 살펴보았습니다. 운영 환경/상황에 맞는 executor를 써보면 좋을것 같습니다.

### Reference

---

[**Airflow official document**](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/executor/index.html#executor-types) 

[**Airflow sourcecode**](https://github.com/apache/airflow/tree/main/airflow)