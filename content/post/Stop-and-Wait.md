---
title: "C 语言实现对 Stop-and-Wait 协议的模拟"
date: 2023-09-15:01:37+08:00
comments: true
categories:
    - Computer Network
---

## 协议设计

|~|事件|动作|
|:-:|:-:|:-:|
|发送方|从应用层收到数据|若处于等待上层数据状态，产生一个分组并发送，启动计时器；若处于等待ACK状态，将数据存入缓冲区|
|发送方|超时|重传当前未确认的数据包|
|发送方|收到ACK|若对应当前数据包的ACK，停止计时器，开始发送缓冲区中的下一个数据包；若ACK不对应当前数据包，重传；|
|接收方|收到无差错的数据包|为数据包发送一个对应的ACK，若不是冗余分组，将分组向上层传输|
|接收方|收到损坏的数据包|发送上一个ACK|

## 协议实现

### *缓冲区实现*

缓冲区结构定义如下：

```C
  typedef struct node {
    Pkt* packet;
    struct node* next;
  }Node;
  Node* head_A;  //A缓冲区链表的头节点
  Node* tail_A;  //A缓冲区链表的尾节点
```

- 缓冲区使用用链表实现的队列，维护指向队头和队尾的两个指针
- 初始化缓冲区时初始化表头指针，尾指针指向表头指针
- 接收到来自应用层的信息时将数据封装为数据包，分配内存进行存储，新建一个对应链表节点入队；接收到ACK后将对应的数据包出队，释放内存

### *普通数据包的封装、检验与发送*

```C
  //发送数据包
  void send_packet(int calling_entity, Pkt *packet);
  //计算检验和
  int count_checksum(Pkt *packet);
  //检验检验和
  int checksum_valid(Pkt *packet);
  //在特定内存位置生成一个数据包
  void make_packet(Pkt *packet, int sequmn, int acknum, Msg message);
```

- 将数据包的封装、检验、发送都封装在对应的函数中以供调用
- 数据包的seqnum设置为与缓冲区存储位置对应的序号，acknum设置为-1来指示这是一个普通数据包，而非ACK
- 数据包的checksum设置为数据包的seqmun，acknum和数据包信息包含的20个字符对应的整型数之和
- 检验数据包是否损坏时重新计算检验和，若与数据包的检验和相等，则证明数据包没有损坏

### *ACK数据包的生成与判别*

```C
  //发送ACK数据包
  void send_ack(int calling_entity, int acknum)
```

- 将ACK数据包的生成封装在固定的函数中以供调用
- 使用累计确认的方法，ACKn表示在序号为n之前的数据包都已经被正确接收
- ACK数据包的seqnum设置为-1用来指示这是一个ACK数据包而非普通数据包

### *发送端工作流程*

维护缓冲区和上一个发送的数据包的序号last_sent_seqnum

- A_init()
  1. 初始化缓冲区队列
  2. last_sent_seqnum初始化为0
- A_output()
  1. 生成数据包，存入缓冲区
  2. 如果是缓冲区唯一的数据包，发送；否则退出
- A_input()
  1. 检验收到的数据包是否完整，如果数据包损坏，退出
  2. 如果收到的ACK对应的是当前数据包，将已确认的数据包出队，释放内存，发送队列中下一个数据包
  3. 如果收到的ACK对应的不是当前数据包，重新发送当前数据包
- A_timerinterrupt()
  1. 重新发送当前数据包

### *接收端工作流程*

维护上一个收到的数据包序号last_received_seqnum

- B_init()
  1. last_received_seqnum初始化为-2
- B_input()
  1. 检验收到的数据包是否完整，如果数据包损坏，发送上一个数据包对应的ACK，退出
  2. 发送数据包对应的ACK
  3. 检查是否是冗余数据包，若不是冗余的，将数据包内的信息传给应用层

## 代码

