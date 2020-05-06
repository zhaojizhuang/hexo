---
layout: post
title:  "分布式锁"
date:   2020-02-10 11:40:18 +0800
categories: 分布式
tags:  ["分布式", "go"]
author: zhaojizhuang

---



# 分布式锁

## k8s 的选主机制

通过生成一个 k8s资源来实现

[https://blog.csdn.net/weixin_39961559/article/details/81877056](https://blog.csdn.net/weixin_39961559/article/details/81877056)


## etcd自己实现的锁和选主
[https://yq.aliyun.com/articles/70546](https://yq.aliyun.com/articles/70546)

`Etcd` 的 `v3` 版本官方 `client` 里有一个 `concurrency` 的包，里面实现了分布式锁和选主。本文分析一下它是如何实现的。


- 锁的code [https://github.com/coreos/etcd/blob/master/clientv3/concurrency/mutex.go#L26](https://github.com/coreos/etcd/blob/master/clientv3/concurrency/mutex.go#L26)

- 选主的实现与锁的实现非常类似 [https://github.com/coreos/etcd/blob/master/clientv3/concurrency/election.go#L31](https://github.com/coreos/etcd/blob/master/clientv3/concurrency/election.go#L31)

## `etcd` 的分布式锁，利用 `etcd` 的租约机制

[https://blog.51cto.com/5660061/2381931](https://blog.51cto.com/5660061/2381931)

```go
package main

import (
	"context"
	"fmt"
	"github.com/coreos/etcd/clientv3"
	"time"
)

func main() {
	config := clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	}
	client, err := clientv3.New(config)
	if err != nil {
		fmt.Println(err)
	}
	lease := clientv3.NewLease(client)
	// Grant：分配一个租约。
	// Revoke：释放一个租约。
	// TimeToLive：获取剩余TTL时间。
	// Leases：列举所有etcd中的租约。
	// KeepAlive：自动定时的续约某个租约。
	leaseResp, err := lease.Grant(context.TODO(), 10) //创建一个租约，它有10秒的TTL：
	if err != nil {
		fmt.Println(err)
	}
	leaseID := leaseResp.ID
	ctx, cancelFunc := context.WithCancel(context.TODO())
	// 两个defer用于释放锁
	defer cancelFunc()
	defer lease.Revoke(context.TODO(), leaseID)

	// 抢锁和占用期间，需要不停的续租，续租方法返回一个只读的channel
	keepChan, err := lease.KeepAlive(ctx, leaseID)
	if err != nil {
		fmt.Println(err)
	}

	// 处理续租返回的信息
	go func() {
		for {
			select {
			case keepResp := <-keepChan:
				if keepChan == nil {
					fmt.Println("lease out")
					goto END
				} else {
					fmt.Println("get resp", keepResp.ID)
				}
			}
		}
	END:
	}()
	kv := clientv3.NewKV(client)
	//put一个kv，让它与租约关联起来，从而实现10秒后自动过期
	putResp, err := kv.Put(context.TODO(), "/cron/lock/job1", "", clientv3.WithLease(leaseID))
	if err != nil {
		fmt.Println(err)
		return
	}

	fmt.Println("写入成功:", putResp.Header.Revision)

	//定时看key过期没
	for {
		getResp, err := kv.Get(context.TODO(), "/cron/lock/job1")
		if err != nil {
			fmt.Println(err)
			return
		}
		if getResp.Count == 0 {
			fmt.Println("kv过期了")
			break
		}
		fmt.Println("还没过期:", getResp.Kvs)
		time.Sleep(time.Second)
	}
}

```



```go 
package main

import (
	"context"
	"fmt"
	"go.etcd.io/etcd/clientv3"
	"time"
)

type ETCDMutex struct {
	Ttl     int64
	Conf    clientv3.Config
	Key     string
	cancel  context.CancelFunc
	lease   clientv3.Lease
	leaseID clientv3.LeaseID
	txn     clientv3.Txn
}

func (em *ETCDMutex) init() error {
	client, err := clientv3.New(em.Conf)
	if err != nil {
		return err
	}
	em.txn = clientv3.KV(client).Txn(context.TODO())

	if err != nil {
		return err
	}
	em.lease = clientv3.NewLease(client)
	leaseResp, err := em.lease.Grant(context.TODO(), em.Ttl)
	if err != nil {
		return err
	}
	var ctx context.Context
	ctx, em.cancel = context.WithCancel(context.TODO())
	em.leaseID = leaseResp.ID
	_, err = em.lease.KeepAlive(ctx, em.leaseID)
	return err
}
func (em *ETCDMutex) lock() error {
	err := em.init()
	if err != nil {
		return err
	}
	// CreateRevision ==0 表示key不存在
	//LOCK:
	txnResp, err := em.txn.If(clientv3.Compare(clientv3.CreateRevision(em.Key), "=", 0)).
		Then(clientv3.OpPut(em.Key, "", clientv3.WithLease(em.leaseID))).Commit()
	if err != nil {
		return err
	}
	if !txnResp.Succeeded { //判断txn.if条件是否成立
		return fmt.Errorf("抢锁失败")
	}
	return nil
}

func (em *ETCDMutex) UnLock() {
	em.cancel()
	em.lease.Revoke(context.TODO(), em.leaseID)
	fmt.Println("释放了锁")
}

func main() {
	var conf = clientv3.Config{
		Endpoints:   []string{"172.16.196.129:2380", "192.168.50.250:2380"},
		DialTimeout: 5 * time.Second,
	}
	eMutex1 := &ETCDMutex{
		Conf: conf,
		Ttl:  10,
		Key:  "lock",
	}
	eMutex2 := &ETCDMutex{
		Conf: conf,
		Ttl:  10,
		Key:  "lock",
	}
	//groutine1
	go func() {
		err := eMutex1.lock()
		if err != nil {
			fmt.Println("groutine1抢锁失败")
			fmt.Println(err)
			return
		}
		//可以做点其他事，比如访问和操作分布式资源
		fmt.Println("groutine1抢锁成功")
		time.Sleep(10 * time.Second)
		defer eMutex1.UnLock()
	}()

	//groutine2
	go func() {
		err := eMutex2.lock()
		if err != nil {
			fmt.Println("groutine2抢锁失败")
			fmt.Println(err)
			return
		}
		//可以做点其他事，比如访问和操作分布式资源
		fmt.Println("groutine2抢锁成功")
		defer eMutex2.UnLock()
	}()
	time.Sleep(30 * time.Second)
}

```

https://studygolang.com/articles/16307?fr=sidebar