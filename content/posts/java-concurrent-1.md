---
title: "Java中的锁、条件变量"
date: 2018-10-21T02:54:59+08:00
showDate: true
draft: false
tags: ["Java","并发"]
---


本文是阅读《Java 核心技术》中并发章节的笔记。

## 一个例子

下面这段代码模拟了一个银行，这个银行共有100个账户，每个账户有初始余额100元，银行共有10000元。每个账户启动一个线程在银行内向其他账户随机转钱。按理来说，银行总的钱数应该不变的，实际上这段代码执行后银行的总额并不是10000元。

``` java
import java.util.Arrays;

class Bank {

    private double[] accounts;

    public Bank(int acountCount, Double initialAmount) {
        // 银行有bankCount个账户，每个账户有initialAmount余额
        accounts = new double[acountCount];
        Arrays.fill(accounts, initialAmount);
    }

    public void transfer(int from, int to, Double amount) {
        // 转出账户减少amount
        accounts[from] -= amount;
        // 转入账户增加amount
        accounts[to] += amount;
        System.out.printf("transfer %f from %d to %d, total amount is %f\n", amount, from, to, getTotalAmount());
    }

    public double getTotalAmount() {
        return Arrays.stream(accounts).sum();
    }

    public int getAccountCount() {
        return accounts.length;
    }
}

public class Locks {

    public static final int ACCOUNT_COUNT = 100;

    public static final double INITIAL_AMOUNT = 100;

    public static final int DELAY = 2;

    public static void main(String[] args) {
        Bank bank = new Bank(ACCOUNT_COUNT, INITIAL_AMOUNT);
        for (int i = 0; i < ACCOUNT_COUNT; ++i) {
            int from = i;
            Runnable task = () -> {
                try {
                    while (true) {
                        int to = (int)(Math.random() * bank.getAccountCount());
                        double transferAmount = INITIAL_AMOUNT * Math.random();
                        bank.transfer(from, to, transferAmount);
                        Thread.sleep((int)(Math.random() * DELAY));
                    }
                } catch (InterruptedException e) {

                }
            };
            Thread t = new Thread(task);
            t.start();
        }
    }
}
```

运行输出结果

```  
transfer 96.134765 from 0 to 26, total amount is 9939.794101
transfer 57.480124 from 85 to 59, total amount is 9944.150062
transfer 96.258327 from 3 to 94, total amount is 9944.150062
transfer 95.821430 from 3 to 40, total amount is 10043.375407
transfer 76.855122 from 97 to 67, total amount is 9944.150062
transfer 17.620491 from 2 to 53, total amount is 9944.150062
transfer 99.762802 from 38 to 89, total amount is 9944.150062
transfer 74.513962 from 26 to 43, total amount is 9944.150062
```

从输出结果可以看出，银行总的总钱数是不定的。如果现实生活中银行是这样的，谁敢存钱呢？

## 竞态条件(Race Condition)
在多线程环境下，多个线程同时读写同一块内存区域，且该内存区域的状态因线程的执行顺序不同得到不同结果。这种情况称为*竞态条件(Race Condition)*。

上面的银行转账的例子就存在竞态条件，转账线程通过修改账户余额来完成转账的，相关的语句如下：

``` java
// 转出账户减少amount
accounts[from] -= amount;
// 转入账户增加amount
accounts[to] += amount;
```

拿accounts[from] -= amount为例，这句话看起来是一条语句，但在JVM执行时是分成多条指令执行的。

accounts[from] -= amount可以分解为以下动作：

1. 从内存读取accounts[from]到寄存器中。
2. 将寄存器中的accounts[from]减去amount。
3. 再将寄存器中的结果写回到accounts[from]。

同样地，accounts[to] += amount也可以分解成上面的三步操作（第2步换成加amount)。

假设账户1向账户2转3元，账户3向账户1转3元。如果账户1扣除3元的操作和账户3向账户1增加3元的指令交织在一起，可能会出现错误，看如下的执行步骤：

1. 账户3线程读取账户1余额为x到寄存器
2. 账户1线程读取账户1余额为x到寄存器
3. 账户3线程在寄存器中加3为x + 3
4. 账户1线程在寄存器中减3为x - 3
5. 账户3线程将x + 3写入到账户1余额
6. 账户1线程将x - 3写入到账户1余额

