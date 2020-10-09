### Bank_Service_Management

一、实验介绍

1.1 实验内容

端午节当天，某个银行从早上八点开始服务并只服务到中午十二点就停止营业。假设当天银行只提供了 w 个服务窗口进行服务，问：
平均每分钟有多少个顾客抵达银行?
平均每个顾客占用服务窗口的时间是多少？
1.2 实验知识点

OOP 编程思想
std::rand() 函数原理
概率编程
排队理论
链式队列数据结构及其模板实现
事件驱动的设计
蒙特卡洛方法
CPU 资源争夺模型
时间片轮转调度
二、实验原理

2.1 要解决的问题

蒙特卡洛方法这个名字听起来很高大上，但它的本质其实是使用计算机的方法对问题进行模拟和复现。本次实验将使用蒙特卡洛方法来模拟银行排队这个问题：

我们先来分析一下这个业务的逻辑：

首先我们要分析银行提供服务的逻辑。在银行服务中，所有顾客都是通过取号排队的方式等待服务的，这和火车站买票有所不同，在火车站买票时，顾客必须在某一个窗口所排的队列下进行排队，且无法变更自己所属的窗口，否则只能从队尾重新排队。换句话说，对于银行提供的服务来说，所有用户都是位于同一个队列上的，当某个服务窗口可用时，才会从排队队列的队首取出一个新的用户来办理银行业务。即代码实现过程中，服务窗口可以创建 w 个，但只需要实现一个顾客队列即可。

其次，对于顾客而言，有两个属性是能够被抽象出来的：

到达银行的时间；
需要服务的时间。
并且，这两个属性是随机的。到此，我们整个的排队模型就变成了：


下面我们来详细对这个问题的实现逻辑进行分析，让我们的程序能够给出类似下面的结果：

2.2 计算机中的随机

std::rand() 函数的原理

C++ 中的 std::rand() 函数产生的随机数并不是真正意义上的随机数，它并不服从数学上的均匀分布。为了使我们的模拟系统变得更加真实，我们需要知道 std::rand() 函数的原理。

std::rand() 生成的是一个随机的二进制序列（在硬件底层更好实现），这个序列的每一位0或者1的概率都是相等的。而对于 std::rand()%n 这个运算，会在 [0, n-1] 之间生成随机数，所以，如果 n-1 的二进制表示的值不都是由 1 组成，那么这里面的数是不会从均匀分布了（因为某些位可能不能为 1）。

所以，当且仅当 [0, n-1] 中的随机数可以用这个序列的子序列表示时，才能满足均匀分布。换句话说，仅当 n-1 的二进制数全为1 时，0，1出现的概率才是均等的。

我们先来实现随机这个类，在 /home/shiyanlou/ 目录下创建 queuesystem 文件夹，在这个文件夹下新建 Random.hpp 文件：
```
//
//  Random.hpp
//  QueueSystem
//

#ifndef Random_hpp
#define Random_hpp

#include <cstdlib>
#include <cmath>

class Random {
public:
    // [0, 1) 之间的服从均匀分布的随机值
    static double uniform(double max = 1) {
        return ((double)std::rand() / (RAND_MAX))*max;
    }
};
#endif /* Random_hpp */
```

这样的话，当我们调用 Random::uniform() 时，便能获得真正的服从均匀分布的随机数了。当指定参数后，便能够生成 [0, max) 之间的随机值了。

三、实验步骤

下面，我们进入系统的设计和建模的具体实施步骤。

3.1 主函数逻辑设计

对于一个银行而言，对外界来说只需要提供两个参数：

总共的服务时间
服务窗口的数量
在 /home/shiyanlou/queuesystem/ 目录下新建 main.cpp 文件，我们希望实现这样的代码，：
```
// main.cpp
// QueueSystem
#include "QueueSystem.hpp"

#include <iostream>
#include <cstdlib>

int main() {
    std::srand((unsigned)std::time(0)); // 使用当前时间作为随机数种子
    int total_service_time = 240;       // 按分钟计算
    int window_num         = 4;
    int simulate_num       = 100000;    // 模拟次数

    QueueSystem system(total_service_time, window_num);
    system.simulate(simulate_num);

    std::cout << "The average time of customer stay in bank: "
              << system.getAvgStayTime() << std::endl;
    std::cout << "The number of customer arrive bank per minute: "
              << system.getAvgCustomers() << std::endl;
    return 0;
}
```

