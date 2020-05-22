# go

## 基础知识

+ map，slice，channel属于引用类型，make返回的是Type，new返回的是指针类型的Type（即\*Type），new也可以创建map、slice、channel，但是返回的是对应的指针，如果想要使用，需要通过取指针的值`*`来进行使用；

+ defer 
    defer的如果是个函数，必须先确定函数参数，例如函数中的参数也是由另一个函数执行的结果，则必须执行完成，再继续；当defer压栈时，参数全部确定，并入栈，后续即使值变了，这个defer中的参数也不会改变；

+ slice

    ```golang
    type sliceHeader struct {
        Data unsafe.Pointer
        Len  int
        Cap  int
    }
    ```
    结构体为，cap，len，unsafe.Pointer;其中Data为指向数组的指针；

    注意`[]int{1,2,3}`是slice，`[4]int{1,2,3}`是数组，`array[1:2:3]`表示从下标为1开始切，到下标2为止，cap为3-1，注意顺序必须是`[a<=b<=c]`；

    slice扩容策略，容量小于1024，变成2倍，否则1.25倍；注意尽量避免对长度为4的array取[0:2]小于长度的slice操作，这样append后的长度不大于4时，原数组的大小 仍可用，append就直接修改源数组了，直接修改也是相同；range slice时，拿到的value是数组中值的拷贝，所以必须使用下标的方式进行修改；

+ string 
  
    结构体
```golang
    type stringStruct struct {
        str unsafe.Pointer
        len int
    }
```
str指向一个byte数组，len就是长度了，str是无法更改的；
在byte[]转string时，并不会拷贝，会直接返回一个string结构体，这个string的指针指向那个字节数组;

+ hash

  hash最重要的实现点在于hash映射与hash冲突；

    + hash函数：<br>
    hash函数的实现很大程度上决定了hash表的读写性能；理想状态下，hash应该将不同的键映射到不同的索引上，这要求hash输出的范围要大于输入的范围，但是由于键的范围远大于映射的范围，所以理想状态几乎不可能实现；<br>
    通过算法可以避免hash碰撞的问题，但是要hash尽可能的分布均匀，不均匀会造成较差的写入是的碰撞与读取时的性能；<br>
    hash函数与索引下标的关系：index:= hash(k)%(len(array)-1)；<br>

    + hash冲突：键的范围是远大于hash映射的范围的，所以hash冲突必不可免，为了解决hash冲突，有两种实现方法，1.开放寻址法，2.链地址法；
        1. 开放寻址法：kv最终是存储在数组中的，当通过hash获取到索引下标后，发现当前索引对应的kv已存在，则将当前kv写入下一个不为空的索引中index+n；查找时，如果发现当前k与得到的k不匹配，则往下遍历，直至找到或者遍历完数组；开放寻址法的问题在于，随着kv的存储，hash的空间越来越少，冲突时需要遍历的时间越来越长，性能急剧下降，最终写满hash后，hash失效；
        2. 链地址法：链地址法使用类似开放寻址法的方法，但是不再是往当前存储kv的数组中写，而是写入一个新的kv，每个kv节点包含指向下一个同hash的kv，也就是hash数组中，每个index对应的都会是一个链表，这种方式解决了开放寻址法的空间耗尽的导致失效的问题；

    go的hash结构：

    ```golang
    type hmap struct {
        count     int
        flags     uint8
        B         uint8
        noverflow uint16
        hash0     uint32

        buckets    unsafe.Pointer
        oldbuckets unsafe.Pointer
        nevacuate  uintptr

        extra *mapextra
    }
    ```

    count：当前hash的元素个数<br>
    B：当前hash中buckets的数量，但是因为hash中buckets的数量都是2的倍数，所以该字段会储存对数，len(buckets) == 2^B<br>
    hash0：hash的种子，它为hash值导入随机性，在创建hash时确定；它作为hash函数的一个参数；<br>
    oldbuckets：在hash扩容时，保存扩容前的buckets字段；


