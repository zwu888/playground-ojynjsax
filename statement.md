# std::lock
定义在<mutex>中。
template<class Lockable1, class Lockable2, class... LockableN>
void lock(Lockable1& lock1, Locakable2& lock2, LockableN&... lockn);


使用死锁避免算法来Lock给定的Lockable的对象lock1，lock2，...，lockn，以防止死锁。

以一种不确定的调用顺序来调用lock, try_lock, unlock，以锁定lock1，lock2，...，lockn. 如果对lock或unlock的调用导致了异常，那么对于任何已经被lock住的对象，在异常被再次抛出前，unlock将被调用。

示例程序：
下面的例子使用std::lock来锁定几对mutex而没有死锁。
```C++ runnable
#include <mutex>
#include <thread>
#include <iostream>
#include <vector>
#include <functional>
#include <chrono>
 
struct Employee {
    Employee(std::string id) : id(id) {}
    std::string id;
    std::vector<std::string> lunch_partners;
    std::mutex m;
    std::string output() const
    {
        std::string ret = "Employee " + id + " has lunch partners: ";
        for( const auto& partner : lunch_partners )
            ret += partner + " ";
        return ret;
    }
};
 
void send_mail(Employee &, Employee &)
{
    // simulate a time-consuming messaging operation
    std::this_thread::sleep_for(std::chrono::seconds(1));
}
 
void assign_lunch_partner(Employee &e1, Employee &e2)
{
    {
        static std::mutex io_mutex;
        std::lock_guard<std::mutex> lk(io_mutex);
        std::cout << e1.id << " and " << e2.id << " are waiting for locks" << std::endl;
    }
 
    // use std::lock to acquire two locks without worrying about 
    // other calls to assign_lunch_partner deadlocking us
    {
        std::lock(e1.m, e2.m);
        std::lock_guard<std::mutex> lk1(e1.m, std::adopt_lock);
        std::lock_guard<std::mutex> lk2(e2.m, std::adopt_lock);
// Equivalent code (if unique_lock's is needed, e.g. for condition variables)
//        std::unique_lock<std::mutex> lk1(e1.m, std::defer_lock);
//        std::unique_lock<std::mutex> lk2(e2.m, std::defer_lock);
//        std::lock(lk1, lk2);
        std::cout << e1.id << " and " << e2.id << " got locks" << std::endl;
        e1.lunch_partners.push_back(e2.id);
        e2.lunch_partners.push_back(e1.id);
    }
    send_mail(e1, e2);
    send_mail(e2, e1);
}
 
int main()
{
    Employee alice("alice"), bob("bob"), christina("christina"), dave("dave");
 
    // assign in parallel threads because mailing users about lunch assignments
    // takes a long time
    std::vector<std::thread> threads;
    threads.emplace_back(assign_lunch_partner, std::ref(alice), std::ref(bob));
    threads.emplace_back(assign_lunch_partner, std::ref(christina), std::ref(bob));
    threads.emplace_back(assign_lunch_partner, std::ref(christina), std::ref(alice));
    threads.emplace_back(assign_lunch_partner, std::ref(dave), std::ref(bob));
 
    for (auto &thread : threads) thread.join();
    std::cout << alice.output() << '\n'  << bob.output() << '\n'
              << christina.output() << '\n' << dave.output() << '\n';
}

```

# [译注： 以上的线程函数需要锁定2个mutex，这就有可能造成死锁。譬如，thread 1要锁定A、B而刚锁定了A，thread 2要锁定B、C而只锁定了B，thread 3要锁定C、A而只锁定了A，这就造成了死锁。std::lock(lock1,lock2,...lockn)就是为了解决这种需要线程函数需要同时锁定多个锁而引起的死锁问题的。笔者猜测，基本原理是按顺序锁，比如资源A、B、C，必须按照此顺序进行锁定，若没有获得A的锁，就不能够去锁定B和C。这是一种解决死锁的方法。]