3.2 对象及逻辑设计
总结一下，现在我们需要实现的东西有：
服务窗口类(会被创建 w 个)
顾客队列类(只会被创建一个)
顾客结构(包含两个随机属性: 到达时间, 服务时间)
为了更好练习 C++，我们会弃用诸如 vector 这些快捷编码的标准库来进行『过度编码』，自行编写模板类。
根据前面的问题描述，我们可以初步确定这样一些类的设计需求：

QueueSystem 类: 负责整个队列系统的模拟；
ServiceWindow 类: 队列系统的服务窗口对象，每当一个银行创建时，服务窗口会被创建，为了让整个问题更加灵活，我们假设需要创建 window_num 个窗口；
Queue 类: 银行队列系统的顾客排队的队列；
Random 类: 在第二节中已经讨论过。
然而，在设计 ServiceWindow 之前，我们要考虑 ServiceWindow 类到底要放置什么成员，首先，对于一个服务窗口，会有一个顾客属性，用于存放顾客。另一方面，一个窗口只会有两种状态：要么正在服务（被占用），要么空闲。因此 ServiceWindow 中首先会有下面的枚举，在 /home/shiyanlou/queuesystem/ 目录下新建 ServiceWindow.hpp 文件：
```
// ServiceWindow.hpp
// QueueSystem

enum WindowStatus {
    SERVICE,
    IDLE,
};
```

既然我们要在 ServiceWindow 中存放顾客，由于顾客本身并不需要提供什么方法，因此可以直接将顾客设计为一个结构体 Customer，同时，顾客也会成为等待队列中的一员。所以，Customer 也可以被称之为队列的一个 Node，此外，每个顾客说需要的服务时间是随机的，但是到达时间并不应该由顾客自身确定（我们在下一节再讨论为什么），所以Customer结构的默认构造应该被设计出来，在 /home/shiyanlou/queuesystem/ 目录下新建 Node.hpp 文件：
```
//  Node.hpp
//  QueueSystem
#ifndef Node_hpp
#define Node_hpp

#include "Random.hpp"

#define RANDOM_PARAMETER 100

struct Node {
    int arrive_time;
    int duration;
    struct Node *next;

    // 默认到达事件为0，需要服务的事件是随机的
    Node(int arrive_time = 0,
         int duration = Random::uniform(RANDOM_PARAMETER)):
        arrive_time(arrive_time),
        duration(duration),
        next(NULL) {}
};

typedef struct Node Node;
typedef struct Node Customer;

#endif /* Node_h */
```

那么，结合前面的 WindowStatus枚举和 Customer结构，我们的 ServiceWindow 类可以这样设计，因为窗口本身涉及的操作还算是比较简单，比如设置窗口状态是否繁忙，获取当前服务顾客的到达时间来方便后续计算等等，因此我们直接将其设计成类内的 inline 函数：
```
//  ServiceWindow.hpp
//  QueueSystem

#ifndef ServiceWindow_hpp
#define ServiceWindow_hpp

#include "Node.hpp"

enum WindowStatus {
    SERVICE,
    IDLE,
};

class ServiceWindow {
public:
    inline ServiceWindow() {
        window_status = IDLE;
    };
    inline bool isIdle() const {
        if (window_status == IDLE) {
            return true;
        } else {
            return false;
        }
    }
    inline void serveCustomer(Customer &customer) {
        this->customer = customer;
    }
    inline void setBusy() {
        window_status = SERVICE;
    }
    inline void setIdle() {
        window_status = IDLE;
    }
    inline int getCustomerArriveTime() const {
        return customer.arrive_time;
    }
    inline int getCustomerDuration() const {
        return customer.duration;
    }
private:
    Customer customer;
    WindowStatus window_status;
};

#endif /* ServiceWindow_hpp */
```

3.3 事件驱动的设计

有了上面的这些设计，似乎我们只要编写好用户排队队列，就已经足够描述整个排队的系统了，然而，在上面的设计中，还有一个很大的问题，那就是：整个系统还处于静止状态。当顾客位于等待队列时，窗口什么时候服务下一个顾客，如何处理这里面的逻辑，到目前为止，我们都没有思考过。
为了让整个系统『运行』起来，我们还要考虑整个系统的运行时间线。这里我们给出一种事件驱动的设计。
在前面的分析中，我们知道整个系统中，无非出现两种事件：