```C
#include <stdio.h>
#include <stdlib.h>

/* ******************************************************************
 ALTERNATING BIT AND GO-BACK-N NETWORK EMULATOR: VERSION 1.1  J.F.Kurose

   This code should be used for PA2, unidirectional or bidirectional
   data transfer protocols (from A to B. Bidirectional transfer of data
   is for extra credit and is not required).  Network properties:
   - one way network delay averages five time units (longer if there
     are other messages in the channel for GBN), but can be larger
   - packets can be corrupted (either the header or the data portion)
     or lost, according to user-defined probabilities
   - packets will be delivered in the order in which they were sent
     (although some can be lost).
**********************************************************************/

#define BIDIRECTIONAL 0    /* change to 1 if you're doing extra credit */
                           /* and write a routine called B_output */

#define   A    0
#define   B    1

/* a "msg" is the data unit passed from layer 5 (teachers code) to layer  */
/* 4 (students' code).  It contains the data (characters) to be delivered */
/* to layer 5 via the students transport level protocol entities.         */
struct msg {
  char data[20];
  };

/* a packet is the data unit passed from layer 4 (students code) to layer */
/* 3 (teachers code).  Note the pre-defined packet structure, which all   */
/* students must follow. */
struct pkt {
   int seqnum;
   int acknum;
   int checksum;
   char payload[20];
    };

/********* STUDENTS WRITE THE NEXT SEVEN ROUTINES *********/

/**说明:
    acknum = -1表示不是ack
*/

#define TIMER_INCRESEMENT 20.0

typedef struct pkt Pkt;
typedef struct msg Msg;

typedef struct node {
  Pkt* packet;
  struct node* next;
}Node;

Node* head_A;  //A缓冲区链表的头节点
Node* tail_A;  //A缓冲区链表的尾节点
Node* head_B;  //B缓冲区链表的头节点
Node* tail_B;  //B缓冲区链表的尾节点

int last_received_seqnum_B;  //上一个传送到B应用层的数据包的seqnum
int last_sent_seqnum_A;  //上一个从A传输层发送的数据包的seqnum

int tolayer3(int, Pkt);
int starttimer(int, float);
int stoptimer(int);
int tolayer5(int AorB, char datasent[20]);

int init();
int generate_next_arrival();

//释放缓冲区内存
void
make_free(Node* node)
{
  free(node->packet);
  free(node);
}

//发送数据包
void
send_packet(int calling_entity, Pkt *packet)
{
  tolayer3(calling_entity, *packet);
  starttimer(calling_entity, TIMER_INCRESEMENT);
}

//计算检验和
int
count_checksum(Pkt* packet)
{
  int sum = 0;
  int i;

  for(i = 0; i < 20; ++i) {
    sum += packet->payload[i];
  }
  sum = sum + packet->acknum + packet->seqnum;
  return sum;
}

//检验检验和
int
checksum_valid(Pkt *packet)
{
  if(packet->checksum == count_checksum(packet))
    return 1;
  else
    return 0;
}

void
send_ack(int calling_entity, int acknum)
{
  Pkt packet;
  int i;

  packet.acknum = acknum;
  for(i = 0; i < 20; ++i) {
    packet.payload[i] = '\0';
  }
  packet.seqnum = -1;
  packet.checksum = count_checksum(&packet);
  tolayer3(calling_entity, packet);
}

//生成一个数据包
Pkt*
make_packet(int sequmn, int acknum, Msg message)
{
  Pkt* packet = malloc(sizeof(Pkt));
  int i;

  packet->seqnum = sequmn;
  packet->acknum = acknum;
  packet->checksum = 0;
  for(i = 0; i < 20; ++i) {
    packet->payload[i] = message.data[i];
  }
  packet->checksum = count_checksum(packet);
  return packet;
}

/* called from layer 5, passed the data to be sent to other side */
void
A_output(message)
  struct msg message;
{
  int seqnum;

  printf("A_output()--Got message from application layer, processing......\n");
  //确定序列号
  seqnum = !last_sent_seqnum_A;
  last_sent_seqnum_A = seqnum;
  //生成数据包,acknum = -1表示不是ack
  Pkt* packet = make_packet(seqnum, -1, message);
  //将数据包添加到缓冲区
  Node* node = malloc(sizeof(Node));
  node->packet = packet;
  tail_A->next = node;
  tail_A = node;
  //如果是缓冲区唯一的数据包,发送;否则等待收到ack或者定时器截止
  if(head_A->next == tail_A) {
    printf("A_output()--Sending packet to network layer, with seqnum = %d......\n", head_A->next->packet->seqnum);
    send_packet(A, head_A->next->packet);
  }
}

void
B_output(message)  /* need be completed only for extra credit */
  struct msg message;
{
}

/* called from layer 3, when a packet arrives for layer 4 */
void
A_input(packet)
  struct pkt packet;
{
  Node* temp;

  printf("A_input()--Got packet from networ layer, checking packet......\n");
  //如果数据包有比特差错
  if(!checksum_valid(&packet)) {
    printf("A_input()--Got corrupted packet from network layer, resending last packet\n");
    stoptimer(A);
    send_packet(A, head_A->next->packet);
    return;
  }
  //如果收到ack
  if(packet.acknum != -1) {
    printf("A_input()--Receied ack with acknum = %d, processing......\n", packet.acknum);
    stoptimer(A);
    //如果收到当前缓冲区头部数据包对应的ack,发送下一个数据包
    if(packet.acknum == head_A->next->packet->seqnum) {
      //将已经确认发送成功的数据包移出缓冲区
      printf("A_input()--Got ACK with expected acknum, with acknum = %d, moving to next packet......\n", packet.acknum);
      temp = head_A->next;
      head_A->next = temp->next;
      if(tail_A == temp) {
        tail_A = head_A;
      }
      make_free(temp);
      //如果缓冲区中存在下一个待发送的数据包，发送
      if(head_A != tail_A) {
        send_packet(A, head_A->next->packet);
      }
    }
    //否则重新发送当前数据包
    else {
      printf("A_input()--Got rebundant ACK, with acknum = %d\n, resending packet......", packet.acknum);
      if(head_A->next != NULL)
        send_packet(A, head_A->next->packet);
    }
  }
  //如果是其他数据包
  else {
    printf("A_input()--Got normal packet from network layer, with seqnum = %d\n", packet.seqnum);
  }
}

/* called when A's timer goes off */
void
A_timerinterrupt()
{
  printf("A_timerinterrupt()--Calling A's timerinterrupt, resending packet with seqnum = %d......\n", head_A->next->packet->seqnum);
  send_packet(A, head_A->next->packet);
}  

/* the following routine will be called once (only) before any other */
/* entity A routines are called. You can use it to do any initialization */
void
A_init()
{
  printf("A_init()--\n");
  head_A = malloc(sizeof(Node));
  head_A->next = NULL;
  tail_A = head_A;
  last_sent_seqnum_A = 0;
}


/* Note that with simplex transfer from a-to-B, there is no B_output() */

/* called from layer 3, when a packet arrives for layer 4 at B*/
void
B_input(packet)
  struct pkt packet;
{
  printf("B_input()--Got packet from network layer, checking packet......\n");
  //如果数据包有比特差错,发送上一个数据包的ack
  if(!checksum_valid(&packet)) {
    printf("B_input()--Got corrupted packet from network layer, resending last packet......\n");
    send_ack(B, last_received_seqnum_B);
    return;
  }
  //如果收到ack
  if(packet.acknum == 1 || packet.acknum == 0) {
    printf("B_input()--Got ACK from network layer, with acknum = %d\n", packet.acknum);
  }
  //如果是其他数据包
  else {
    printf("B_input()--Got normal packet from network layer, with seqnum = %d\n", packet.seqnum);
    //发送ack
    printf("B_input()--Sending ack to network layer with acknum = %d......\n", packet.seqnum);
    send_ack(B, packet.seqnum);
    //检查是否冗余
    if(packet.seqnum != last_received_seqnum_B) {
      //发送到应用层
      printf("B_input()--Got packet with expected seqnum, sending message to application layer......\n");
      tolayer5(B, packet.payload);
      last_received_seqnum_B = packet.seqnum;
    }
  }
}

/* called when B's timer goes off */
void
B_timerinterrupt()
{
}

/* the following rouytine will be called once (only) before any other */
/* entity B routines are called. You can use it to do any initialization */
void
B_init()
{
  printf("B_init()--\n");
  last_received_seqnum_B = -2;
}


/*****************************************************************
***************** NETWORK EMULATION CODE STARTS BELOW ***********
The code below emulates the layer 3 and below network environment:
  - emulates the tranmission and delivery (possibly with bit-level corruption
    and packet loss) of packets across the layer 3/4 interface
  - handles the starting/stopping of a timer, and generates timer
    interrupts (resulting in calling students timer handler).
  - generates message to be sent (passed from later 5 to 4)

THERE IS NOT REASON THAT ANY STUDENT SHOULD HAVE TO READ OR UNDERSTAND
THE CODE BELOW.  YOU SHOLD NOT TOUCH, OR REFERENCE (in your code) ANY
OF THE DATA STRUCTURES BELOW.  If you're interested in how I designed
the emulator, you're welcome to look at the code - but again, you should have
to, and you defeinitely should not have to modify
******************************************************************/

struct event {
   float evtime;           /* event time */
   int evtype;             /* event type code */
   int eventity;           /* entity where event occurs */
   struct pkt *pktptr;     /* ptr to packet (if any) assoc w/ this event */
   struct event *prev;
   struct event *next;
 };
struct event *evlist = NULL;   /* the event list */

int insertevent(struct event *);

/* possible events: */
#define  TIMER_INTERRUPT 0  
#define  FROM_LAYER5     1
#define  FROM_LAYER3     2

#define  OFF             0
#define  ON              1



int TRACE = 1;             /* for my debugging */
int nsim = 0;              /* number of messages from 5 to 4 so far */ 
int nsimmax = 0;           /* number of msgs to generate, then stop */
float time = 0.000;
float lossprob;            /* probability that a packet is dropped  */
float corruptprob;         /* probability that one bit is packet is flipped */
float lambda;              /* arrival rate of messages from layer 5 */   
int   ntolayer3;           /* number sent into layer 3 */
int   nlost;               /* number lost in media */
int ncorrupt;              /* number corrupted by media*/

int
main()
{
   struct event *eventptr;
   struct msg  msg2give;
   struct pkt  pkt2give;
   
   int i,j;
   char c; 
  
   init();
   A_init();
   B_init();
   
   while (1) {
        eventptr = evlist;            /* get next event to simulate */
        if (eventptr==NULL)
           goto terminate;
        evlist = evlist->next;        /* remove this event from event list */
        if (evlist!=NULL)
           evlist->prev=NULL;
        if (TRACE>=2) {
           printf("\nEVENT time: %f,",eventptr->evtime);
           printf("  type: %d",eventptr->evtype);
           if (eventptr->evtype==0)
        printf(", timerinterrupt  ");
             else if (eventptr->evtype==1)
               printf(", fromlayer5 ");
             else
      printf(", fromlayer3 ");
           printf(" entity: %d\n",eventptr->eventity);
           }
        time = eventptr->evtime;        /* update time to next event time */
        if (nsim==nsimmax)
   break;                        /* all done with simulation */
        if (eventptr->evtype == FROM_LAYER5 ) {
            generate_next_arrival();   /* set up future arrival */
            /* fill in msg to give with string of same letter */    
            j = nsim % 26; 
            for (i=0; i<20; i++)  
               msg2give.data[i] = 97 + j;
            if (TRACE>2) {
               printf("          MAINLOOP: data given to student: ");
                 for (i=0; i<20; i++) 
                  printf("%c", msg2give.data[i]);
               printf("\n");
      }
            nsim++;
            if (eventptr->eventity == A) 
               A_output(msg2give);  
             else
               B_output(msg2give);  
            }
          else if (eventptr->evtype ==  FROM_LAYER3) {
            pkt2give.seqnum = eventptr->pktptr->seqnum;
            pkt2give.acknum = eventptr->pktptr->acknum;
            pkt2give.checksum = eventptr->pktptr->checksum;
            for (i=0; i<20; i++)  
                pkt2give.payload[i] = eventptr->pktptr->payload[i];
     if (eventptr->eventity ==A)      /* deliver packet by calling */
           A_input(pkt2give);            /* appropriate entity */
            else
           B_input(pkt2give);
     free(eventptr->pktptr);          /* free the memory for packet */
            }
          else if (eventptr->evtype ==  TIMER_INTERRUPT) {
            if (eventptr->eventity == A) 
        A_timerinterrupt();
             else
        B_timerinterrupt();
             }
          else  {
      printf("INTERNAL PANIC: unknown event type \n");
             }
        free(eventptr);
        }

terminate:
   printf(" Simulator terminated at time %f\n after sending %d msgs from layer5\n",time,nsim);
}

int
init()                         /* initialize the simulator */
{
  int i;
  float sum, avg;
  float jimsrand();
  
  
   printf("-----  Stop and Wait Network Simulator Version 1.1 -------- \n\n");
   printf("Enter the number of messages to simulate: ");
   scanf("%d",&nsimmax);
   printf("Enter  packet loss probability [enter 0.0 for no loss]:");
   scanf("%f",&lossprob);
   printf("Enter packet corruption probability [0.0 for no corruption]:");
   scanf("%f",&corruptprob);
   printf("Enter average time between messages from sender's layer5 [ > 0.0]:");
   scanf("%f",&lambda);
   printf("Enter TRACE:");
   scanf("%d",&TRACE);

   srand(9999);              /* init random number generator */
   sum = 0.0;                /* test random number generator for students */
   for (i=0; i<1000; i++)
      sum=sum+jimsrand();    /* jimsrand() should be uniform in [0,1] */
   avg = sum/1000.0;
   if (avg < 0.25 || avg > 0.75) {
    printf("It is likely that random number generation on your machine\n" ); 
    printf("is different from what this emulator expects.  Please take\n");
    printf("a look at the routine jimsrand() in the emulator code. Sorry. \n");
    exit(-1);
    }

   ntolayer3 = 0;
   nlost = 0;
   ncorrupt = 0;

   time=0.0;                    /* initialize time to 0.0 */
   generate_next_arrival();     /* initialize event list */
}

/****************************************************************************/
/* jimsrand(): return a float in range [0,1].  The routine below is used to */
/* isolate all random number generation in one location.  We assume that the*/
/* system-supplied rand() function return an int in therange [0,mmm]        */
/****************************************************************************/
float jimsrand() 
{
  double mmm = 0x7fffffff;   /* largest int  - MACHINE DEPENDENT!!!!!!!!   */
  float x;                   /* individual students may need to change mmm */ 
  x = rand()/mmm;            /* x should be uniform in [0,1] */
  return(x);
}  

/********************* EVENT HANDLINE ROUTINES *******/
/*  The next set of routines handle the event list   */
/*****************************************************/

int
generate_next_arrival()
{
   double x,log(),ceil();
   struct event *evptr;
   //char *malloc();
   float ttime;
   int tempint;

   if (TRACE>2)
       printf("          GENERATE NEXT ARRIVAL: creating new arrival\n");
 
   x = lambda*jimsrand()*2;  /* x is uniform on [0,2*lambda] */
                             /* having mean of lambda        */
   evptr = (struct event *)malloc(sizeof(struct event));
   evptr->evtime =  time + x;
   evptr->evtype =  FROM_LAYER5;
   if (BIDIRECTIONAL && (jimsrand()>0.5) )
      evptr->eventity = B;
    else
      evptr->eventity = A;
   insertevent(evptr);
} 

int
insertevent(p)
   struct event *p;
{
   struct event *q,*qold;

   if (TRACE>2) {
      printf("            INSERTEVENT: time is %lf\n",time);
      printf("            INSERTEVENT: future time will be %lf\n",p->evtime); 
      }
   q = evlist;     /* q points to header of list in which p struct inserted */
   if (q==NULL) {   /* list is empty */
        evlist=p;
        p->next=NULL;
        p->prev=NULL;
        }
     else {
        for (qold = q; q !=NULL && p->evtime > q->evtime; q=q->next)
              qold=q; 
        if (q==NULL) {   /* end of list */
             qold->next = p;
             p->prev = qold;
             p->next = NULL;
             }
           else if (q==evlist) { /* front of list */
             p->next=evlist;
             p->prev=NULL;
             p->next->prev=p;
             evlist = p;
             }
           else {     /* middle of list */
             p->next=q;
             p->prev=q->prev;
             q->prev->next=p;
             q->prev=p;
             }
         }
}

int
printevlist()
{
  struct event *q;
  int i;
  printf("--------------\nEvent List Follows:\n");
  for(q = evlist; q!=NULL; q=q->next) {
    printf("Event time: %f, type: %d entity: %d\n",q->evtime,q->evtype,q->eventity);
    }
  printf("--------------\n");
}



/********************** Student-callable ROUTINES ***********************/

/* called by students routine to cancel a previously-started timer */
int
stoptimer(AorB)
int AorB;  /* A or B is trying to stop timer */
{
 struct event *q,*qold;

 if (TRACE>2)
    printf("          STOP TIMER: stopping timer at %f\n",time);
/* for (q=evlist; q!=NULL && q->next!=NULL; q = q->next)  */
 for (q=evlist; q!=NULL ; q = q->next) 
    if ( (q->evtype==TIMER_INTERRUPT  && q->eventity==AorB) ) { 
       /* remove this event */
       if (q->next==NULL && q->prev==NULL)
             evlist=NULL;         /* remove first and only event on list */
          else if (q->next==NULL) /* end of list - there is one in front */
             q->prev->next = NULL;
          else if (q==evlist) { /* front of list - there must be event after */
             q->next->prev=NULL;
             evlist = q->next;
             }
           else {     /* middle of list */
             q->next->prev = q->prev;
             q->prev->next =  q->next;
             }
       free(q);
       return 0;
     }
  printf("Warning: unable to cancel your timer. It wasn't running.\n");
}

int
starttimer(AorB,increment)
int AorB;  /* A or B is trying to stop timer */
float increment;
{

 struct event *q;
 struct event *evptr;
 //char *malloc();

 if (TRACE>2)
    printf("          START TIMER: starting timer at %f\n",time);
 /* be nice: check to see if timer is already started, if so, then  warn */
/* for (q=evlist; q!=NULL && q->next!=NULL; q = q->next)  */
   for (q=evlist; q!=NULL ; q = q->next)  
    if ( (q->evtype==TIMER_INTERRUPT  && q->eventity==AorB) ) { 
      printf("Warning: attempt to start a timer that is already started\n");
      return 0;
      }
 
/* create future event for when timer goes off */
   evptr = (struct event *)malloc(sizeof(struct event));
   evptr->evtime =  time + increment;
   evptr->evtype =  TIMER_INTERRUPT;
   evptr->eventity = AorB;
   insertevent(evptr);
} 


/************************** TOLAYER3 ***************/
int
tolayer3(AorB,packet)
int AorB;  /* A or B is trying to stop timer */
struct pkt packet;
{
 struct pkt *mypktptr;
 struct event *evptr,*q;
 //char *malloc();
 float lastime, x, jimsrand();
 int i;


 ntolayer3++;

 /* simulate losses: */
 if (jimsrand() < lossprob)  {
      nlost++;
      if (TRACE>0)    
        printf("          TOLAYER3: packet being lost\n");
      return 0;
    }  

/* make a copy of the packet student just gave me since he/she may decide */
/* to do something with the packet after we return back to him/her */ 
 mypktptr = (struct pkt *)malloc(sizeof(struct pkt));
 mypktptr->seqnum = packet.seqnum;
 mypktptr->acknum = packet.acknum;
 mypktptr->checksum = packet.checksum;
 for (i=0; i<20; i++)
    mypktptr->payload[i] = packet.payload[i];
 if (TRACE>2)  {
   printf("          TOLAYER3: seq: %d, ack %d, check: %d ", mypktptr->seqnum,
   mypktptr->acknum,  mypktptr->checksum);
    for (i=0; i<20; i++)
        printf("%c",mypktptr->payload[i]);
    printf("\n");
   }

/* create future event for arrival of packet at the other side */
  evptr = (struct event *)malloc(sizeof(struct event));
  evptr->evtype =  FROM_LAYER3;   /* packet will pop out from layer3 */
  evptr->eventity = (AorB+1) % 2; /* event occurs at other entity */
  evptr->pktptr = mypktptr;       /* save ptr to my copy of packet */
/* finally, compute the arrival time of packet at the other end.
   medium can not reorder, so make sure packet arrives between 1 and 10
   time units after the latest arrival time of packets
   currently in the medium on their way to the destination */
 lastime = time;
/* for (q=evlist; q!=NULL && q->next!=NULL; q = q->next) */
 for (q=evlist; q!=NULL ; q = q->next) 
    if ( (q->evtype==FROM_LAYER3  && q->eventity==evptr->eventity) ) 
      lastime = q->evtime;
      evptr->evtime =  lastime + 1 + 9*jimsrand();
 


 /* simulate corruption: */
 if (jimsrand() < corruptprob)  {
    ncorrupt++;
    if ( (x = jimsrand()) < .75)
       mypktptr->payload[0]='Z';   /* corrupt payload */
      else if (x < .875)
       mypktptr->seqnum = 999999;
      else
       mypktptr->acknum = 999999;
    if (TRACE>0)    
 printf("          TOLAYER3: packet being corrupted\n");
    }  

  if (TRACE>2)  
     printf("          TOLAYER3: scheduling arrival on other side\n");
  insertevent(evptr);
} 

int
tolayer5(AorB,datasent)
  int AorB;
  char datasent[20];
{
  int i;  
  if (TRACE>2) {
     printf("          TOLAYER5: data received: ");
     for (i=0; i<20; i++)  
        printf("%c",datasent[i]);
     printf("\n");
   }
  
}
```
