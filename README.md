#轻量级webserver服务器（附个人理解）
## 下面是我的思路，各位读者可以选择观看，帮助您阅读源码：（底部附带我认为可以改进的地方以及代码中的不足之处）

webserver作为小父亲（无血缘关系，只是为了更好理解），封装了调用各个类的函数调用。（更大的父亲是main，不过他只是调用webserver的函数）

本次代码中，每一个http连接（当epoll监听到有数据上来的时候，会将一个http_conn对象放进线程池的任务队列）都会在逻辑上绑定一个定时器timer，保证当一个http连接长时间没有读写操作的时候直接从epoll上面摘除。这个定时器根据时间戳在一个链表上按升序排序，每一次对应的http连接有读写操作的时候都调整一下这个时间戳（给时间戳延长3*TIME的时间），并且调整它在链表上对应的位置，每经过一次timeout都去链表查看一下时间戳（如果时间戳比当前时间小就丢弃）。也封装了一个阻塞队列（通过互斥锁和条件变量实现），以及一个锁的类（有互斥锁，条件变量，信号量）

​       <u>首先</u>创建webserver对象，自然调用构造函数，先初始化大量的http_conn类（主要作用就是对通信套接字的读，解析，写回）以及client_data类（**主要成员是为了指向定时器，定时器并没有直接绑定读写的http_conn类，而是通过具有唯一性的套接字在逻辑上 进行绑定，因为套接字本质是文件描述符，而且在一个进程中是唯一的，所以可以通过一个套接字标识通讯的http_conn类以及定时器，比如说user[fd]和timer[fd]，虽然没有进行绑定，但是根据这个套接字就可以准确的操作有联系user和timer，使得他们在逻辑上具有绑定性**）。

还初始化了一个工具类Utils（里面封装了一系列函数，比如把套接字挂树上(epoll)，设置信号函数以及信号的回调函数，还有设置fd非阻塞等）

<u>接着对log日志初始化</u>（单例模式—->后面有对这个实现的讲解）通过实例化对象调用init初始化阻塞队列并创建子线程，记录当前时间，并且以追加的方式打开从webserver传来的日志路径加文件名字（**把当前日期加在日志名中作为日志的真实路径和名字**）。通过子线程回调函数是准备从阻塞队列中拿取客户记录的日志（**异步模式，同步模式是直接向日志中写**），当有客户向阻塞队列中添加任务后，解除阻塞，加锁向日志文件中写入（为何加锁，不是单例模式而且只有一个子线程吗？—–>日志支持异步或者同步，当异步模式向阻塞队列中加入日志信息，但队列满了，需要直接向文件写入而不是继续向队列中写入，然而子线程也在向文件写入，此时就需要加锁维护，保证只有一个线程向文件中写入。）——–>当客户每次向日志中添加内容之前都要检查当前日期是否不同和日志行数是否超限，是的话就要重新创建新的日志。

 <u>然后是对数据库连接池初始化</u>（单例模式）通过实例化对象调用init，将初始化一个数据库连接的许多参数传入进来，初始化一定数量的MYSQL连接，加入到list容器中，并且给将实例化对象传给每一个http连接，可以从连接池中拿取一个数据库连接，提前将数据库中的用户和密码提取到hhtp类的map中，节省后期用户开销。

<u>接下来时对线程池的初始化</u>，通过构造函数初始化一定数量的子线程，子线程阻塞等待任务队列。

<u>接着是监听的初始化</u>，创建监听套接字，然后绑定IP和端口，初始化一个epoll实例（底层数据结构是红黑树，所以我喜欢喊它树），接着把套接字挂树上（可设置LT或ET+非阻塞)，创建一对匿名套接字（当信号触发的时候通过套接字传送），将一端挂树上，设置捕捉函数，来信号的时候捕捉下来并且写进另一端。

<u>最后就开始循环监听并且对任务进行读写操作</u>

1.如果是有新客户连接，accept一个客户端fd，给它加定时器并且挂树上（这里很关键，因为是通过调用timer函数，此时初始化的是timer[fd]的时间戳，把fd传进了user[fd]，所以当epoll监听到fd的时候我们就可以把user[fd]丢进线程池的任务队列【**并且给它里面大部分数据重置**—->通过init函数实现】，而不是监听到一个fd的任务请求不知道把哪个user即http_conn类放进任务队列），并且给定时器在链表上排序。如果连接数量超过了上限就写回“buzy”，关闭套接字。

2.如果对端关闭，删除定时器，从链表上取下，从树上取下。

3.如果是匿名套接字，并且是读事件，从套接字读出来，如果是SIGALRM，查询一遍定时器链表，把超时的取下来，并且从epoll上取下。

4.如果是客户端读事件，先通过套接字找到对应timer，然后修改timer时间戳，重新在链表上排序。

客户端fd是LT模式：将user[fd]，放进线程池任务队列，设置m_stat，让子线程处理任务的时候知道是读事件，主线程先阻塞等待看是否读取成功（读的时候，如果客户端发送的请求体比较大，程序设置的读缓冲区为防止溢出会返回false），如果返回false，从epoll取下fd，从链表取下定时器。