有顾客到达
有顾客离开
其中，第二种顾客离开的事件，同时还包含了窗口服务等待队列中的下一个顾客这个事件。所以，我们如果能够维护一个事件列表，那么就能够驱动整个队列系统的运行了。因为，当事件发生时，我们通知这个队列系统更新他自身的状态即可。

综上所述，我们可以先设计事件表中的事件结构，在 /home/shiyanlou/queuesystem/ 目录下新建 Event.hpp 文件：
```
//  Event.hpp
//  QueueSystem

#ifndef Event_hpp
#define Event_hpp

#include "Random.hpp"
#define RANDOM_PARAMETER 100

struct Event {
    int occur_time;

    // 使用 -1 表示到达事件, >=0 表示离开事件, 同时数值表示所离开的服务窗口
    int event_type;

    Event* next;

    // 默认为到达事件，发生事件随机
    Event(int occur_time = Random::uniform(RANDOM_PARAMETER),
          int event_type = -1):
        occur_time(occur_time),
        event_type(event_type),
        next(NULL) {}
};

#endif /* Event_hpp */
```
这里我们使用了一个小小的 trick，那就是用整数来表示事件的类型，而不是简单的使用枚举。

这是因为，对于 ServiceWindow 来说，我们可以使用数组来管理多个 ServiceWindow，那么对应的事件类型如果涉及为整数，事件类型就可以同时作为 ServiceWindow 的索引下标了，当 event_type 大于等于 0 时，数值还表示离开的服务窗口。

又因为事件列表、顾客队列，本质上可以归类为同一个结构，那就是队列：只不过他们的入队方式有所差异，对于事件列表而言，入队方式必须按发生事件的时间顺序入队，而对于顾客，则是直接添加到队尾。考虑到了这一点，我们便能很容易的利用模板来设计队列的基本需求了，在 /home/shiyanlou/queuesystem/ 目录下新建 Queue.hpp 文件：
```
//
//  Queue.hpp
//  QueueSystem
//

#ifndef Queue_hpp
#define Queue_hpp

#include <iostream>
#include <cstdlib>

#include "Event.hpp"

// 带头结点的队列
template <typename T>
class Queue
{
public:
    Queue();
    ~Queue();
    void clearQueue();             // 清空队列
    T* enqueue(T &node);
    T* dequeue();
    T* orderEnqueue(Event &event); // 只适用于事件入队
    int  length();
private:
    T *front;  // 头结点
    T *rear;   // 队尾
};
#endif /* Queue_hpp */
```
3.4 QueueSystem

经过前面的讨论，我们已经完成了对所有基本结构的设计，根据这些设计，我们能够初步确定我们要实现的队列系统的基本结构。

首先，根据对主函数的设计，初始化整个队列系统我们需要两个参数：

银行的总服务时间(分钟) int total_service_time
银行开放的服务窗口数 int window_num
其次，我们需要 QueueSystem 发开放至少三个接口：

模拟 simulate()
获得顾客平均逗留时间 getAvgStayTime()
获得平均每分钟顾客数 getAvgCustomers()
第三，内部需要实现的内容包括：

系统运行前的初始化 init()
让系统运行的 run()
系统结束一次运行的清理工作 end()
第四，整个系统需要管理的核心成员有：

可供服务的窗口 ServiceWindow* windows
顾客等待队列 Queue<Customer> customer_list
事件列表 Queue<Event> event_list
当前的系统事件 Event* current_event
第五，处理事件的方法：

处理顾客到达事件 void customerArrived()
处理顾客离开事件 void customerDeparture()
最后，我们所希望的平均顾客逗留时间和平均每分钟的顾客数涉及的四个变量：

