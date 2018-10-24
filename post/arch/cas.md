---
title: "使用CAS解决一个业务问题"
date: 2018-10-21T18:47:24+08:00
draft: false
---

### 问题描述
最近项目遇到这样一个问题，有一个用mysql数据库表模拟的任务队列，生产者会往表中增加任务，消费者采用轮询的方式去获取任务。其中新加入的任务的状态为NEW(1)，当任务被拿走并处于计算中的状态为PENDING(2)，当任务处理成功的状态为SUCCESS(0)，当任务被处理失败的状态为FAILED(3)。消费者从轮询数据库，如果有NEW状态的任务，拿到任务，则修改状态为PENDING。问题是，有多个消费者同时去查询数据库表，并更新表项，存在并发问题。

### 初步解决
对于数据库并发问题，一个直觉的做法就是采用加锁的办法，因此采用下面这种方式实现:
```go
    tx := db.Begin()
    tx = tx.Raw("SELECT * FROM parser_job WHERE id=? FOR UPDATE", job.Id).Scan(&job)
    if tx.RecordNotFound() {
        tx.Rollback()
        return nil, nil
    }
    if tx.Error != nil {
        logs.Errorf("DequeJob db query failed, err:%s", tx.Error.Error())
        tx.Rollback()
        return nil, errors.New(tx.Error.Error())
    }
    columns := make(map[string]interface{}, 0)
    columns["status"] = common.STATUS_PENDING
    err := tx.Exec("UPDATE parser_job SET status=? WHERE id=?", common.STATUS_PENDING, job.Id).Error
    if err != nil {
        logs.Errorf("DequeJob db update failed, err:%s", err.Error())
        tx.Rollback()
        return nil, err
    }
    err = tx.Commit().Error
    if err != nil {
        logs.Errorf("DequeJob db commit failed, err:%s", err.Error())
        return nil, err
    }
```
由于查询和更新是两步操作，不是原子操作，并且使用了for update，所以这里使用了事务，其中`SELECT * FROM parser_job WHERE id=? FOR UPDATE`由于id是主键，因此如果查询到数据，则产生行锁(参考：[Mysql中select for update排他锁分析](https://blog.csdn.net/claram/article/details/54023216))。
但是这个地方真的有效吗？一个消费者把任务拿走了，同时更新状态为PENDING，其他的消费者即使被阻塞在行锁，等行锁解除, 但还是会进去修改这个job的状态，并拿走任务。这是一个错误的解法。
### 正确解法
针对我们的场景，我们只是要防止一个任务被多次消费，并且是通过状态来控制是否被消费过。这里加锁是不能解决问题的。采用CAS(compare and set)的方法解决：
```go
    columns := make(map[string]interface{}, 0)
    columns["status"] = common.STATUS_PENDING
    columns["update_time"] = time.Now()
    res = db.Table("parser_job").Where("id=? and status=?", job.Id, common.STATUS_NEW).Update(columns)
    if res.Error != nil {
        logs.Errorf("update parser_job failed, err:%s", res.Error.Error())
        return nil, res.Error
    }
    if res.RowsAffected == 0 {
        return nil, nil
    }

    res2 := db.Table("parser_job").Where("id=?", job.Id).First(&job)
    if res2.RecordNotFound() {
        logs.Warnf("find parser_job failed, record not found by id=?", job.Id)
        return nil, nil
    }
    if res2.Error != nil {
        logs.Warnf("find parser_job failed by id=%d, err:%s", job.Id, res2.Error.Error())
        return nil, res2.Error
    }
```
这里，采用更新的方式拿任务，在更新件中加上status的判断，如果更新操作影响的行数为1，则说明拿到任务。这个地方是一个乐观锁，即使两个消费者同时更新操作，也只有一个消费者更新成功，因为一条sql语句具有原子性。
