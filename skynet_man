6 服务调度
local skynet = require "skynet"
--让当前的任务等待 time * 0.01s 。
skynet.sleep(time)  
​
--启动一个新的任务去执行函数 func , 其实就是开了一个协程，函数调用完成将返回线程句柄
--虽然你也可以使用原生的coroutine.create来创建协程，但是会打乱skynet的工作流程
skynet.fork(func, ...) 
​
--让出当前的任务执行流程，使本服务内其它任务有机会执行，随后会继续运行。
skynet.yield()
​
--让出当前的任务执行流程，直到用 wakeup 唤醒它。
skynet.wait()
​
--唤醒用 wait 或 sleep 处于等待状态的任务。
skynet.wakeup(co)       
​
--设定一个定时触发函数 func ，在 time * 0.01s 后触发。
skynet.timeout(time, func)  
​
--返回当前进程的启动 UTC 时间（秒）。
skynet.starttime()  
​
--返回当前进程启动后经过的时间 (0.01 秒) 。
skynet.now()
​
--通过 starttime 和 now 计算出当前 UTC 时间（秒）。
skynet.time()

6.1 使用sleep休眠
local skynet = require "skynet"
​
skynet.start(function ()
    skynet.error("begin sleep")
    skynet.sleep(500)   
    skynet.error("begin end")
end)
运行结果(还是先运行main.lua)：
$ ./skynet examples/config
testsleep                       #输入testsleep
[:01000010] LAUNCH snlua testsleep
[:01000010] begin sleep
#执行权限已经被第一个服务testsleep给使用了，这里输入个新的服务test，并不会马上就启动服务
test        
[:01000010] begin end       
[:01000012] LAUNCH snlua test  #等第一个服务完成任务了，才启动新的服务
[:01000012] My new service
在console服务中输入testsleep之后，马上再输入test，会发现，test服务不会马上启动，因为这个时候 console正在忙于第一个服务testsleep初始化，需要等待5秒钟之后，输入的test 才会被console处理。

6.2 在服务中开启新的线程
​ 在skynet的服务中我们可以开一个新的线程来处理业务（注意这个的线程并不是传统意义上的线程，更像是一个虚拟线程，其实是通过协程来模拟的）。

示例代码: testfork.lua

local skynet = require "skynet"
​
function task(timeout)
    skynet.error("fork co:", coroutine.running())
    skynet.error("begin sleep")
    skynet.sleep(timeout)
    
    skynet.error("begin end")
end
​
skynet.start(function ()
    skynet.error("start co:", coroutine.running())
    skynet.fork(task, 500)  --开启一个新的线程来执行task任务
    --skynet.fork(task, 500)  --再开启一个新的线程来执行task任务
end)
运行结果：

$ ./skynet examples/config
testfork                        #输入testfork
[:0100000a] LAUNCH snlua testfork   
[:0100000a] start thread: 0x7f684d967568 false #不同的两个协程
[:0100000a] fork  thread: 0x7f684d969f68 false
[:0100000a] begin sleep
​ 可以看到在testfork启动后，consloe服务仍然可以接受终端输入的test，并且启动。

​ 以后如果遇到需要长时间运行，并且出现阻塞情况，都要使用skynet.fork在创建一个新的线程(协程)。

 

 

查看源码skynet.lua了解底层实现，其实就是使用coroutine.create实现

​ 每次使用skynet.fork其实都是从协程池中获取未被使用的协程，并把该协程加入到fork队列中，等待一个消息调度，然后会依次把fork队列中协程拿出来执行一遍，执行结束后，会把协程重新丢入协程池中，这样可以避免重复开启关闭协程的额外开销。

 

6.3 长时间占用执行权限的任务
示例代码：busytask.lua

local skynet = require "skynet"
​
function task(name)
    local i = 0
    skynet.error(name, "begin task")
    while ( i < 200000000)
    do
        i = i+1
    end
    skynet.error(name, "end task", i)
end
​
skynet.start(function ()
    skynet.fork(task, "task1")
    skynet.fork(task, "task2")
end)
运行结果：

$ ./skynet examples/config
busytask
[:01000010] LAUNCH snlua busytask
[:01000010] task1 begin task       --先运行task1
[:01000010] task1 end task 200000000
[:01000010] task2 begin task        --再运行task2
[:01000010] task2 end task 200000000
​
​ 上面的运行结果充分说明了，skynet.fork 创建的线程其实通过lua协程来实现的，即一个协程占用执行权后，其他的协程需要等待。