顾客的总逗留时间 int total_customer_stay_time
一次运行中系统服务的中顾客数量 int total_customer_num
每分钟平均顾客数 double avg_customers
顾客平均逗留时间 double avg_stay_time
事实上，可以预见的是，在处理顾客服务逻辑的时候，我们还需要一个方法 getIdleServiceWindow 来获取当前服务窗口的状态，从而增加代码的复用度。
所以，在 /home/shiyanlou/queuesystem/ 目录下新建 QueueSystem.hpp 文件，整个 QueueSystem 类的代码设计为：
```
#ifndef QueueSystem_hpp
#define QueueSystem_hpp

#include "Event.hpp"
#include "Queue.hpp"
#include "ServiceWindow.hpp"

class QueueSystem {

public:
    // 初始化队列系统
    QueueSystem(int total_service_time, int window_num);

    // 销毁
    ~QueueSystem();

    // 启动模拟
    void simulate(int simulate_num);

    inline double getAvgStayTime() const {
        return avg_stay_time;
    }
    inline double getAvgCustomers() const {
        return avg_customers;
    }

private:
    // 让队列系统运行一次
    double run();

    // 初始化各种参数
    void init();

    // 清空各种参数
    void end();

    // 获得空闲窗口索引
    int getIdleServiceWindow();

    // 处理顾客到达事件
    void customerArrived();

    // 处理顾客离开事件
    void customerDeparture();

    // 服务窗口的总数
    int window_num;

    // 总的营业时间
    int total_service_time;

    // 顾客的逗留总时间
    int total_customer_stay_time;

    // 总顾客数
    int total_customer_num;

    // 核心成员
    ServiceWindow*  windows;
    Queue<Customer> customer_list;
    Queue<Event>       event_list;
    Event*          current_event;

    // 给外部调用的结果
    double avg_customers;
    double avg_stay_time;

};
#endif /* QueueSystem_hpp */
```

在这一节中，我们设计了整个银行排队系统的基本逻辑，并借鉴了事件驱动的思想设计了驱动队列系统的事件类。本节中我们一共创建了：

Event.hpp
Node.hpp
Queue.hpp
Random.hpp
ServiceWindow.hpp
QueueSystem.hpp
main.cpp
现在我们的代码还不能够直接运行，本节我们先关注理清我们的业务逻辑。在下一节中，我们将实现这些代码的详细逻辑，这包括：

Queue.hpp 中模板链式队列的具体实现
QueueSystem.cpp 中的详细服务逻辑
Random.hpp 中更复杂的随机概率分布
在这些实现中，我们将进一步巩固下面的知识的运用：

C++ 类模板
链式队列的数据结构
概率编程

一、概述

实验所需的前置知识

C++ 基本语法知识
实验所巩固并运用的知识

OOP 编程思想
std::rand() 函数原理
概率编程
排队理论
链式队列数据结构及其模板实现
事件驱动的设计
蒙特卡洛方法
CPU 资源争夺模型
时间片轮转调度
要解决的问题

端午节当天，某个银行从早上八点开始服务并只服务到中午十二点就停止营业。假设当天银行只提供了 w 个服务窗口进行服务，问：

平均每分钟有多少个顾客抵达银行?
平均每个顾客占用服务窗口的时间是多少？
上节实验回顾

在上一节实验中，我们已经理解清楚了整个队列系统的业务设计，并借鉴了事件驱动的思想设计了驱动队列系统的事件类，一共创建了：

Event.hpp
Node.hpp
Queue.hpp
Random.hpp
ServiceWindow.hpp
QueueSystem.hpp
main.cpp
这样的几个文件。这次实验中，我们来专注实现这些代码的详细逻辑，这包括：

Queue.hpp 中模板链式队列的具体实现
QueueSystem.cpp 中的详细服务逻辑
Random.hpp 中更复杂的随机概率分布
二、实验步骤

接下来，我们进行具体的代码编写工作。

2.1 模板链队的实现

上节实验中我们给出了模板链队的基本设计需求，下面我们来实现这个队列。

2.1.1 链式队列的设计理由

首先解释为什么我们要使用链式队列，而不是用数组队列：我们知道，链表的优点是插入元素的算法复杂度是 O(1)O(1) 的。虽然对于顾客的等待队列来说，我们不涉及在整个队列中插入元素，入队和出队都是发生在队列的两头。但是，我们在讨论事件的时候就遇到了这样的一个问题：事件在队列中的插入，是按事件发生时间的顺序进行插入的。这时，就不一定发生在队列的两端了。我们的队列要同时作为这两种结构的基础，显然，使用链表结构是最佳的选择。

2.1.2 实现