+ iota，iota可以理解为常亮的索引下标，在const出现时置为0，后续增加一条常量定义，则iota+= 1，不管你从哪开始使用iota；当使用其他值的时候，iota计数仍存在，但不使用，直到重新使用；例如：
  
    ```golang
        const (
            a = "?"  // iota = 0, a = "?"
            b = iota // iota += 1, b = 1
            c        // iota += 1, c = 2
            d = "ha" // 独立值，iota += 1, d = "ha"
            e        // 不表明默认继承上一个变量的定义 iota += 1, e = "ha"
            f = 100  // iota +=1 f = 100
            g = iota // 恢复使用iota, iota +=1 g = 6
            h        // iota +=1 h = 7
            i        // iota +=1 i = 8
        )
    ```

+ type A struct{} 定义的结构体属于struct种类，*A则属于ptr

+ context

    context用于控制子程序的退出，用于多个Goroutine或者模块之间同步取消信号或者截止时间，用于减少对资源的消耗和长时间占用，避免资源浪费；虽然传值也是它的功能之一，但是使用场景很少，使用时需谨慎


## 指针真的比值传高效吗？

传指针与传值很好理解，一个是完全的副本，一个是内存地址的副本（go的传参全部都是传值，指针也是传值，不过值是具体数据在内存中的地址，通过地址再找到对应内存中的数据），但是并不能直接说那个是完全的更高效；

指针的大小根据不同的架构有不同的大小，x32为32bit，x64为64bit；正常情况下，指针的大小比中大型结构体要小，比float，bool等标量类型大；表面看来，大部分情况下使用指针比传值要有更好的性能，但是性能涉及的因素太多，不能随便下定论；

linux下的进程拥有栈和堆，用来存放变量；栈为函数局部空间，当函数执行完成后，直接释放；堆则是需要gc进行释放；编译器会对变量进行逃逸分析，来决定该变量分配到堆还是栈；当出现变量共享时，则会发生逃逸，栈空间不足以使用时，发生逃逸，动态类型会发生逃逸（参数类型为interface，典型的fmt.P系列）；当使用传值的时候，若该变量大小在栈允许的大小范围内，则分配到栈上，否则，分配到堆；当是引用类型时，既指针，会分配到堆上；

很明显，栈上分配的内存有着更高的效率，栈不需gc，函数调用完成后自动回收；当变量发生逃逸时，会分配到堆上；如果堆上的数据过多，则会影响gc的效率；


## csp并发模型

csp：两个独立的并发实体通过共享的通讯管道进行通信的并发模型；go借用了这个思想，实体之间通过channel进行通信来实现数据共享；

go推荐仅管内存同步访问控制，通过管理同步访问的顺序，来管理内存，而不是通过锁来控制内存来同步访问；

Goroutine 和 channel 是 Go 语言并发编程的 两大基石。goroutine 用于执行并发任务，channel 用于 goroutine 之间的同步、通信；

go里面最经典的一句话：不要通过共享内存来通信，而要通过通信来实现内存共享。前面半句说的是通过 sync 包里的一些组件进行并发编程；而后面半句则是说 Go 推荐使用 channel 进行并发编程。两者其实都是必要且有效的。实际上看完本文后面对 channel 的源码分析，你会发现，channel 的底层就是通过 mutex 来控制并发的。只是 channel 是更高一层次的并发编程原语，封装了更多的功能；

channel是实现csp模型的重要点；不能往关闭channel里面写数据，写了会panic，但是可以接收，但是会收到当前chan的0值


## gpm调度模型与策略

+ 系统的进程、线程
    系统调度的最小单位是线程；在操作系统中，进程是程序运行时的一个类似容器的环境，它包含了程序运行时所需要的各种资源，包括内存、线程、句柄；而线程则是具体的执行单位，他会被系统调度，来执行各种处理；
    goroutine对系统而言，只是一个用户级的线程，系统甚至并不知道goroutine的存在，goroutine都是依靠调度器调度到对应的系统线程上来执行的；