客户端fd是ET+io非阻塞：先读，返回false就如上取下，否则就放进任务队列。

读完之后子线程会接着**解析请求行，头，体，并且通过主从状态机根据状态判断接下来的行为**(这里比较重要，需要仔细阅读源码逻辑），解析过程会将关键数据存储到变量之中，如果在哪里解析出错就直接返回，**重新挂起监听。**（我认为不对的点，后面有讲解）

如果解析完毕，开始判断是请求登录还是注册，如果是注册，从map中find是否已经注册，如果没有，将登录界面文件名接入一个存储文件真实路径加文件名的变量，并且将用户和密码写入数据库，如果有就接入注册error的文件名；如果是登录，判断用户和密码是否可以在map中找到，逻辑同上；如果是别的请求，给出对应的界面文件名接入。然后判断文件是否正常，对应一系列的 HTTP_CODE状态。如果这个文件是一个普通的正常文件，通过将文件进行共享内存映射（mmap）。操作完毕之后将fd以写事件挂树上（reactor反应堆模型—–>不会一股脑写，而是根据客户状态写）

5.监听到写事件：

接着通过判断 HTTP_CODE状态，如果是不正常文件，直接将状态码和请求头以及错误体通过一个函数格式化之后放进写缓冲区，且有一个m_write_idx来指示其长度，通过一个iovec结构体发送；如果是正常文件，请求头同上，通过两个iovec结构体，将写缓冲区和mmap映射区两个不连续的地址空间通过一次系统调用发送给客户端（没有把mmap映射区和写缓冲区相连，减少了拷贝操作，也减少了写缓冲区的负担，写缓冲区只存放请求头一般不会溢出）。如果客户端缓冲区比较小，一次性无法完全写入，只能重新挂起写事件，等待对方可写之后触发写事件继续写，上一次写的状态也全部保留，保证写数据的准确性，并且每一次完成完整的写操作之后都会将此http_conn类恢复到原始状态（不会全部恢复，恢复的大多是跟读写操作有关的）。



#### **单例模式是如何实现的：（重要知识点：实例化一个类需要调用它的构造函数）**

先说非单例模式的：创建类的对象或者new可以分别在栈上和堆上分配内存，调用构造函数初始化类。

单例模式：需要先将构造函数给私有话，防止每创建一个对象就实例化一次，还要将拷贝构造函数和赋值运算符给delete，防止把实例化对象的资源复制出来。（默认还有移动构造函数和其他的，不过不用管，因为移动过构造函数是进行资源的转移，不会复制）。

因为静态成员是类的一部分，所以如果在程序启动的时候就初始化静态成员变量，它可以调用构造函数（即使是私有的），进行类的实例化——->饿汉模式

可以设置一个静态成员函数，因为这个成员函数是类的一部分，在这个函数中创建一个对象，当程序调用这个函数的时候，这个对象也可以调用构造函数进行类的实例化（这个对象必须是静态的，保证只能进行一次创建）——->懒汉模式

#### Reactor反应堆模型和Proactor模型主要区别：

proactor主线程监听到事件后将io操作交给内核完成(减少了户态-内核态切换），子线程仅处理运算过程。

reactor主线程监听到事件后将io操作和运算都交给子线程完成。

#### 阻塞io的缺陷（基本使用非阻塞io）：

若使用阻塞 IO，当数据持续到达时，`epoll_wait` 会频繁通知，但 `read()` 可能因无数据而阻塞，导致 CPU 空转。



### 改进建议：

线程池中的线程数应该可以动态管理，创建出一个管理线程，当忙线程数大于存活线程数的80%或者任务队列中的任务数大于存活线程数就进行线程的增加，每次增加一定数量的线程，如果忙线程数小于存活线程数的20%就销毁一定数量的线程（这是一种算法，可自己根据实际需求进行调整）。

数据库连接池也是同理。



### 本次项目代码我认为是有些地方有问题的：

1.读缓冲区是固定大小的，如果客户发送请求体过大那就一直无法成功，但考虑到是轻量级的，只需要在读缓冲区即将溢出的时候设置错误状态并发送给客户端（改进方法：可以使用动态缓冲区，比如说vector，将要溢出的时候可以resize vector的大小）

2.http_conn类中解析请求后，如果解析不完整，应该发送错误号，并且重置此http对象，而不是重新将fd挂树上监听：因为代码重新将这个客户端fd读挂在树上，是为了继续读取未完全读入的请求体，但实际上，在读取请求体的那段代码已经设定了如果请求体太大会直接丢弃本次请求，没有让你再读一次这个说法。

3.负责存放用户名和密码的map容器应该是http_conn类静态成员变量（静态成员属于类而非类的对象，所有对象共享这一个静态变量），可以保证所有http对象公用此map—->这样才能保证每一个用户注册的账户能够在每一个hhtp对象中查阅。

