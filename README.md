﻿# LogSystem  C++ - 基于多设计模式下的同步&异步⽇志系统

日志系统:  
- 日志:程序运行过程中所记录的程序运行状态信息  
- 作用:记录了程序运行状态信息,以便于程序员能够随时根据状态信息,对系统的运行状态进行分析  
- 例子:平常调试时打印的这些信息就是日志  
- 功能:能够让用户非常简便的进行日志的输出,以及控制　

## 1.项目介绍:
本项目主要支持以下功能:
- 支持多级别日志消息  
    > 不同级别的日志应对着不同的场景,能够通过日志等级来限制输出.  
    > 当程序在调试阶段,大于调试级别的日志都可以输出.当程序发布时,将限制级别设置为错误,则低于错误级别的日志不会再输出.
- 支持同步日志和异步日志
    > 同步表示一个工作由自己来干(主线程),异步表示一个工作由别人来干(多线程)  
    > 同步是指和日志系统和主程序使用同一个线程;  
    > 异步是指日志系统独立于业务使用一个独立的线程.  
    > 异步日志的好处是:当出现诸如数据库写入失败或磁盘满等问题时,同步日志会阻塞,影响业务运行;而异步日志只会阻塞日志输出,不会影响业务运行.
- 支持可靠写入日志到控制台、文件以及滚动文件中
    > 滚动文件（可以切换不同文件输入日志）  
    > 还可以定期删除
- 支持扩展不同的日志落地目标地
    > 能够输出到数据库，服务器等. 具有高扩展性
- 支持多线程程序并发写日志
    > 该日志系统是线程安全的多线程日志系统,多个日志线程对同一个文件写入是安全的

## 2.开发环境
- CentOS 7
- vscode/vim
- g++/gdb
- Makefile

## 3.核心技术
- 类层次设计(主要以继承和多态的应用)  
    > 类的模块化设计的一种思想
- C++11(多线程、auto、智能指针、右值引用等)
- 双缓冲区  
- 生产消费模型  
- 多线程  
- 设计模式(单例、工厂、代理、建造者等)  

## 4.环境搭建
本项目不依赖其他第三方库，只需要安装好CentOS/Ubuntu + vscode/vim环境即可开发

## 5.日志系统介绍
### 5.1 为什么需要日志系统
- 生产环境的产品为了保证其稳定性和安全性是不允许开发人员附加调试器去排查问题，这种情况可以借助日志系统打印一些日志来帮助开发人员解决问题  
- 已上线的产品客户端出现bug无法复现,复现耗时太长等问题,也就是场景复现困难的情况下,如果能够在当时出问题的时候能够记录详细的信息的话,那排除的时候就方便很多.因此可以借助日志系统打印日志并上传到服务端帮助开发人员进行分析
- 一些高频操作在少量调试次数下可能无法触发我们想要的行为,如通过断点的暂停方式,我们不得不重复操作几十次,上百次甚至更多,导致排查问题效率非常低下,可以借助打印日志的方式排查问题
- 在分布式、多线程、多进程代码中，出现bug比较难以定位，可以借助日志系统打印log帮助定位bug，分析是哪台主机、哪个线程、哪个进程出现的问题,能够更加具象
- 帮助首次接触项目代码的新开发人员理解代码的运行流程。日志系统记录了项目的运行过程，能够记录程序第一步，第二步干了什么等等，能够对项目运行的流程有个基础的了解,让接下来阅读代码时会更加轻松


### 5.2 日志系统技术实现
- 利用printf、std::cout等输出函数将日志信息打印到控制台
    > 学习阶段常使用的一种方式
- 对于大型商业化项目，为了方便排查问题，我们一般会将日志输出到文件或者数据库系统方便查询和分析日志，主要分为同步日志和异步日志方式
  
#### 5.2.1 同步写日志
同步日志是指当输出日志时，必须等待日志输出语句执行完毕后，才能执行后面的业务逻辑语句，日志输出语句与程序的业务逻辑语句将在同一个线程运行。
每次调用一次打印日志API就对应一次系统调用write写日志文件。  
![img](./img/图片1.png)  
好处是设计思想简单,流程简单。缺点是在高并发场景下，随着日志数量不断增加，同步日志系统容易产生系统瓶颈。  
1.一方面，大量的日志打印需要等量的write系统调用，有一定系统开销  
2.另一方面， 打印日志的进程附带了大量同步的磁盘IO，影响程序性能  
3.如果在向服务器发送日志的网络IO过程中遇到网络拥塞,则程序会阻塞住，影响业务正常运行

