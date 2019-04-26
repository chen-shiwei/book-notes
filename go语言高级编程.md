# go语言高级编程
## 1语言基础
### 1.5面对并发的内存模型
- cpu 单核模式 “顺序” 执行机器指令
- 顺序编程：

    所有的指令都是以串行的方式执行，在 相同的时刻  有且 仅有一个CPU 在 顺序 执行程序的指令
- 并行编程：
    - 多线程：
        ```
        多核 系统级的多线程支持 主流的编程语言特性
        ```
    - 消息传递
        - Erlang语言 并发体之间不共享内存
        - Go 语言 CSP模型 Goroutine之间是共享内存
    - 理论上，多线程和基于消息的并发编程是等价的
#### 1.5.1 Goroutine和系统线程
- Goroutine
    - 轻量级线程（可能是2KB或4KB），不等价 系统级线程
        - 当遇到深度递归导致当前栈空间不足时，动态地伸缩栈的大小（主流实现中栈的最大值可达到1GB）
    - go关键字启动
    - 调度
        - 半抢占式的协作调度
        - go语言自己调度
        - Goroutine发生阻塞时才会导致调度
        - 同时发生在用户态，调度器会根据具体函数只保存必要的寄存器，切换的代价要比系统线程低得多
        - runtime.GOMAXPROCS，控制当前运行正常非阻塞Goroutine的系统线程数目
- 系统级线程
    - 一个固定大小的栈（一般默认可能是2MB）
        - 作用  保存函数递归调用时参数和局部变量
        - 缺点
            - 浪费
            - 对于少数需要巨大栈空间的线程来说又面临栈溢出的风险（内存不够用）
            - 降低固定的栈大小，要么增大栈的大小以允许更深的函数递归调用，但这两者是没法同时兼得的
#### 1.5.2 原子操作
- 并发编程中“最小的且不可并行化”
- 标准库
    - sync/atomic
    - sync.Mutex
    - sync/once
#### 1.5.3 顺序一致性内存模型
- 同一个Goroutine线程内部，顺序一致性内存模型是得到保证的
- 不同的Goroutine之间，并不满足顺序一致性内存模型，需要通过明确定义的同步事件来作为同步的参考
#### 1.5.4 初始化顺序
![Alt text](https://chai2010.cn/advanced-go-programming-book/images/ch1-12-init.ditaa.png "go初始化顺序")
- 在main.main函数执行之前所有代码都运行在同一个Goroutine中
- 如果某个init函数内部用go关键字启动了新的Goroutine的话，新的Goroutine和main.main函数是并发执行的
- 所有的init函数和main函数都是在主线程完成，它们也是满足顺序一致性模型的
#### 1.5.5 Goroutine的创建
```
    go func()
```
#### 1.5.6 基于Channel的通信
- 无缓冲 的Channel上的"发送"操作总在对应的"接收操作"完成前发生 
    > 接受者准备先完成后发生完成
- 带缓冲 的Channel，对于Channel的第K个接收完成操作发生在第K+C个发送操作完成之前，其中C是Channel的缓存大小
 > K+C发送完成后，第K个接受完成
#### 1.5.7 不靠谱的同步
- 先后顺序不可靠的
- 使用显示的同步
### 1.6 常见的并发模式
- CSP（Communicating Sequential Process，通讯顺序进程）
    - CSP理论的核心概念：同步通信
    - Do not communicate by sharing memory; instead, share memory by communicating.
        - 不要通过共享内存来通信，而应通过通信来共享内存。
- 并发不是并行
    - 并发
        - 并发更关注的是程序的设计层面，
        - 并发的程序完全是可以顺序执行的
    - 并行 
        - 更关注的是程序的运行层面
        - 并行一般是简单的大量重复
        - 只有在真正的多核CPU上才可能真正地同时运行
        - 例如GPU中对图像处理都会有大量的并行运算
- Go语言 内建的并发支持
- 原子操作或互斥锁 简单好实现
- Channel 代码简洁
#### 1.6.2 生产者消费者模型
#### 1.6.3 发布订阅模型
- golang.org/x/tools/godoc/vfs 
    - 控制访问该虚拟文件系统的最大并发数
#### 1.6.5 赢者为王
- 采用并发编程的动机
    - 简化问题
    - 提升性能
### 1.7 错误和异常
 - recover 将内部异常转为错误处理
 - 捕获异常不是最终的目的
    ```
        如果异常不可预测，直接输出异常信息是最好的处理方式
    ```
#### 1.7.1 错误处理策略
- defer 可以让我们在打开文件时马上思考如何关闭文件
- panic抛出异常 
    - 函数将停止执行后续的普通语句
    - 但是之前注册的defer函数调用仍然保证会被正常执行，然后再返回到调用者
    - 在异常发生时，如果在defer中执行recover调用，它可以捕获触发panic时的参数，并且恢复到正常的执行流程
    ```
        1 必须在defer函数中直接调用recover
        2 如果defer中调用的是recover函数的包装函数的话，异常的捕获工作将失败！
    ```
    ```go
        // 1
            func main() {
                defer func() {
                    // 无法捕获异常
                    if r := MyRecover(); r != nil {
                        fmt.Println(r)
                    }
                }()
                panic(1)
            }

            func MyRecover() interface{} {
                log.Println("trace...")
                return recover()
                }
        //2
            func main() {
                defer func() {
                    defer func() {
                        // 无法捕获异常
                        if r := recover(); r != nil {
                            fmt.Println(r)
                        }
                    }()
                }()
                panic(1)
            }
        //3
        func main() {
            // 可以正常捕获异常
            defer MyRecover()
            panic(1)
        }
        //4
        func main() {
            // 无法捕获异常
            defer recover()
            panic(1)
        }
        //5
        func main() {
            defer func() {
                if r := recover(); r != nil { ... }
                // 虽然总是返回nil, 但是可以恢复异常状态
            }()

            // 警告: 用`nil`为参数抛出异常
            panic(nil)
        }
    ```
## 2CGO编程
## 第4章 RPC和Protobuf
- RPC(Remote Procedure Call) 远程过程调用
    - 分布式系统中不同节点间流行的通信方式
