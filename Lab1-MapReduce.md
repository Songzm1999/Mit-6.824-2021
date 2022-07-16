# 设计思路

由于在课程的官网上不建议把代码都贴出来，我这里只是附上一些数据结构和介绍一下设计的思路。整个程序分为四个部分：

![](.\images\总体结构.png)

## Commons

Commons.go中主要包括一些常量以及Worker和Coordinator都会用到的数据结构：

Task的类型包括Map、Reduce、Sleep以及所有任务执行完成ENDPROGRAM；

在Coordinator中保存着各个任务的状态信息，分为未分配、执行中、执行结束；

TaskStruct结构体中保存着Task的信息，用于在Worker和Coordinator通信时传输数据，包括任务的类型，任务的标号，任务需要读取的文件列表

```go
package mr

import (
	"log"
	"os"
)

type TASKTYPE int
type TASKSTATE int

//Ask task type
const (
	MAPTASK    TASKTYPE = 0		//执行Map函数
	REDUCETASK TASKTYPE = 1		//执行Reduce函数
	SLEEPTASK  TASKTYPE = 2		//没有任务分配，sleep后重新申请
	ENDPROGRAM TASKTYPE = 3		//程序执行完成，准备退出
)

//finish task state
const (
	FAIL    = 0
	SUCCESS = 1
)

//task state
const (
	UNLOCATED TASKSTATE = 0
	FINISHED  TASKSTATE = 1
	RUNNING   TASKSTATE = 2
)

const DIR = "./"

type TaskStruct struct {
	TaskType    TASKTYPE
	IndexOfTask int
	Files       []string
}
```

## RPC

Worker和Coordinator之间使用RPC通信，所以实验的难点之一在于通信数据结构的设计

这部分主要包括AskTask的Request 和Response；FinishTask的Request 和Response

```go
// Add your RPC definitions here.
type AskTaskRequest struct {
	State int
}

type AskTaskResponse struct {
	Task        TaskStruct		//执行的任务信息
	NumOfReduce int				//Reduce任务的总数
}

type FinishedTaskRequest struct {
	Task   TaskStruct			//执行的任务信息
	Result bool					//执行结果
}

type FinishTaskResponse struct {
	State int
}
```



## Worker

Worker这部分的执行逻辑比较单一，就是不断地向Coordinator请求任务，来不同的任务类型执行即可

![](.\images\lab1-Worker.jpg)

需要注意的一点是，在Reduce执行需要读取中间文件，这一步可以直接读取文件夹下那些文件符合条件，也可以通过在Coordinator中保存所有中间文件信息，并传输给Reducer。我这里选择的是后者，所以在Map执行结束后，将中间文件列表存储在Task.Files中传输回去。

## Coordinator

Coordinator的数据结构如下：

```go
type Coordinator struct {
	// Your definitions here.
	lock          sync.Mutex			//由于存在多个Worker同时访问，所以简单粗暴使用lock解决并发问题
    
    nMap, nReduce int					//Map和Reduce的任务数量
	maps, reduces []TASKSTATE			//维护着每个Map和Reduce任务的状态信息，UNLOCATED、RUNNING、FINISHED
		
	mapfiles      []string				//map阶段读取的文件列表，即输入参数
	interfiles    []string				//中间文件列表
	reducefiles   [][]string			//通过对中间文件分类到不同的Reduce任务的文件列表

	logFile   *os.File
	logger    *log.Logger				//日志工具
    
	taskChan  chan TaskStruct			//Task的队列，通过不断从队列中取任务
	taskPhase TASKPHASE					//当前程序执行到那个阶段：Map阶段、Reduce阶段、结束
}

type TASKPHASE int

const (
	MapPhase               TASKPHASE = 1
	ReducePhase            TASKPHASE = 2
	END                    TASKPHASE = 3
)
```



在Coodinator中主要函数可以分为以下三个部分：

### AskTask

```go
func (c *Coordinator) AskTask(args *AskTaskRequest, resp *AskTaskResponse) error
```