确定了使用链表结构，还需要面临一个链表是否需要设计头结点的问题。我们知道，有头结点的链表能够更加方便的进行代码实现，但是会损耗一个对象的空间来放置头结点。在早期的计算机软件开发中，内存是非常珍贵的，通常在设计队列的时候会设计成不带头结点的链表。但是在现在计算机中，这点内存就不值一提了，所以我们使用带头结点的链表结构。

那么，队列的构造和析构函数就需要负责管理整个链表的声明周期，维护好头结点和尾节点的变化，修改 /home/shiyanlou/queuesystem/ 目录下的 Queue.hpp 文件：
```
//
// Queue.hpp
// QueueSystem
//

// 构造一个带头结点的链式队列，节点位于队列尾部
Queue() {
    this->front = new T;
    // 如果内存申请失败，则不应该继续运行程序了
    if (!this->front) {
        exit(-1);
    }

    // 初始化节点
    this->front->next = NULL;
    this->rear = this->front;
}
// 销毁一个队列时候需要释放节点中申请的内存
~Queue() {
    // 清空当前队列中的元素
    this->clearQueue();

    // 再删除头结点
    delete this->front;
}
```
其次，对于一般的入队和出队逻辑：
```
//
// Queue.hpp
// QueueSystem
//

// 入队时，传递节点指针，外部数据不应该由此类进行管理，所以将数据拷贝一份
// 并返回头指针
T* enqueue(T &node) {
    T *new_node = new T;
    if (!new_node) {
        exit(-1);
    }
    *new_node = node;
    this->rear->next = new_node;
    this->rear = new_node;
    return this->front;
}
// 出队时，从队头元素出队
T* dequeue() {
    // 如果队列空，则返回空指针
    if (!this->front->next) {
        return NULL;
    }

    T *temp_node;
    temp_node = this->front->next;
    this->front->next = temp_node->next;

    // 如果队列中只有一个节点，那么记得将队尾节点指针置为头结点
    if (this->rear == temp_node) {
        this->rear = this->front;
    }
    return temp_node;
}
```
对于事件的顺序入队逻辑：
```
//
// Queue.hpp
// QueueSystem
//

// 事件时的顺序插入，事件有自身的发生事件，应该按事件顺序进行插入
T* orderEnqueue(Event &event) {
    Event* temp = new Event;
    if (!temp) {
        exit(-1);
    }
    *temp = event;

    // 如果这个列表里没有事件, 则把 temp 事件插入
    if (!this->front->next) {
        this->enqueue(*temp);
        return this->front;
    }

    // 按时间顺序插入
    Event *temp_event_list = this->front;

    // 如果有下一个事件，且下一个事件的发生时间小于要插入的时间的时间，则继续将指针后移
    while (temp_event_list->next && temp_event_list->next->occur_time < event.occur_time) {
        temp_event_list = temp_event_list->next;
    }

    // 将事件插入到队列中
    temp->next = temp_event_list->next;
    temp_event_list->next = temp;

    // 返回队列头指针
    return this->front;
}
```
队列的长度计算：
```
//
// Queue.hpp
// QueueSystem
//

int  length() {
    T *temp_node;
    temp_node = this->front->next;
    int length = 0;
    while (temp_node) {
        temp_node = temp_node->next;
        ++length;
    }
    return length;
}
```
最后，清空当前队列中的元素：
```
//
// Queue.hpp
// QueueSystem
//

void clearQueue() {
    T *temp_node;

    while (this->front->next) {
        temp_node = this->front->next;
        this->front->next = temp_node->next;
        delete temp_node;
    }

    this->front->next = NULL;
    this->rear = this->front;
}
```
注意，清空队列的逻辑应该注意，我们在依次删除完队列中元素的内存时，应该将头和尾节点进行复位，否则会出现很严重的内存泄露问题，因为我们的入队是通过尾指针实现的。
在 C++ 中，一个非常头疼的事情就是对动态申请的内存进行有效的管理，一不小心就会出现内存泄露的问题。

我们来看入队方法和出队方法中两个很关键的设计：

入队时尽管引用了外部的数据，但是并没有直接使用这个数据，反而是在内部新分配了一块内存，再将外部数据复制了一份。
出队时，直接将分配的节点的指针返回了出去，而不是拷贝一份再返回。
在内存管理中，本项目的代码使用这样一个理念：谁申请，谁释放。