#### 5.2.2 异步写日志
异步日志是指在进行日志输出时，日志输出语句与业务逻辑语句并不是在同一个线程中运行，而是有专门的线程用于进行日志输出操作。业务线程只需要将日志放到一个内存缓冲区中，不用等待即可继续执行后续业务逻辑（作为日志的生产者），而日志的落地操作交给单独的日志线程去完成（作为日志的消费者），这是一个典型的生产-消费模型。  
![img](./img/图片2.png)  
这样做的好处是即使日志没有真正的完成输出也不会影响程序的主业务，对业务运行的影响最小, 提高程序的性能:
- 主线程调用日志打印接口成为非阻塞操作
- 同步的磁盘IO从主线程中剥离出来交给单独的线程完成

## 6.相关知识补充
### 6.1 不定参数

- C风格不定参数
- C++风格不定参数

### 6.2 设计模式

设计模式是程序员对代码开发经验的总结,是解决特定问题的一系列套路.他不是语法规定,而是一套用来提高代码可复用性、可维护性、可读性、稳健性以及安全性的解决方案.

#### 六大原则:

- 单一职责原则(Single Responsibility Principle) 
  - 概念:类的职责应该单一,一个方法只做一件事.职责划分清晰,每次改动到最小单位的方法或类.
  - 说明:两个完全不一样的功能不应该放在一个类中,一个类中应该是一组相关性很高的函数、数据的封装
  - 用例:网络聊天 = 网络通信+聊天, 因此应该分割成网络通信类+聊天类
- 开闭原则(Open Closed Principle)
  - 概念:对扩展开放,对修改封闭
  - 说明:对软件实体的改动,最好用扩展而非修改的方式
  - 用例:超市促销,商品价格需要修改,选择不修改商品原来的价格,而是新增促销价格
- 里氏替换原则(Liskov Substitution Principle)
  - 概念: 只要父类能出现的地方,子类就可以出现,而且替换为子类也不会产生任何错误或异常(子类可以切片给父类,但是不能反过来)
  - 概念: 派生类(子类)对象可以在程序中代替其基类(超类)对象.
  - 注意: 在继承类时,务必重写父类中所有的方法,尤其需要注意父类的protected方法,子类尽量不要暴露自己的pulibc方法供外界调用.
  - 说明: 子类必须完全实现父类的方法,孩子类可以有自己的个性.覆盖或者实现父类的方法时,输入参数可以被放大,输出可以被缩小
  - 用例: 跑步运动员类-会跑步,子类长跑运动员-会跑步且擅长长跑,子类短跑运动员-会跑步且擅长短跑.
- 依赖倒置原则(Dependence Inversion Principle)
  - 高层模块不应该依赖底层模块,两个都应该依赖其抽象.不可分割的原子逻辑就是底层模式,原子逻辑组装成的就是高层模块.
  - 模块间依赖通过抽象(接口)发生,具体类之间不直接依赖.
  - 说明:每个类都尽量有抽象类,任何类都不应该从具体类派生.尽量不要重写基类的方法.结合里氏替换原则使用
  - 用例:奔驰车司机类--只能开奔驰;  司机类 -- 给什么车就开什么车; 
- 迪米特法则(Law of Demeter),又叫"最少知道法则"
  - 尽量减少对象之间的交互,从而减小类之间的耦合.一个对象应该对其他对象有最少的了解.对类的低耦合提出了明确的要求
    - 如果一个方法放在本类中,既不增加类间的关系,也对本类不产生负面影响,那就放置在本类中
  - 用例:老师让班长点名--老师给班长一个名单,班长完成点名勾选,返回结果.而不能是班长点名,老师勾选