最终账户1中有余额x - 3元，按照常理账户1应该有余额x元，3元消失了。



## 锁

锁的作用是防止两个线程进入一个代码段（称为临界区，critical section），举例如下

``` java
lockObject.lock();
// 临界区
lockObject.unlock();
```

当一个线程获得临界区的锁并进入临界区后，其他线程无法获得锁，只能在临界区外等待，临界区中的代码就不会出现错误。

在银行转账的代码中，如果给transfer函数加锁，保证任何时刻只有一个线程进行转账操作，就能保证结果的正确性。

``` java
class Bank {

    // 创建一个可重入锁
    private Lock transferLock = new ReentrantLock();

    public void transfer(int from, int to, Double amount) {
        try {
            // 获得锁，进入临界区
            transferLock.lock(); 
            // 转出账户减少amount
            accounts[from] -= amount;
            // 转入账户增加amount
            accounts[to] += amount;
            System.out.printf("transfer %f from %d to %d, total amount is %f\n", amount, from, to, getTotalAmount());
        } finally {
            // 释放锁，离开临界区
            transferLock.unlock();
        }
    }

}
```

释放锁操作要放在finally块中，保证抛出异常了还能释放锁。

## 条件变量

如果我想在transfer函数中增加一个限制，只有账户余额大于转出额时才能进行转账，否则就等待其他账户转钱进来，满足条件后再转。看下面的代码：

``` java
class Bank {

    public void transfer(int from, int to, Double amount) {
        try {
            // 获得锁，进入临界区
            transferLock.lock(); 
            // 等待转出账户余额大于amount
            while(accounts[from] < amount);
            // 转出账户减少amount
            accounts[from] -= amount;
            // 转入账户增加amount
            accounts[to] += amount;
            System.out.printf("transfer %f from %d to %d, total amount is %f\n", amount, from, to, getTotalAmount());
        } finally {
            // 释放锁，离开临界区
            transferLock.unlock();
        }
    }
    
}
```

在临界区中增加一个while循环，等待当前账户的余额大于转出金额。实际上这段代码在执行时会进入一个死锁状态，假设某账户不满足转出条件，等待其他账户转入。但他没有释放锁，其他线程无法进入临界区也就不能进行转账，那么会出现所有线程都在等待的状态。

我们可以使用条件变量来解决这种情况。条件变量与锁关联，主要有两个操作：等待，唤醒。当一个线程获得一个锁后，再对与**该锁关联的条件变量**调用等待操作时，调用线程会将自己加入该条件变量的等待队列并**释放锁**，当其他线程在该条件变量上调用唤醒操作时，等待队列中的线程会被唤醒。

在transfer函数中使用条件变量可以在账户余额不满足转出条件时使线程**释放锁并进入等待队列**，当其他线程完成转账操作时再调用唤醒操作将等待中的线程唤醒。示例代码如下：

``` java
	private Lock transferLock = new ReentrantLock();

    private Condition transferCondition = transferLock.newCondition();

    public void transfer(int from, int to, Double amount) {
        try {
            transferLock.lock();
            // await操作放在循环中为了在唤醒时再次检查是否满足转出条件，可以避免虚假唤醒（spurious wakeup）
            while(accounts[from] < amount) {
                // 释放锁，进入等待队列
                transferCondition.await();
            };
            // 转出账户减少amount
            accounts[from] -= amount;
            // 转入账户增加amount
            accounts[to] += amount;
            System.out.printf("transfer %f from %d to %d, total amount is %f\n", amount, from, to, getTotalAmount());
            // 唤醒等待队列中所有等待线程
            transferCondition.signalAll();
        } catch (InterruptedException e) {
            transferCondition.signalAll();
        } finally {
            transferLock.unlock();
        }
    }
```





## Java Object内置的锁和条件变量

Java比较有意思的一个地方是所有的Object都内置一个锁和条件变量。如果一个函数声明中带有synchronized关键字，这个函数在调用过程中会隐式获得实例的锁。

``` java
class TestSynchronized {
    synchronized public void hello() {
        // 操作
    }
}

// 类似于

class TestSynchronized {
    public void hello() {
        this.intrinsicLock.lock();
        // 操作
        this.intrinsicLock.unlock();
    }
}
```

对于内置的条件变量，可以用wait()，notify()和notifyAll()进行操作。