这部分就是接受来自Worker的请求，当程序还有未执行的Task时，然后返回一个Task给Worker

![](.\images\lab1-AskTask.jpg)

### FinishTask

```go
func (c *Coordinator) FinishTask(args *FinishedTaskRequest, resp *FinishTaskResponse) error	
```

当Worker执行完成一个Task后，调用FinishTask函数，在Coordinator中修改相应Task的状态即可。

- MapTask

  - 执行成功：

    1. 保存中间文件
    2. 修改任务状态为FINISH
    3. 判读是否所有Map执行完成，如果是进入Reduce阶段，并向TaskChan中追加ReduceTask

  - 执行失败

    ​	再次将该任务放入Taskchan中

- ReduceTask

  - 执行成功

    1. 修改任务状态为FINISH
    2. 判读是否所有Reduce执行完成，如果是进入END阶段

  - 执行失败

    再次将该任务放入Taskchan中

### CheckFault

```go
func checkFault(taskType TASKTYPE, indexOfTask int, c *Coordinator)
```

在实验的测试中包括一个crash的测试，改测试随机的退出一些Map或者Reduce，这是就需要Coordinator在一定时间后如果该任务还未完成就分配给一个新的Worker。

这里我使用的是定时器的方式，每个任务在Asktask函数中下发后，调用go checkFault函数启动一个定时器，10s检查该任务状态是否完成，如果没有，就重新放入队列中，并修改任务状态为未分配状态。

```go
timer1 := time.NewTimer(10 * time.Second)
<-timer1.C
```



# 遇到的问题

## 1. 运行示例程序报错：fatal error: stdlib.h: 没有那个文件或目录

参考官方文档http://nil.csail.mit.edu/6.824/2021/labs/lab-mr.html中的步骤，首先clone实验的框架。

运行示例程序wc.go

```shell
$ cd ~/6.824
$ cd src/main
$ go build -race -buildmode=plugin ../mrapps/wc.go
$ rm mr-out*
$ go run -race mrsequential.go wc.so pg*.txt
$ more mr-out-0
```

在运行`go build -race -buildmode=plugin ../mrapps/wc.go`时**报错如下**：

```shell
# runtime/cgo
_cgo_export.c:3:10: fatal error: stdlib.h: 没有那个文件或目录
    3 | #include <stdlib.h>
      |          ^~~~~~~~~~
compilation terminated.
```

参考网上需要运行：

```shell
sudo apt-get install build-essential
```

但又出现报错：

```shell
The following packages have unmet dependencies
```

又不得不安装aptitude包管理软件

**解决方法如下：**

```shell
# 1. 修改ubuntu源为本身自带的源地址（阿里源或者其它都可能无法安装build-essential）
# 2. 安装aptitude包管理软件
# 3. 安装build-essential
# 4. 运行go build -race -buildmode=plugin ../mrapps/wc.go即可
```



## 2.crash测试

虽然在程序中有检查Worker退出的情况，但是测试依然不通过，crash测试的源码如下：

```shell
SOCKNAME=/var/tmp/824-mr-`id -u`

( while [ -e $SOCKNAME -a ! -f mr-done ]
  do
    timeout -k 2s 180s ../mrworker ../../mrapps/crash.so
    sleep 1
  done ) &

( while [ -e $SOCKNAME -a ! -f mr-done ]
  do
    timeout -k 2s 180s ../mrworker ../../mrapps/crash.so
    sleep 1
  done ) &

while [ -e $SOCKNAME -a ! -f mr-done ]
do
  timeout -k 2s 180s ../mrworker ../../mrapps/crash.so
  sleep 1
done

wait

# 新加两个wait
# wait
# wait
```

经过查询资料这段代码的意思是维护有三个Worker在工作，但是调试得知同时只有一个在并行，我这里用的是2021年的代码，查询2020年的测试得知最后的wait有三个，应该是这里测试代码出问题了。只需要加上两个wait即可。