队列这个对象，应该管理的是自身内部使用的内存，释放在这个队列生命周期结束后，依然没有释放的内存。

2.2 队列系统的实现

首先，整个银行的队列系统在被构造出来时，可供服务的窗口就应该被创建好，于是我们有了对构造函数和析构函数的实现，修改 /home/shiyanlou/queuesystem/ 目录下的 QueueSystem.cpp 文件：
```
//
// QueueSystem.cpp
// QueueSystem
//

QueueSystem::QueueSystem(int total_service_time, int window_num):
    total_service_time(total_service_time),
    window_num(window_num),
    total_customer_stay_time(0),
    total_customer_num(0) {
    // 创建服务窗口
    this->windows = new ServiceWindow[window_num];
}
QueueSystem::~QueueSystem() {
    delete[] windows;
}
void QueueSystem::simulate(int simulate_num) {
    double sum = 0;
    for (int i = 0; i != simulate_num; ++i) {
        sum += run();
    }
    avg_stay_time = (double)sum / simulate_num;
    avg_customers = (double)total_customer_num / (total_service_time*simulate_num);
}
```
接下来，simulate() 方法：
```
//
// QueueSystem.cpp
// QueueSystem
//

void QueueSystem::simulate(int simulate_num) {
    double sum = 0;
    for (int i = 0; i != simulate_num; ++i) {
        // 每一遍运行，我们都要增加在这一次模拟中，顾客逗留了多久
        sum += run();
    }

    // 计算平均逗留时间
    avg_stay_time = (double)sum / simulate_num;
    // 计算每分钟平均顾客数
    avg_customers = (double)total_customer_num / (total_service_time*simulate_num);
}
```
然后，对于 run(), init(), end() 这三个方法，部署了整个系统的动态运行，同时运行又是由 event_list 来驱动的：
```
//
// QueueSystem.cpp
// QueueSystem
//

// 系统开启运行, 初始化事件链表, 部署第一个事件
void QueueSystem::init() {
    // 第一个事件肯定是到达事件, 使用默认构造
    Event *event = new Event;
    current_event = event;
}
// 系统开始运行，不断消耗事件表，当消耗完成时结束运行
double QueueSystem::run() {
    this->init();
    while (current_event) {
        // 判断当前事件类型
        if (current_event->event_type == -1) {
            customerArrived();
        } else {
            customerDeparture();
        }
        delete current_event;
        // 获得新的事件
        current_event = event_list.dequeue();
    };
    this->end();
    // 返回顾客的平均逗留时间
    return (double)total_customer_stay_time/total_customer_num;
}
// 系统运行结束，将所有服务窗口置空闲，并清空用户的等待队列和事件列表
void QueueSystem::end() {
    // 设置所有窗口空闲
    for (int i=0; i != window_num; ++i) {
        windows[i].setIdle();
    }
    // 顾客队列清空
    customer_list.clearQueue();

    // 事件列表清空
    event_list.clearQueue();

}
```
最后我们来到了最为核心的逻辑：处理顾客到达事件和顾客离开事件。

首先来看顾客到达事件的逻辑：

顾客到达时，总的顾客数应该增加。
而当前事件发生时，下一个顾客同时应该被创建，如果下一个到达的顾客在银行关门之前到达，那么久应该被添加到事件列表中。
当前事件到达的顾客应该被插入到等待队列中。
如果服务窗口有空闲，那么应该立即服务队列中的第一个顾客。
顾客开始服务后，其离开时间应该被确定，这时候要向事件列表中顺序插入一个顾客离开的事件。
如果你还记的上一节实验中我们提到过 Customer 的到达时间不应该由自己决定，那么我们现在来解释这样设计的原因。

目前我们设计的系统是由事件来驱动的，而所有到达事件的创建，是在当前有顾客到达后，才会得到下一个顾客的到达时间。

这就是我们为什么没有将顾客的到达时间设计为让用户自己决定。