+ goroutine的优势在于上下文切换在完全用户态进行，无需像线程一样频繁在用户态与内核态之间切换，节约了资源消耗;

+ GPM模型则是：
    + G:goroutine go协同程序，轻量的用户级线程；每个G存储着goroutine的运行堆栈、状态和任务函数；goroutine的栈不是系统分配的，是go进行动态分配的，初始为2k，会根据使用动态伸缩；
    + P:processor逻辑处理器，是G和M的调度者，负责处理M与G之间的关系，可通过runtime.GOMAXPROCS(int)来设置，runtime.NumCPU()获取cpu数量，一般联合起来用；
    + 每个P拥有自己的LRQ（local run queue）存储用户创建的goroutine对象，未分配的G存放在GRQ（global run queue）中，刚创建的goroutine都在GRQ中，等待分配到某个P的LRQ中；使用P作为桥梁，连接G与M，也是解耦了；G只有绑定到P后才能被调度；对于M来说，P提供了运行的上下文环境context、内存分配状态、任务队列LRQ；P的数量决定了可并行运行的G的数量
    + M:machine，内核级线程的封装，所有的goroutine都是跑在machine上的，任务都是M来做的；

epoll是个啥？

## gc的方式和性能影响

+ 常用的gc方式
    + 引用计数
    + 标记清除 
    + 复制收集 
    + 分代收集

