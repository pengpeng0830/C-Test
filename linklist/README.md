# linklist（单向链表）
- linklist.c 是一个单向链表，实现了链表的创建，插入节点，删除节点，查找节点，删除所有节点，历遍节点并打印。插入和删除节点都是通过具体位置定位的。
- linklist-V02.c 是一个单向链表，和linklist.c功能一样。修改了插入删除查找节点的方式。插入节点只可以插入到头或尾，然后返回节点地址。删除节点通过找到该节点地址实现。查找节点是通过查找的数据和链表里的数据相同，返回链表对应的地址。
# Queue（队列）
- Queue.c 是一个队列，实现了创建队列，数据入列和出列。队列的头是有数据的，数据出列是在头出列；尾是没有数据的，尾的前一个位置存放最后入列的数据。目的是方便区分队列的数据是空还是满，头=尾，说明队列空。头=尾+1，说明队列满。
# softtimer（基于单向链表实现的软件定时器）
- 创建一个空的头节点，这个头节点没有有效数据，只是作为一个链表的头，不可删除，插入的节点也只能插入到它后面。  
- 配置一个硬件定时器，每1ms发生一次中断，毫秒计数一次，计数不断的从0-0xffff循环计数。当创建一个新的定时器，要获取一下当前的毫秒计数为开始时间点，比如是5000，然后定时周期是1秒，那1秒以后，毫秒计数到6000，就可以判断定时时间到。然后更新一下开始时间点，更新为6000，那么下次就要到7000的时候会完成定时。以此类推。这是基本的设计原理。  
- 完成该定时器用到的结构体  
typedef struct _T_SOFTTIMER  
{  
    PF_SOFTTIMER_CB pfSofttimerCB;    /* 回调函数 */  
    uint8  u8TaskID;                  /* 定时器ID */  
    uint16 u16OldTime;                /* 基准时间 */  
    uint16 u16Period;                 /* 定时周期 */  
    uint16 u16Count;                  /* 执行次数 */  
}T_SOFTTIMER;  
定时器在嵌入式研发中是非常常用也是非常基础的。目前想到的参数是这么多，后期会随着学习的深入再优化该定时器。  
- 定时器时间到肯定是要干一些事情的，这个事情通过一个回调函数完成，类型是typedef uint32 (*PF_SOFTTIMER_CB)(void *pPara); pPara是一个指针，所以可以输入任何类型的参数。每个任务都需要一个自己的ID用作区分。基准时间就是上面提到的开始时间点，只要当前时间减去基准时间大于等于定时周期，就完成一次定时。不同的使用场景需要的定时次数可能不相同，有些场合只要定时一次即可，有些场合需要无限次。那可以通过执行次数实现。若为1-65534则代表实际次数，每执行完一次就少一次。65535则表示定时次数无穷。还有一种场景，就是不知道实际需要的定时次数，是在满足一定条件以后，就不需要定时了，那么可以通过删除定时器的方式实现。 
- 关于回调函数的使用，有2种方式，一种是定时时间到，立刻执行。另一种是轮询完所有的定时器节点，然后执行。这里用的方法是后者，如果回调函数的时间比较长，会打断定时器的正常执行。所以先把定时器全部执行完，发现定时到，先置标志。最后通过标志来一一执行回调函数。  
- 关于定时原理的设计，我遇到了一个坑，想了很久才解决，解决以后发现问题这么简单，完全是我考虑的太多，但是既然踩了这个坑，我就记录一下。这里定时器的原理是当前时间-起始时间（就是结构体里的基准时间）>=定时周期，就判断为定时时间到。写代码的时候考虑到，毫秒计时时间转了一圈以后，就是65535的下一个又是从0开始，当前时间-起始时间会<0。我就换了个思路，代码中写的是 当前时间>=定时周期+起始时间。但是这样会引入新的问题，就是定时时间+起始时间会大于65535，然后溢出以后变成一个很小的数，当前时间肯定就大于这个数，造成这个溢出时间段，会永远满足定时完成的判断条件。我想了很多办法去解决该问题，都觉得太麻烦没有去尝试。最后在回到问题的最开始的地方，问题就解决了，代码中只要写 当前时间-起始时间>=定时周期 就可以了。虽然当前时间小于起始时间，可减出来的数肯定不会是负数，而且在测试以后，发现就是我想要的数。举个栗子：当前时间和起始时间都是65535，那减出来是0，那么下一次，当前时间变成了0,0-65535是多少，不太清楚...应该不是1...因为我的C语言不过关，我才会换种写法，然后引入一些新的要解决的问题。对于16位的数，0-65535就是1！那代码中不需要绕弯子，直接写成 当前时间-起始时间>=定时周期 就可以了。
- 说明：代码中已经将基准时间和定时周期都改成了uint32类型。可以支持（2^32）ms的定时，最长约49.7天的时间。
# Event（基于队列实现的事件队列）
- 将所有的事件，都放入一个队列管理。当满足事件的触发条件以后，就将事件插入队列。CPU会不断轮询事件队列是否为空，只要事件不为空，就去执行队列里的事件，直到为空，表示事件都执行完毕。这样就不需要每个事件都去判断是否发生，而只要判断队列是否为空即可。举例说明：有3个按键，按下每个按键会执行相应的任务。按照裸机的思维，在按键中断里置标志位，然后再while(1)里不断的查询这3个标志位是否改变。而利用队列的方式，可以在中断里插入事件，然后在while(1)里只需要检查队列是否为空即可。