当然，你也可以让整个银行系统在创建之初就确定好当天的所有事件，这样的话顾客的到达时间就有其自身决定了。
```
//
// QueueSystem.cpp
// QueueSystem
//

// 处理用户到达事件
void QueueSystem::customerArrived() {

    total_customer_num++;

    // 生成下一个顾客的到达事件

    int intertime = Random::uniform(100);  // 下一个顾客到达的时间间隔，我们假设100分钟内一定会出现一个顾客
    // 下一个顾客的到达时间 = 当前事件的发生时间 + 下一个顾客到达的时间间隔
    int time = current_event->occur_time + intertime;
    Event temp_event(time);
    // 如果下一个顾客的到达时间小于服务的总时间，就把这个事件插入到事件列表中
    // 同时将这个顾客加入到 customer_list 进行排队
    if (time < total_service_time) {
        event_list.orderEnqueue(temp_event);
    } // 否则不列入事件表，且不加入 cusomer_list


    // 处理当前事件中到达的顾客
    Customer *customer = new Customer(current_event->occur_time);
    if (!customer) {
        exit(-1);
    }
    customer_list.enqueue(*customer);

    // 如果当前窗口有空闲窗口，那么直接将这个顾客送入服务窗口
    int idleIndex = getIdleServiceWindow();
    if (idleIndex >= 0) {
        customer = customer_list.dequeue();
        windows[idleIndex].serveCustomer(*customer);
        windows[idleIndex].setBusy();

        // 顾客到窗口开始服务时，就需要插入这个顾客的一个离开事件到 event_list 中
        // 离开事件的发生时间 = 当前时间事件的发生时间 + 服务时间
        Event temp_event(current_event->occur_time + customer->duration, idleIndex);
        event_list.orderEnqueue(temp_event);
    }
    delete customer;
}
```
其中，对于 getIdleServiceWindow() 方法的实现为：
```
//
// QueueSystem.cpp
// QueueSystem
//

int QueueSystem::getIdleServiceWindow() {
    for (int i=0; i!=window_num; ++i) {
        if (windows[i].isIdle()) {
            return i;
        }
    }
    return -1;
}
```
其次，处理顾客的离开事件，有以下几步逻辑：

如果用户离开时，银行已经关门了，那么我们就不需要做任何处理了，因为所有队列都要被销毁了，且根据顾客的到达事件的处理设计，超过总服务事件的顾客到达后不会被加入到事件列表，所以赶紧处理完剩下的离开事件就好。
顾客离开后，顾客在银行中的逗留时间应该为在队列中的等待时间，加上自身的服务时间。
如果当前顾客离开后，等待队列中还有人等待，就应该马上服务等待的顾客，否则，当前窗口就应该被设置为空闲。
因此：
```
//
// QueueSystem.cpp
// QueueSystem
//

// 处理用户离开事件
void QueueSystem::customerDeparture() {
    // 如果离开事件的发生时间比中服务时间大，我们就不需要做任何处理
    if (current_event->occur_time < total_service_time) {
        // 顾客逗留时间 = 当前顾客离开时间 - 顾客的到达时间
        total_customer_stay_time += current_event->occur_time - windows[current_event->event_type].getCustomerArriveTime();
        // 如果队列中有人等待，则立即服务等待的顾客
        if (customer_list.length()) {
            Customer *customer;
            customer = customer_list.dequeue();
            windows[current_event->event_type].serveCustomer(*customer);

            // 离开事件
            Event temp_event(
                current_event->occur_time + customer->duration,
                current_event->event_type
            );
            event_list.orderEnqueue(temp_event);

            delete customer;
        } else {
            // 如果队列没有人，且当前窗口的顾客离开了，则这个窗口是空闲的
            windows[current_event->event_type].setIdle();
        }

    }
}
```
至此，我们修改了 Queue.hpp ，增加了 QueueSystem.cpp。我们来完成最后的运行工作：

$ g++ -std=c++11 main.cpp QueueSystem.cpp -o main 
$ ./main


可以看到，当每100分钟内至少出现一个顾客、且这个顾客的服务时间为 0 至 100 分钟的之间的均匀分布的随机值时，我们模拟了 100000 次，得到的结果为：

平均每位顾客在银行中逗留了 34.85 分钟
平均每分钟会有 0.01 个顾客到达，即每 100 分钟才会有一个顾客到达（这也侧面印证了我们的程序结果运行正确）
三、拓展

更符合现实规律的复杂概率分布(可选)

这样的结果还不够正确。

