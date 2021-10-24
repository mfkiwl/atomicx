# CorePartitionOS

Version 2.7.0 release

![image](https://user-images.githubusercontent.com/1805792/125191254-6591cf80-e239-11eb-9e89-d7500e793cd4.png)

What is atomic? Atomic is a general purpose **cooperative** thread lib for embedded applications (single core or confined within other RTOS) that allows you partition your application "context" (since core execution) into several controlled context using cooperative thread. So far here nothing out of the ordinary, right? Lets think again:

* **DO NOT DISPLACE STACK, IT WILL STILL AVAILABLE FOR PROCESSING**, the *Stack Page* will only hold a backup of the most necessary information needed, allowing stacks in few bites most if the time. This implementation if highly suitable for Microcontrollers like ATINY85, for example, that only has 512 bites, and you can have 5 or more threads doing things for you, only backup the most important context information.
* Since it implements Cooperative thread every execution will atomic between *atomicx* thrteads.
* CoreX **DO NOT DISPLACE STACK**, yes, it will use a novel technique that allow you to use full stack memory freely, and once done, just call `Yield()` to switch the context.
    1. Allow you to use all your stack during thread execution and only switch once back to an appropriate place
    ``` 
        Stack memory
        *-----------*
        |___________| Yield()
        |___________|    thread 0..N
        |___________|     |       .  - After execution execution
        |___________|     |      /|\   is done, the developer can
        |___________|     |       |    choose wether to switch 
        |___________|     |       |    context, using only what is
        |___________|    \|/      |    necessary
        |___________|     ---------
        |           |     - During context
        *-----------*       can goes deeper as 
                            necessary
    ```
* Due to the **zero stack-displacement** technology, developers can ensure minimal stack memory page, allowing ultra sophisticated designes and execution stack diving and only backing up to the stack memory page what is necessary.
                           
* Full feature for IPC (_Inter Process Communication_)
    * Thread safe Queues for data communication
    * EVERY Smart Lock can transport information ab atomicx::message
    * Message is composed by "size_t atomix::message" and a "size_t tag"
        * This novel concept of "tag"s for an atomicx::message gives the message meaning.
        * Since atomicx::message uses "size_t" messages can also transport pointers
    * Smart Locks can Lock and Shared Lock in the same object, making
    * Full QUEUE capable to transport objects. 

* Full feature for IPN (_Inter Process Notification_)
    * Thread can wait for an event to happen.
    * On event notification a atomix::message can be sent/received
    
* A message broker based on observer pattern
    * A thread can subscribe (in a more dynamic way) to any information, instead of adding subscriptions, the process method to confirm if it IsSubscribed. This will allow async messages to be received.
    * Sync wait for broker messages.
    * Also uses atomicx::message

* ALL *WAIT* actions will be block the thread, on thread level, till the message or notification occurs.
 
* **IMPORTANT** since all threads will be executed in the "regular" stack, it will not be jailed in the stack memory page, *DO NOT USE STACK ADDRESS TO COMMUNICATE with another threads or source of information that could be read by others thread.

``` C++
//
//  main.cpp
//  atomic
//
//  Created by GUSTAVO CAMPOS on 28/08/2021.
//

#include <unistd.h>
#include <sys/time.h>
#include <unistd.h>

#include <cstring>
#include <cstdint>
#include <iostream>
#include <setjmp.h>
#include <string>

#include "atomic.hpp"

using namespace thread;

#ifdef FAKE_TIMER
uint nCounter=0;
#endif

void ListAllThreads();

atomic_time Atomic_GetTick (void)
{
#ifndef FAKE_TIMER
    usleep (20000);
    struct timeval tp;
    gettimeofday (&tp, NULL);

    return (atomic_time)tp.tv_sec * 1000 + tp.tv_usec / 1000;
#else
    nCounter++;

    return nCounter;
#endif
}

void Atomic_SleepTick(atomic_time nSleep)
{
#ifndef FAKE_TIMER
    usleep ((useconds_t)nSleep * 1000);
#else
    while (nSleep); usleep(100);
#endif
}


class Thread : public atomic
{
public:
    Thread(atomic_time nNice, const char* pszName) : stack{}, atomic (stack), m_pszName(pszName)
    {
        SetNice(nNice);
    }

    ~Thread()
    {
        std::cout << "Deleting " << GetName() << ": " << (size_t) this << std::endl;
    }
    
    void run() noexcept override
    {
        size_t nCount=0;
        
        do
        {
            std::cout << __FUNCTION__ << ", Executing " << GetName() << ": " << (size_t) this << ", Counter: " << nCount << std::endl << std::flush
                        
            nCount++;
                        
        }  while (Yield());

    }

    void StackOverflowHandler (void) override
    {
        std::cout << __FUNCTION__ << ":" << GetName() << "_" << (size_t) this << ": needed: " << GetUsedStackSize() << ", allocated: " << GetStackSize() << std::endl;
    }

    const char* GetName (void) override
    {
        return m_pszName;
    }

private:
    uint8_t stack[1024];
    const char* m_pszName;
};


int main()
{
    Thread t1(200, "Producer 1");
    Thread t2(500, "Producer 2");
    

    std::cout << "Start contexts for thread 3" << std::endl;

    {
        Thread t3_1(0, "Eventual 3.1");
        Thread t3_2(0, "Eventual 3.2");
        Thread t3_3(0, "Eventual 3.3");
    }
    
    
    std::cout << "end context" << std::endl;

    Thread t4(1000, "Producer 4");

    atomic::Start();
    
}

```