6.4 使用skynet.yield让出执行权
示例代码：testyield.lua

local skynet = require "skynet"
​
function task(name)
    local i = 0
    skynet.error(name, "begin task")
    while ( i < 200000000)
    do
        i = i+1
        if i % 50000000 == 0 then
            skynet.yield()
            skynet.error(name, "task yield")
        end
    end
    skynet.error(name, "end task", i)
end
​
skynet.start(function ()
    skynet.fork(task, "task1")
    skynet.fork(task, "task2")
end)
运行结果：

$ ./skynet examples/config
testyield
[:01000010] LAUNCH snlua testyield
[:01000010] task1 begin task
[:01000010] task2 begin task
[:01000010] task1 task yield
[:01000010] task2 task yield
[:01000010] task1 task yield
[:01000010] task2 task yield
[:01000010] task1 task yield
[:01000010] task2 task yield
[:01000010] task1 task yield
[:01000010] task1 end task 200000000
[:01000010] task2 task yield
[:01000010] task2 end task 200000000
​ 通过使用skynet.yield() 然后同一个服务中的不同线程都可以得到执行权限。

 

6.5 线程间的简单同步
​ 同一个服务之间的线程可以通过，skynet.wait以及skynet.wakeup来同步线程

示例代码：testwakeup.lua

local skynet = require "skynet"
local cos = {}
​
function task1()
    skynet.error("task1 begin task")
    skynet.error("task1 wait")
    skynet.wait()               --task1去等待唤醒
    --或者skynet.wait(coroutine.running())
    skynet.error("task1 end task")
end
​
​
function task2()
    skynet.error("task2 begin task")
    skynet.error("task2 wakeup task1")
    skynet.wakeup(cos[1])           --task2去唤醒task1，task1并不是马上唤醒，而是等task2运行完
    skynet.error("task2 end task")
end
​
​
skynet.start(function ()
    cos[1] = skynet.fork(task1)  --保存线程句柄
    cos[2] = skynet.fork(task2)
end)
运行结果：

$ ./skynet examples/config
testwakeup
[:01000010] LAUNCH snlua testwakeup
[:01000010] task1 begin task
[:01000010] task2 begin task
[:01000010] task1 wait
[:01000010] task2 wakeup task1
[:01000010] task2 end task
[:01000010] task1 end task
 

需要注意的是：skynet.wakeup除了能唤醒wait线程，也可以唤醒sleep的线程

 

 

6.6 定时器的使用
​ skynet中的定时器，其实是通过给定时器线程注册了一个超时时间，并且占用了一个空闲协程，空闲协程也是从协程池中获取，超时后会使用空闲协程来处理超时回调函数。

​

6.6.1 启动一个定时器
示例代码：testtimeout.lua

local skynet = require "skynet"
​
function task()
    skynet.error("task", coroutine.running())
end
​
skynet.start(function ()
     skynet.error("start", coroutine.running())
    skynet.timeout(500, task) --5秒钟之后运行task函数,只是注册一下回调函数，并不会阻塞
end)
运行结果：

$ ./skynet examples/config
testtimeout
[:01000010] LAUNCH snlua testtimeout
[:01000010] task
 

6.6.2 skynet.start源码分析
​ 其实skynet.start服务启动函数实现中，就已经启动了一个timeout为0s的定时器，来执行通过skynet.start函数传参得到的初始化函数。其目的是为了让skynet工作线程调度一次新服务。这一次服务调度最重要的意义在于把fork队列中的协程全部执行一遍。

6.6.3 循环启动定时器
local skynet = require "skynet"
​
function task()
    skynet.error("task", coroutine.running())
    skynet.timeout(500, task)
end
​
skynet.start(function ()  --skynet.start启动一个timeout来执行function，创建了一个协程
     skynet.error("start", coroutine.running())
    skynet.timeout(500, task) --由于function函数还没用完协程，所有这个timeout又创建了一个协程
end)
 

​ 执行结果，交替使用协程池中的协程：