数学家们对排队问题进行了更加科学和严密的研究（其实是当年贝尔实验室解决电话接入等待问题进行了一系列的研究），提出了一套对于排队的理论，学名叫做排队论。他们证明了这样的两个结论：

1. 单位时间内平均到达的顾客数如果为 n ，那么每两个顾客的到达时间的时间间隔这个随机变量是服从参数为 1/n 的泊松分布；

2. 每个顾客平均需要的服务时间如果为 t，那么 t 应该服从 参数为 1/t 的指数分布。

看到这个结论，应该能够说服你我们为什么将下一个顾客的到达时间设计成当前顾客到达后再随机生成了，因为这个时间间隔是服从泊松分布的，能够让我们方便的验证实验结果。
对此，根据指数分布和泊松分布的公式：

泊松分布：

P(X=k)=\frac{e^{-\lambda}\lambda ^k}{k!}P(X=k)= 
k!
e 
−λ
 λ 
k
 
​	 

指数分布：

当 x < 0 时, P(x) = 0P(x)=0 否则，P(x) = \lambda e^{-\lambda x}P(x)=λe 
−λx
 

我们先实现 Random.hpp 中的指数分布和泊松分布,修改 /home/shiyanlou/queuesystem/ 目录下的 Random.hpp 文件：
```
//
//  Random.hpp
//  QueueSystem
//

#ifndef Random_hpp
#define Random_hpp

#include <cstdlib>
#include <cmath>

enum RandomType {
    UNIFORM,
    EXPONENTAIL,
    POISSON,
};

class Random {
public:

    // 给定分布类型和参数，获得随机值
    static double getRandom(RandomType type, double parameter) {
        switch (type) {
            case UNIFORM:
                return uniform(parameter);
                break;
            case EXPONENTAIL:
                return exponentail(parameter);
            case POISSON:
                return poisson(parameter);
            default:
                return 0;
                break;
        }
    }
    // [0, 1) 之间的服从均匀分布的随机值
    static double uniform(double max = 1) {
        return ((double)std::rand() / (RAND_MAX))*max;
    }
    // 服从 lambda-指数分布的随机值
    static double exponentail(double lambda) {
        return -log(1 - uniform()) / lambda;
    }
    // 服从 lambda-泊松分布的随机值
    static double poisson(double lambda) {
        int t = 0;
        double p = exp(-lambda);
        double f = p;
        double u = uniform();
        while (true) {
            if (f > u)
                break;
            t++;
            p = p*lambda / t;
            f += p;
        }
        return t;
    }
};

#endif /* Random_hpp */
```
然后我们再来修改队列系统中的随机数产生逻辑，我们假设平均每分钟到达 2 个顾客，那么在 QueueSystem.cpp 和 Event.hpp 中，应该修改到达事件的时间间隔和默认到达事件(第一个到达事件) 为泊松分布：
```
// QueueSystem.cpp
int intertime = Random::uniform(100);  // 下一个顾客到达的时间间隔，我们假设100分钟内一定会出现一个顾客
// Event.hpp
Event(int occur_time = Random::uniform(RANDOM_PARAMETER),
          int event_type = -1):
```
更改为：
```
// QueueSystem.cpp
int intertime = Random::getgetRandom(POISSON, 0.5);  // 下一个顾客到达的时间间隔，服从参数为 2 的泊松分布
// Event.hpp
Event(int occur_time = Random::getRandom(POISSON, 0.5), int event_type = -1):
```
而每个顾客平均需要的服务时间如果为 10 分钟，则 Node.hpp 的服务时间 duration 应该为更改为指数分布：
```
// Node.hpp
Node(int arrive_time = 0,
         int duration = Random::uniform(RANDOM_PARAMETER)):
```
更改为：
```
// Node.hpp
Node(int arrive_time = 0,
         int duration = Random::getRandom(EXPONENTAIL, 0.1)):
```
这时候，我们再重新运行：



结果为：

平均每位顾客在银行中逗留了 19.80 分钟
平均每分钟会有 2 个顾客到达（又一次侧面印证了我们的程序结果运行非常准确，同时也符合理论的结果）

四、回顾

Queue.hpp 中模板链式队列的具体实现
QueueSystem.cpp 中的详细服务逻辑
Random.hpp 中更复杂的随机概率分布