+ 三色标记：白色:垃圾，灰色:待处理，黑色:正常的对象
    1. 起初，所有的对象都是白色的；
    2. 从根出发，扫描所有可以到达的对象，标记为灰色，放入待处理队列；
    3. 从队列中取出灰色对象，将其引用的对象标记为灰色，并放入队列中，自身标记为黑色；
    4. 重复3步骤，直到灰色队列为空，此时，白对象就是无引用的垃圾，对此进行回收；

    ![三色标记](https://github.com/shhch/library/releases/download/1.0/IMG_0850.GIF)

+ go的gc历史：
    + <=1.3: 标记清除，stw；
    + 1.3: 标记清除的升级版，标记与清除分开执行，减少stw的时间；
    + 1.5: 三色标记法；
    + 1.8: 混合写屏障；

## sync包

+ 常用：mutex，rwmutex，waitgroup

+ mutex的实现原理（互斥所，等待协程自旋）

    mutex有两个变量，stat表状态，他是int32的，waiter的数量，woken与starving用来控制协程争抢锁的过程，locked表示锁的状态；抢锁就是抢着给locked赋值的权利；
    ```
       29位     1位       1位       1位
    | waiter | woken | starving | locked |
    ```

    直接抢锁成功，locked=>1，解锁locked=>0；抢失败，waiter++，协程被阻塞，等待唤醒（locked=0）；<br>
    解锁时，先locked=0，若waiter!=0，则waiter--，释放一个信号量，唤醒一个协程；<br>

    如果唤醒的过程中被其他自旋的协程抢了，那么会再次自旋；可以防止加锁失败时，协程直接进入阻塞，有一定机会抢到锁；<br>
    自旋对应cpu的pause指令，cpu对该指令什么都不做，相当于空转，但是会消耗cpu，对程序而言相当于sleep了，但是不是sleep状态，自旋期间会持续检测locked是否为0，两次检测期间，执行的则是pause指令；

    当大量的协程自旋时，会对cpu有较大的消耗，所以需要对自旋进行限制：

    1. 自旋次数要少，一般4次；
    2. cpu是多核，协程本身只是通过调度器跑在一个或多个核上的，只有一个核的时候，同时只能有一个协程在工作；对应的，单核情况下其他协程不可能释放锁；
    3. Processor数量要大于1，和第二条是类似的问题；
    4. 协程的可运行调度队列要为空，否则会延迟其他协程调度；
    由此可见，协程的自旋必须是cpu足够空闲时才行，可以充分利用闲余的cpu，但在cpu紧张时，不会占用cpu资源；

    自旋也会造成一个问题，会导致某些被阻塞的协程一直无法获得锁，从而进入饥饿状态starving，1.8版本后增加starving状态来避免这种情况，一旦进入此状态，那么将不会进行自旋，释放锁后，一定会进行唤醒另一个协程并加锁成功；该状态分为两种：<br>
    normal模式：starving为0，协程抢锁失败后，不会立即转入阻塞排队，会判断是否满足自旋，满足则进入自旋，否则阻塞；<br>
    starvation模式：当锁释放时，还会释放一个信号来唤醒一个阻塞协程，该阻塞协程在被cpu唤醒到抢锁的过程内，被其他自旋的抢了，则会再次阻塞，但是在阻塞前会判断阻塞的时间，如果超过了1ms，那么将starving状态设为1；此状态下无自旋，饥饿的协程在被唤醒抢锁后，会将starving--；

    woken状态用于加锁与解锁之间的通信，当同一时刻，加锁解锁同时进行，加锁的协程可能处于自旋状态，此时，把woken=1，通知解锁的进程不比释放唤醒信号，直接解锁就可以了；

    重复解锁会panic


## grpc

grpc是http2，http2默认并不加密，但是定义了会话层TLS，与https类型，SSL/TLS 分为单向认证和双向认证；基本思路是采用公钥加密法，客户端向服务器端索要公钥，然后用公钥加密信息，服务器收到密文后，用自己的私钥解密；
1. 客户端向服务端传送客户端 SSL 协议的版本号、支持的加密算法种类、产生的随机数，以及其它可选信息；
2. 服务端返回握手应答，向客户端传送确认 SSL 协议的版本号、加密算法的种类、随机数以及其它相关信息；
3. 服务端向客户端发送自己的公钥；
4. 客户端对服务端的证书进行认证，服务端的合法性校验包括：证书是否过期、发行服务器证书的 CA 是否可靠、发行者证书的公钥能否正确解开服务器证书的“发行者的数字签名”、服务器证书上的域名是否和服务器的实际域名相匹配等；
5. 客户端随机产生一个用于后面通讯的“对称密码”，然后用服务端的公钥对其加密，将加密后的“预主密码”传给服务端；
6. 服务端将用自己的私钥解开加密的“预主密码”，然后执行一系列步骤来产生主密码；
客户端向服务端发出信息，指明后面的数据通讯将使用主密码为对称密钥，同时通知服务器客户端的握手过程结束；
7. 服务端向客户端发出信息，指明后面的数据通讯将使用主密码为对称密钥，同时通知客户端服务器端的握手过程结束；
8. SSL 的握手部分结束，SSL 安全通道建立，客户端和服务端开始使用相同的对称密钥对数据进行加密，然后通过 Socket 进行传输。


+ metadata，可以理解为一次rpc调用中额外的参数，k-v格式，通过以下方式调用

    客户端
    ```golang
    // 客户端生成和发送md
    md := metadata.New(map[string]string{"hello": "world", "foo": "bar"})
    // 新建一个有 metadata 的 context
    ctx := metadata.NewOutgoingContext(context.Background(), md)
    // 单向 RPC
    response, err := client.SomeRPC(ctx, someRequest)
    ```

    服务端
    ```golang
    // 服务器接收md
    md, ok := metadata.FromIncomingContext(ctx)
    ```
    

## gin

+ gin中间件，强大的功能，可作为前置或后置拦截（过滤）器，路由分组处理、请求前后header处理，认证等

    gin.Default会自行使用Logger和Recovery两个中间件，所以需要gin.New()来自行添加中间件；通过router := gin.New()之后的router.Use(gin.Logger())添加多个中间件（这样添加的是全局中间件），中间件需要返回 gin.HandlerFunc 函数，所以定义返回函数；

    定义中间件函数：
    ```golang
    func Auth() gin.HandlerFunc {
        return func(context *gin.Context) {
            // 这里处理权限判断逻辑
            println("已经授权")
            context.Next()
        }
    }

    // 指定使用中间件，然后在router中使用，它会按照顺序进行执行
    router.GET("/profile/", Auth(),gin.Logger(), handler.UserProfile)

    // 分组使用
    router := gin.New()
    user := router.Group("user", gin.Logger(),gin.Recovery())
    {
        user.GET("info", func(context *gin.Context) {
            context.Set("key","lalalal")
        })
        user.GET("article", func(context *gin.Context) {
            k,ok := context.Get("key")
        })
    }
    ```

## beego

+ 环境变量配置

    beego支持从环境变量中读取配置信息，配置方式为`key = ${env_name||default}`，获取时，使用beego.AppConfig.String("key")即可；

    如果env_name环境变量不存在，则使用default，环境变量的优先级是高于默认值的，他们之间使用`||`分隔；

+ beego过滤器，类似gin中间件，但是没那么灵活
  
    ```golang
    beego.InsertFilter(pattern string, postion int, filter FilterFunc, params ...bool)
    ```

    InsertFilter 函数的三个必填参数，一个可选参数;
    + pattern 路由规则，可以根据一定的规则进行路由，如果你全匹配可以用 *
    + postion 执行 Filter 的地方，四个固定参数如下，分别表示不同的执行过程
        1. BeforeStatic 静态地址之前，session的获取需要在此之后
        2. BeforeRouter 寻找路由之前
        3. BeforeExec 找到路由之后，开始执行相应的 Controller 之前
        4. AfterExec 执行完 Controller 逻辑之后执行的过滤器
        5. FinishRouter 执行完逻辑之后执行的过滤器
    + filter 执行函数 type FilterFunc func(*context.Context)
    + params 多个bool参数判断
        1. 第一个，是否允许如果有输出是否跳过其他filters，flase允许执行多个filter，默认true；
        2. 第二个，是否重置filters的参数，默认是false；因为在filters的pattern和本身的路由的pattern冲突的时候，可以把filters的参数重置

+ beego路由

    + 自动路由<br>
    在router的初始化init中添加`beego.AutoRouter(&admin.UserController{})`其中controller为需要自动路由的控制器，一次只能添加一个，需要多个则要多次调用；匹配规则为Controller前面的名+函数名、例如UserController下的login函数，路由为/user/login，路由为全小写；

    + 固定路由<br>
    使用`beego.Router("url",&Controller{},"post,get:functionName")`，第一个参数为路由的url，第二个参数为控制器，第三个参数分两部分，前半部分为请求方式，多个请求方式用`,`隔开，同时支持使用`*`来表示所有请求方式，后半部分为对应的函数名，前半部分与后半部分使用`:`分割；

    + 正则路由<br>
    固定路由是正则路由的全匹配类型，所以他们的使用方式相同`beego.Router("url",&Controller,"post,get:functionName")`；

        示例：
        + url:"/api/?:id"，可以匹配/api/***，同时可以通过c.Ctx.Input.Param(":id")获取对应的值（c为对应controller）
        + url:"/api/:id"，与上面类似，但是/api/会失败
        + url:"/api/:id([0-9]+)"，可匹配一个或多个数字
        + TODO 正则后面再说...

    + 注解路由<br>
    通过`beego.Include(&Controller{})`来导入注解路由，一次可添加多个控制器；<br>
    同时在对应controller的函数上方添加注解，例如：
        ```golang
        // @router /admin/index/
        func (this *UserController) Index() {
            this.Ctx.WriteString("这是注释路由 /admin/index")
        }
        ```

        导入后，还需要重新自动生成对应的router文件，需要执行`bee run`，bee为beego客户端
    
    + namespace


