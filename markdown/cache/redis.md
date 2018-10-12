# redis
---
## 一 5种结构类型
### 1.1 String(simple dynamic string)
存储的值：字符串、整数、浮点数<br>
结构的读写能力：对整个字符串或字符串中的一部分执行操作，整数和浮点数的自增(increment)、自减操作(decrement)
#### 1.1.1 结构
    struct sds{
        //记录buf数组中已经使用的字节数量
        //等于sds已经保存的字符串的长度
        int length;
        //记录buff中未使用的字节数量
        int free;
        //字节数组
        char buf[];
    }sds;    
使用结构体的方式，结构体里面包括字符串的字符数组，字符的长度，字符的最后一位保留了和c语言语法一样的\0作为结尾
#### 1.1.2 string实现的特性
(1) O(1)复杂度获取字符串的长度<br>
(2) 杜绝缓冲区溢出<br>

    char a[10] = "hello";               
    strcat(a, " world");               
    strcpy(a, "hello world"); 
(3) 减少修改字符串时带来的内存重分配次数 ：空间预分配、惰性空间释放<br>
当对SDS修改的时候如果len的属性值小于1MB则分配两倍len长度的未使用空间，len的属性值大于1MB的时候分配1MB的未使用空间。当SDS修改时候，减少的已使用空间被分配到未使用的部分，保留在SDS中。<br>
(4) 二进制安全<br>
C语言使用’\0’作为判定字符串的结尾，如果你保存的字符串内存在’\0’，c语言自会识别前面的数据，导致后面的数据被忽略，所以是不安全的。而redis是使用了独立的len，这样可以保证即使存储的数据中有’\0’这样的字符，它也是可以支持读取的。<br>
(5) 兼容部分C的字符串函数

### 1.2 List

存储的值：存储的值是一个链表，链表上的每个值都是一个string<br>
结构的读写能力：从链表两端推入或弹出数据、根据偏移量对链表修剪(trim)、读取单个或多个元素、根据值查找或删除元素
#### 1.2.1 结构
		/*
		redis中实现的链表和其他的高级语言是一样的逻辑，使用前节点和后节点的结构 
		实现双向来链表。
		*/
		typdef struct listNode{
            //前节点
            struct listNode *prev;
            //后节点
            struct listNode *next;
            //节点值
            void *value; 
            }listNode;
            /*
            list
		*/
		
		typedef struct list{
            //表头结点
            listNode *head;
            //表尾节点
            listNode *tail;
            //链表所包含的节点数量
            unsigned long length;
            //节点值复制函数
            void *(*dup)(void *ptr);
            //节点值释放函数
            void (*free)(void *ptr);
            //结点值对比函数
            int (*match)(void *ptr, void *key)
		}list;

#### 1.2.2 链表实现的特性
|特点|
|:-|
|双端|
|无环|
|带有表的头指针和尾指针|
|带有表长度的计数器|
|可以保存各种不同类型的值|

### 1.3 Set


