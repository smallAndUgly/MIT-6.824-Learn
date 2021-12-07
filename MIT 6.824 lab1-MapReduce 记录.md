# MIT 6.824 lab1-MapReduce 记录

## 实验遇到的问题

**1. 是否需要等待所有Map任务结束再开启Reduce任务？**

**2. 是否需要coordinator将Map任务的结果整合，将某个特定中间键的Reduce任务交给特定的worker处理？**

**3. 如何监控crash的任务？**

**4. 如何检测到所有任务完毕，使得Done()方法返回true？**

**5. 如何协调任务中的资源竞争问题？**

**6. 如何防止被认为crash的worker传回数据导致数据重复？**

## 数据结构

1. 任务结构体---Job

   - 任务类型Map or Reduce
   - 任务的唯一标识符
   - 任务的文件名
   - 任务的内容

2. 保存Map/Reduce任务状态的两个结构体。

   记录着未开始的任务、正在进行的任务、已经完成的任务。使用map数据集合来进行管理，以任务的唯一标识符来作为Key。使用通道来进行各个状态之间的转换。

3. channel，用来传递需要被Map/Reduce的任务。

4. 中间键列表，使用map集合来管理，Reduce的序列号作为Key，worker传来的文件位置列表作为Value。

## MapReduce过程

1. coordinator 启动，初始化coordinator，注册rpc，开启守护线程，来时刻观察Map/Reduce任务的状态变化。
2. 传入已经分好的任务。
3. worker启动，worker向coordinator发送rpc请求，获取到Map任务。
4. worker将map任务结果存于磁盘中，向coordinator发送Map任务结果的位置。
5. coordinator等待所有Map任务结束，整理结果。
6. worker获取Reduce任务。
7. 等待所有Reduce任务结束后，coordiantor结束，调用Done()方法返回。

## worker获取任务

- rpc调用AskJob()方法来获取任务。
- AskJob()使用select来从需要Map的channel或者需要Reduce的channel中获取任务，返回Job以及获取任务成功的标志。
- 如果获取任务成功，coordinator的守护线程，将Map/Reduce任务的状态由未执行改变为正在执行（通过通道来通知守护线程）。
- 如果没有任务从channel中获取到，返回获取任务失败的标志。
- 如果没有获取到任务，worker调用sleep，等待10s中再次请求任务。
- 任务获取成功后开启监控线程，用来监控任务是否完成，如果该任务仍在正在进行的任务的列表，重新将该任务放入需要Map的channel或者需要Reduce的channel，并将该任务的状态改变为未执行。

## worker完成任务

- rpc调用DoneJob()方法来通知coordinator已经完成任务。
- worker将生成的中间键文件地址列表传给coordinator。
- coordinator收到完成任务的调用，将消息通知给Map/Reduce状态维护的数据结构。
- 首先查看该任务是否再正在进行的列表中，如果不在，说明之前已经又一个worker提交了该任务，舍弃此次worker的提交。
- Map/Reduce的状态由正在进行转变为已完成。
- Map状态表检测未完成列表与正在进行列表是否为空，如果为空，代表所有的Map任务做完，将中间键列表由Reduce序列号分类。
- Reduce状态表检测未完成列表与正在进行列表是否为空，如果为空，代表所有Reduce任务完成，使用channel通知Done()方法，关闭coordinator

## 处理crash的worker

- 当任务被worker取走，开启一个监控线程，sleep10s，10s后根据任务的唯一标识符来从状态表中查看是否存在于正在进行列表中 。
- 如果存在，将该任务重新加入到需要Map的任务队列中。