testtimeout
[:0100000a] LAUNCH snlua testtimeout
[:0100000a] start thread: 0x7f525b16a048 false  #start函数也执行完，这个协程就空闲下来了
[:0100000a] task thread: 0x7f525b16a128 false   #当前服务的协程池中只有两个协程，所以是交替使用
[:0100000a] task thread: 0x7f525b16a048 false
[:0100000a] task thread: 0x7f525b16a128 false
[:0100000a] task thread: 0x7f525b16a048 false
 

6.7 获取时间
示例代码：testtime.lua

local skynet = require "skynet"
​
function task()
    skynet.error("task")
    skynet.error("start time", skynet.starttime()) --获取skynet节点开始运行的时间
    skynet.sleep(200)
    skynet.error("time", skynet.time())     --获取当前时间
    skynet.error("now", skynet.now())       --获取skynet节点已经运行的时间
end
​
skynet.start(function ()
    skynet.fork(task)
end)
 

运行结果：

$ ./skynet examples/config
testtime
[:01000010] LAUNCH snlua testtime
[:01000010] task
[:01000010] start time 1517161846
[:01000010] time 1517161850.73
[:01000010] now 473
6.8 错误处理
​ lua中的错误处理都是通过assert以及error来抛异常，并且中断当前流程，skynet也不例外，但是你真的懂assert以及error吗？

​ 注意这里的error不是skynet.error，skynet.error单纯写日志的，并不会中断流程。

​ skynet中使用assert或error后，服务会不会中断，skynet节点会不会中断？

来看一个例子：

testassert.lua

local skynet = require "skynet"
​
function task1()
    skynet.error("task1", coroutine.running())
    skynet.sleep(100)
    assert(nil)
    --error("error occurred")
    skynet.error("task2", coroutine.running(), "end")
end
​
function task2()
    skynet.error("task2", coroutine.running())
    skynet.sleep(500)
    skynet.error("task2", coroutine.running(), "end")
end
​
skynet.start(function ()  
    skynet.error("start", coroutine.running())
    skynet.fork(task1) 
    skynet.fork(task2) 
end)
运行结果：

testassert
[:0100000a] LAUNCH snlua testassert
[:0100000a] start thread: 0x7fd9b12938e8 false
[:0100000a] task1 thread: 0x7fd9b12939c8 false  
[:0100000a] task2 thread: 0x7fd9b1293aa8 false
[:0100000a] lua call [0 to :100000a : 2 msgsz = 0] error : ./lualib/skynet.lua:534: ./lualib/skynet.lua:156: ./my_workspace/testassert.lua:6: assertion failed!
stack traceback:
    [C]: in function 'assert'
    ./my_workspace/testassert.lua:6: in function 'task1'
    ./lualib/skynet.lua:468: in upvalue 'f'
    ./lualib/skynet.lua:106: in function <./lualib/skynet.lua:105>
stack traceback:
    [C]: in function 'assert'
    ./lualib/skynet.lua:534: in function 'skynet.dispatch_message'
[:0100000a] task2 thread: 0x7fd9b1293aa8 end
​
​ 上面的结果已经很明显了，开了两个协程分别执行task1、task2，task1断言后终止掉当前协程，不会再往下执行，但是task2还是能正常执行。skynet节点也没有挂掉，还是能正常运行。

​ 那么我们在处理skynet的错误的时候可以大胆的使用assert与error，并不需要关注错误。当然，一个好的服务端，肯定不能一直出现中断掉的协程。

如果不想把协程中断掉，可以使用pcall来捕捉异常，例如：

local skynet = require "skynet"
​
function task1()
    skynet.error("task1", coroutine.running())
    skynet.sleep(100)
    assert(nil)
    skynet.error("task2", coroutine.running(), "end")
end
​
function task2()
    skynet.error("task2", coroutine.running())
    skynet.sleep(500)
    skynet.error("task2", coroutine.running(), "end")
end
​
skynet.start(function ()  
    skynet.error("start", coroutine.running())
    skynet.fork(pcall, task1) 
    skynet.fork(pcall, task2) 
end)
运行结果：

testassert
[:0100000a] LAUNCH snlua testassert
[:0100000a] start thread: 0x7f93d3967568 false
[:0100000a] task1 thread: 0x7f93d3967aa8 false
[:0100000a] task2 thread: 0x7f93d3967b88 false
[:0100000a] task2 thread: 0x7f93d3967b88 end