- 接口隔离原则(Interface Segregation Principle)
  - 客户端不应该依赖它不需要的接口,类间的依赖关系应该建立在最小的接口上
  - 说明:接口设计尽量精简单一,但是不要对外暴露没有实际意义的接口
  - 用例:修改密码,不应该提供修改用户信息接口,只要提供单一的最小修改密码接口,更不要暴露数据库操作



从整体上来理解六大设计原则,可以简要地概括成一句话:用抽象构建框架,用实现来扩展细节,具体到每一条设计原则,则对应一条注意事项:

- 单一职责原则:实现类要职责单一;

  类实现要高内聚,低耦合

- 开闭原则:总纲,要对扩展开放,对修改关闭.

  已写好的类不要修改,新的功能通过构造新的类来完成.(功能不够再new新的子类)

- 依赖倒置原则:面向接口编程;

  策略模式,只使用父类接口(虚函数多态),由子类实现细节.

- 里氏替换原则:不要破坏继承体系;

  子类能够完全替换掉父类;子类要满足父类提供的所有接口(重写所有的虚函数)

- 接口隔离原则:在设计接口时要精简单一;

- 迪米特法则:降低耦合;

  

#### 单例模式

​    一个类只能创建一个对象,即单例模式,该设计模式可以保证系统中该类只有一个实例,并提供一个访问它的全局访问点,该实例被所有程序模块共享.比如在某个服务器程序中,该服务器的配置信息存放在一个文件中.这些配置数据由一个单例对象统一读取,然后服务进程中的其他对象再通过这个单例对象获取这些配置信息,这种方式简化了在复杂环境下的**配置管理**.

​    单例模式有两种实现模式:饿汉模式和懒汉模式

- 饿汉模式:程序启动时就会创建一个唯一的实例对象.是以空间换时间的策略.
  - 优点:逻辑简单,使用方便
  - 缺点:初始化时间久
- 因为单例对象已经确定,所以比较适用于多线程环境中,多线程获取单例对象不需要加锁,可以有效的避免资源竞争,提高性能.

单例用途:配置管理,任务队列...



##### 饿汉模式示例

```
//Hungry Mode example
#include<iostream>

class Singleton
{
  public:
    static Singleton& getInstance() { return _instance; } 
    int getData() { return _data;}

  private:
    Singleton():_data(99)//一般情况单例的数据都是从配置文件读取
    {
      //必须要实现一个构造函数用于动态申请.
      std::cout<<"单例对象初始化"<<std::endl;
    }; 
    Singleton(const Singleton& s)=delete;
    Singleton& operator=(const Singleton& s)=delete;
    ~Singleton(){};

  private:
    int _data;
    static Singleton _instance ; //不允许非static成员,会无限定义下去.static只允许初始化一次可以使用
};

Singleton Singleton::_instance; 
//类是一个类型,必须从代码区调用出来才算实例化


int main()
{
  std::cout<<Singleton::getInstance().getData()<<std::endl;
  return 0;
}

```

##### 懒汉模式示例1

双检查加锁方式

```
#include<iostream>
#include<mutex>

//C++通用版本
//注意:存在因内存顺序(代码执行顺序)导致安全隐患

class Singleton
{
  public:
    static Singleton* getInstance()
    {
      if(_instance==nullptr)
      {
        std::lock_guard<std::mutex> lg(_mtx);
        if(_instance == nullptr)
        {
          _instance = new Singleton(); 
          //new过程不安全,底层执行存在代码执行顺序问题,如果先指向,再写数据,在指向成功后,指针就不为nullptr,此后其他线程就能直接使用这块空间,如果还未初始化完毕,此时使用这块内存就会出现问题.因此存在这样的安全隐患.
        }
      }
      return _instance;
    }
    int getData()
    {
      return _data;
    }

  private:
    Singleton():_data(99){
        std::cout<<"Lazy Mode Singleton Initialized!"<<std::endl;
    };   
    Singleton(const Singleton& s) = delete;
    Singleton& operator=(const Singleton&s) = delete;
    ~Singleton(){};

  private:
    //不能在堆栈上申请
    int _data;
    static std::mutex _mtx;
    static Singleton* _instance; //满足动态内存管理要求,需要使用指针
};
Singleton* Singleton::_instance = nullptr; //static成员必须在类外初始化
std::mutex Singleton::_mtx;

int main()
{
  std::cout<<"function main is started"<<std::endl;
  std::cout<<Singleton::getInstance()->getData()<<std::endl;
  return 0;
}
```

