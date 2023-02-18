---
title: asynq 原理分析
date: 2023-02-18 21:17:28
tags: [asynq, task-queue, redis, go]
---

## 概述
Asynq 是一款用 go 语言编写的基于 redis 的分布式任务队列，支持即时任务和定时任务。


## 数据模型
Asynq 使用 redis 的 `List` 和 `Sorted Set` 作为任务队列，队列中只存储任务ID，任务内容使用 redis 的 `Hash` 存储。对应的数据模型如下：

| 数据 | key | value |
| --- | --- | --- |
| `List` 类型队列 | 队列名称 | 成员是任务ID |
| `Sorted Set` 类型队列 | 队列名称 | 成员是任务ID，分数是关联的时间戳 |
| 任务内容 | 任务ID | `{"msg": "...", "state": "..."}` |

任务状态值域： pending，active，completed，retry，archived，scheduled 。

## 运行原理
Asynq 运行时涉及到以下队列：

| 队列类型 | 使用的redis数据类型 | 队列元素内容 | 描述 |
| --- | --- | --- | --- |
| pending | List | task-id | 存储准备执行的任务 |
| active | List | task-id | 存储正在执行的任务 |
| lease | Sorted Set | task-id, lease-expiration-time | 存储正在执行的任务的lease |
| completed | Sorted Set | task-id, task-exipration-time | 存储成功的且设置了保留期的任务 |
| retry | Sorted Set | task-id, retry-time | 存储一段时间后要重试的任务 |
| archived | Sorted Set | task-id, archive-time | 存储归档任务 |
| scheduled | Sorted Set | task-id, process-time | 存储定时任务 |

用户通过 asynq 客户端将任务发布到 asynq 服务器，发布任务时会将即时任务写入 pending 队列，定时任务写入 scheduled 队列。Asynq 服务器端负责执行任务，涉及到 processor，heartbeater，recorver，forwarder，janitor 等多个模块。各模块功能如下：

- processor 模块从 pending 队列中拉取任务，将任务移到 active 队列，为了避免任务执行过程中服务器宕机导致任务丢失，会给每个任务设定一个 lease，并将任务ID和 lease 到期时间写入 lease 列中。每个任务通过单独的 goroutine worker 执行，任务执行有3种情况：
  - 执行成功：将任务从 active 和 lease 队列移除，如果任务设置了保留时间，就将任务写入 completed 队列
  - 执行失败：将任务从 active 和 lease 队列移除，检查任务是否超过指定重试次数，若超过就将任务写入 archived 队列，否则写入 retry 队列
  - 在 lease 有效期内，worker 没有反馈执行结果：留给 recorver 模块处理
- heartbeater 模块定期将 worker 执行信息写入 redis，并延长正在执行的任务的 lease，确保正在执行中的任务不受 lease 到期的影响
- recorver 模块定期检查 lease 队列，将到期的任务移出，检查任务是否超过指定重试次数，若超过就将任务写入 archived 队列，否则写入 retry 队列
- forwarder 模块定期检查 scheduled 和 retry 队列，将到期的任务移到 pending 队列中等待执行
- janitor 模块定期检查 completed 队列，将超过保留期的任务删除

## 其它
- Redis 的 `List` 可用作 FIFO 队列
- Redis 的 `Sorted Set` 可用作优先队列