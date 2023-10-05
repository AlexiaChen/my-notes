#源码分析

## 前言

前几天公司的群里讨论了[Nginx](https://en.wikipedia.org/wiki/Nginx)内部实现的一个数据结构，ngx_queue_t。 实质上是一个[双向链表](https://en.wikipedia.org/wiki/Doubly_linked_list)，但是当时讨论没有讨论出结果来，原因我自己也被绕晕了，今天本着从问题本身出发，不被别人的想法思路所左右，不看网络上任何文章的分析，好好写测试用例来证明自己的想法，其实那天的问题是围绕着两个C语言的宏展开的，那么这篇文章也会重点讲解这两个宏的意义：offsetof 和 ngx_queue_data。所以这篇文章不会涉及到链表插入,删除等细节。

## 正文

首先，Nginx源码实现这个数据结构，用了大量的宏，我们先用我们的思路，把关键信息抽取出来:

``` cpp
#define u_char unsigned char  
typedef struct ngx_queue_s  ngx_queue_t;

struct ngx_queue_s {
    ngx_queue_t  *prev;   //前一个  
    ngx_queue_t  *next;   //下一个  
};
```

以上代码就是这个链表的关键，ngx_queue_s 和 ngx_queue_t都是链表的节点类型，结构体中有链表的前驱（prev）和后驱（next），以实现正反向遍历。唉？ 等等，如果大学的时候，上过数据结构这门课的人会有点奇怪，会这样想：链表的节点应该有相关的数据啊，不然链表有什么意义？  是的，链表的节点必须有数据，但是上面这个结构体却没有，这是怎么回事？如果在当时，我想大部分人会写成以下这样:

``` cpp
struct ngx_queue_s {
    int data;             // 数据
    ngx_queue_t  *prev;   //前一个  
    ngx_queue_t  *next;   //下一个  
};
```

这样每访问一个节点的时候，就可以通过节点的指针指向数据了。但是，你有没有想过，万一这个链表的数据是人名呢？不是int型，又或是其他复杂点的类型怎么办？ 是不是又要重新定一个新的链表节点类型？  这样就麻烦了不少，软件工程里面奉行了一个原则，这个原则就是[DRY原则](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)。所以为了让不同类型采用同一个数据结构实现，就需要类型与数据结构实现分离，让这个数据结构的实现适配不同的类型，当然C++中有STL已经帮你做好了，Java也是。但是C语言就不支持泛型，Nginx由于是C语言实现的，所以采用了一种叫寄宿链表的思想，这种思想就是分离数据类型与数据结构实现的。

看到这里，就有人会想，这种思想跟文章要分析的两个宏有什么关系呢？ 现在我可以先放结论，它们之间有莫大的关系，正因为采用了寄宿链表的思想，才会间接产生了那两个宏，这种变形掩盖了一些实质。所以其实宏不是关键，是寄宿链表的思想才是关键！我们需要知道Why，而不是How，不然会深入无限的细节当中去。

话说回来，下面我们来看那两个宏的实现:

``` cpp
// 取struct_type类型的成员相对于struct_type基地址的偏移量（字节数） 一定要是POD 类型
#define offsetof(struct_type, member)   ((size_t) &((struct_type *) 0)->member)   

#define ngx_queue_data(node_addr,struct_type,member)  (struct_type *)((u_char*)node_addr - offsetof(struct_type, member))
```

我们发现ngx_queue_data这个宏复用了offsetof这个宏，那么首先分析offsetof这个宏:

这个宏根据名字可以很容易就看出来，就是字面意思，取某个结构体成员相对于结构体基址的偏移量（字节）。我们写一段代码测试来验证我们的想法:

``` cpp
typedef struct _SFamily{
    int id;
    char addr[256];
    int nums;
}Family;

//编译时断言，所谓的静态断言, 所以offsetof这个宏在编译时就计算出了结构体成员的偏移
static_assert(offsetof(Family, nums) == 260,"nums offset is 260");
static_assert(offsetof(Family, id) == 0, "id offset is 0");
static_assert(offsetof(Family, addr) == 4, "addr offset is 4");
```

根据代码结果，轻而易举的验证了我们的想法，而且这个求偏移是静态编译时就能得出的。不然以上代码直接不能通过编译。

好了，之后我们先别着急研究ngx_queue_data这个宏是怎么回事。先来看看怎么使用这个数据结构，怎样来用操作数据结构的一些方法，Nginx中已经定义好了，但是这些方法，是大量的宏定义的，这些宏不是关键，我把它们都改成了C/C++的函数形式：

``` cpp
#define u_char unsigned char  

// 取struct_type类型的成员相对于struct_type基地址的偏移量（字节数） 一定要是POD 类型
#define offsetof(struct_type, member)   ((size_t) &((struct_type *) 0)->member)   

typedef struct ngx_queue_s  ngx_queue_t;


struct ngx_queue_s {
    ngx_queue_t  *prev;   //前一个  
    ngx_queue_t  *next;   //下一个  
};

void ngx_queue_init(ngx_queue_s* q){
    (q)->prev = q;                                                          
    (q)->next = q;
}

bool ngx_queue_empty(ngx_queue_s* head){
    return head == head->prev;
}

void ngx_queue_insert_head(ngx_queue_s* head, ngx_queue_s* node){
    
    node->next = head->next;
    head->next->prev = node;
    head->next = node;
    node->prev = head;

}

void ngx_queue_insert_after(ngx_queue_s* pos, ngx_queue_s* node){
    ngx_queue_insert_head(pos, node);
}

void ngx_queue_insert_tail(ngx_queue_s* tail, ngx_queue_s* node){

    node->prev = tail->prev;
    tail->prev->next = node;
    tail->prev = node;
    node->next = tail;
}

// 头节点和尾节点都是哑节点(dummy node)，并不存储实际数据
ngx_queue_s* ngx_queue_head(ngx_queue_s* head){
    return head->next;
}

ngx_queue_s* ngx_queue_last(ngx_queue_s* tail){
    return tail->prev;
}

//直接取 head dummy node
ngx_queue_s* ngx_queue_sentinel(ngx_queue_s* head){
    return head;
}

//当前节点的下一个节点
ngx_queue_s* ngx_queue_next(ngx_queue_s* cur){
    return cur->next;
}

//当前节点的上一个节点
ngx_queue_s* ngx_queue_prev(ngx_queue_s* cur){
    return cur->prev;
}

//删除节点
void ngx_queue_remove(ngx_queue_s* node){
    node->prev->next = node->next;
    node->next->prev = node->prev;
    
}

#define ngx_queue_data(node_addr,struct_type,member)  (struct_type *)((u_char*)node_addr - offsetof(struct_type, member))
```

这些函数的用法官方都提供了，我们根据用法，自己实现些测试用例,我们打算实现两个链表，一个是存放书籍信息的，一个是存放人员信息的，写入数据，然后遍历打印它们，看看接口是否正确：

``` cpp
typedef struct _Book{
    ngx_queue_t link;
    int type;
    char name[256];
}Book;

typedef struct _Person{
   
    int sex;
    char name[256];
    ngx_queue_t link;
}Person;

 Book book1, book2, book3;
    Person person1, person2, person3;

    static_assert(offsetof(Book, link) == 0, "link offset is 0");
    static_assert(offsetof(Person, link) == 260, "link offset is 260");

    book1.type = 1;
    book2.type = 2;
    book3.type = 3;

    person1.sex = 1;
    person2.sex = 0;
    person3.sex = 1;

    strcpy(book1.name, "Harry Potter");
    strcpy(book2.name, "the lord of ring");
    strcpy(book3.name, "Jame");

    strcpy(person1.name, "Fan BingBing");
    strcpy(person2.name, "Wu YanZu");
    strcpy(person3.name, "ShuQi");

    //初始化队列
    ngx_queue_t bookQueue;
    ngx_queue_init(&bookQueue);
    ngx_queue_insert_head(&bookQueue, &book1.link);
    ngx_queue_insert_tail(&bookQueue, &book2.link);
    ngx_queue_insert_tail(&bookQueue, &book3.link);

    ngx_queue_t personQueue;
    ngx_queue_init(&personQueue);
    ngx_queue_insert_head(&personQueue, &person1.link);
    ngx_queue_insert_tail(&personQueue, &person2.link);
    ngx_queue_insert_tail(&personQueue, &person3.link);

   
    for (auto iter = ngx_queue_head(&bookQueue); iter != ngx_queue_sentinel(&bookQueue); iter = ngx_queue_next(iter))
    {
        Book* bookPtr = (Book*)ngx_queue_data(iter, Book, link);

        printf("Book name is: %s . type is %d \n", bookPtr->name, bookPtr->type);
    }

    for (auto iter = ngx_queue_head(&personQueue); iter != ngx_queue_sentinel(&personQueue); iter = ngx_queue_next(iter))
    {
        Person* personPtr = (Person*)ngx_queue_data(iter, Person, link);

        printf("Person name is: %s . sex is %s\n", personPtr->name, personPtr->sex == 1 ? "Female" : "Male");
    }
```

以上代码的打印结果如我们所想，是正确的。注意了，我们遍历访问链表单个节点的时候用了ngx_queue_data这个宏，正是我们需要了解的宏。看来跟C语言数组，C++ vector的遍历访问差不多嘛，访问一个节点元素需要的就是节点的下标（或者说地址）。

接下来我们再看链表节点的真正的数据是定义在哪？

``` cpp
typedef struct _Book{
    ngx_queue_t link;
    int type;
    char name[256];
}Book;

typedef struct _Person{
   
    int sex;
    char name[256];
    ngx_queue_t link;
}Person;
```
注意了，以上两个结构体里面都有链表节点类型ngx_queue_t，变量名link，想想，是不是数据类型与数据结构就分离了？
不同的数据类型复用了相同类型的数据结构？ 这样访问节点就可以通过它们的link成员来正反向迭代了，所以之前的代码iter变量实际就是link成员的地址，我们可以访问每个结构体里面的link成员变量了，那么怎么知道结构体其他数据呢？没有地址，不知道在哪啊？  其实关键就是通过link成员的地址来计算出结构体的地址的,下面举个例子:

``` cpp
// iter为链表节点的地址，而链表节点又在Book和Person结构体中
Book* bookPtr = (Book*)ngx_queue_data(iter, Book, link);

Person* personPtr = (Person*)ngx_queue_data(iter, Person, link);
```

看到上面代码并且联系ngx_queue_data宏的实现，是不是就明白了？

**<font color="red">链表节点成员的地址 ➖ 链表节点成员相对于结构体基址的偏移 = 结构体的基址</font>**

那么是不是就可以通过结构体基址的地址强制转换为结构体的指针，访问结构体其他数据成员了？还不明白？ 那么再看下面:

``` cpp
typedef struct _Book{
    ngx_queue_t link; // 这个link变量的地址知道了，是不是就可以知道Book结构体基址了？  link的地址假如为123，link的偏移为0，那么123 - 0 = 123。123为数据的结构体地址
    int type;
    char name[256];
}Book;

typedef struct _Person{
   
    int sex;
    char name[256];
    ngx_queue_t link;// 这个link变量的地址知道了，是不是就可以知道Person结构体基址了？  link的地址假如为300，link的偏移为260，那么300 - 260 = 40。40为数据的结构体地址
}Person;
```

**EOF**