##### 懒汉模式示例2

使用静态变量方式,C++11后支持的版本,代码更简洁

[静态局部变量](https://zh.cppreference.com/w/cpp/language/storage_duration#.E9.9D.99.E6.80.81.E5.B1.80.E9.83.A8.E5.8F.98.E9.87.8F)

![image-20240524150658606](README.assets/image-20240524150658606.png)

>  C++11只保证静态局部(函数体内)变量初始化的线程安全

```
#include<iostream>
#include<mutex>

//C++11版本懒汉单例模式

class Singleton
{
  public:
    static Singleton& getInstance()
    {
      static Singleton _instance; //C++11支持本地静态变量初始化时是线程安全的
      return _instance;
    }
    int getData()
    {
      return _data;
    }

  private:
    Singleton():_data(99){
        std::cout<<"Lazy Mode Singleton Initialized!"<<std::endl;
    };   
    Singleton(const Singleton& s) = delete;
    Singleton& operator=(const Singleton&s) = delete;
    ~Singleton(){};

  private:
    int _data;
};

int main()
{
  std::cout<<"function main is started"<<std::endl;
  std::cout<<Singleton::getInstance().getData()<<std::endl;
  return 0;
}
```

##### 懒汉模式示例3

使用call_once()函数,C++11后支持的方式,代码更简洁

```
#include<iostream>
#include<thread>
#include<mutex>

std::once_flag g_flag;

class Singleton {
public:
    Singleton(const Singleton& s) = delete;
    Singleton& operator=(const Singleton&s) = delete;
    static Singleton* GetInstance() {
        std::call_once(g_flag,[](){ std::cout<<"do once:"<<std::this_thread::get_id()<<"\n"; _instance = new Singleton; }); //成员函数中lambda默认隐式捕获this,因此可以直接访问到成员变量
        std::cout<<std::this_thread::get_id()<<"\n";
        return _instance;
    }

private:
    Singleton(){};
    static Singleton* _instance;
};
Singleton* Singleton::_instance = nullptr;


int main() {
   std::thread t1(Singleton::GetInstance);
   std::thread t2(Singleton::GetInstance);
   std::thread t3(Singleton::GetInstance);
    
    t1.join();
    t2.join();
    t3.join();
	return 0;
}
```

![image-20240628223634392](README.assets/image-20240628223634392.png)

##### 懒汉模式示例4

C++11原子变量+双检查加锁

```
#include <iostream>
#include <thread>
#include<chrono>
#include<string>
#include<queue>
#include<mutex>

class BlockQueue {
public:
    BlockQueue(const BlockQueue& q) = delete;
    BlockQueue& operator=(const BlockQueue& q) = delete;

    void Push() {

    }

    void Pop() {

    }

    static BlockQueue& GetInstance() {
        if (_instance.load() == nullptr) {
            std::lock_guard<std::mutex>lg(_mtx);
            if (_instance.load() == nullptr) {
                _instance.store(new BlockQueue); //内存顺序不会打乱,安全
            }
        }
        return *(_instance.load());
    }

private:
    BlockQueue() {}
    std::queue<int> _q;
    int _capacity = 5;
    static std::mutex _mtx;
    static std::atomic<BlockQueue*> _instance;
};
std::atomic<BlockQueue*> BlockQueue::_instance = nullptr;
std::mutex BlockQueue::_mtx;


int main() {
    BlockQueue::GetInstance();
    return 0;
}
```







#### 工厂模式

工厂模式是一种创建型设计模式,它提供了一种创建对象的最佳方式.在工厂模式中,我们创建对象时不会对上层暴露创建逻辑,而是通过使用一个共同结构来指向新创建的对象,以此实现创建-使用的分离

工厂模式可以分为:

- 简单工厂模式:简单工厂模式实现由一个工厂对象通过类型决定创建出来指定产品类的实例.假设有个工厂能生产出水果,当客户需要产品的时候明确告知工厂类生产哪类水果,工厂需要接收用户提供的类别信息,当新增产品的时候,工厂内部去添加新产品的生产方式.
