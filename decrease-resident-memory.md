## 问题描述
&emsp;&emsp;新闻推荐`server`内存占用率过高，每个新闻`item`占用内存54K，为了避免造成过多内存碎片和提升性能，使用了内存池。该内存池结构：一个链表连接每个`block`，每个`block`初始为1000个`item`，即每个`block`占用54M内存。然后从数据库load数据，大概生成60000个`item`，也就是大约3G的内存，然后两个内存池，用来做reload数据并指针置换，这样占用内存达6G。内存池使用`calloc`申请内存，底层调用`mmap`，费用有些高，所以一次尽量多申请些内存。为什么用`calloc`，而不用`malloc + memset`，因为后者必然会占用6G内存，`calloc`首先避免了重复的`memset`调用，其次也优化了内存申请逻辑，即申请3G内存，其实并没有真实的占有3G内存，只是拿到了系统发的保证书，意思就是要的时候再给你。其实就是，写内存的时候才会触发`PAGE FAULT`，进而触发`PAGE SWAP`，申请真实的RAM。那么问题出现了，内存池使用的是`calloc`，理论上根本不会占用6G内存，为什么呢？(以上所说内存都是指`top`显示的`RES`内存，即resident memory，不是virtual memory，不知道两者区别可以先去做做功课)

## 看以下例子并回答我的问题

### struct article_item
```
#define MAX_CONTENT_LEN         51200

#pragma pack(1) /* 不推荐1字节对齐，这样会损耗性能，正确写法应该是把按宽度降序排列 */
typedef struct article_item{
    uint32_t    create_time;
    uint32_t    modify_time;
    uint32_t    public_time;
    uint64_t    source_id;
    uint64_t    item_id;
    uint64_t    group_id;
    uint16_t    type;
    uint16_t    duration;
    char        video[512];
    char        original_video[512];
    char        original_thumbnails[512];
    char        category[32];
    char        data_type[32];
    char        url_source[32];
    char        source[32];
    char        title[256];
    char        content[MAX_CONTENT_LEN];
    char        url[512];
    char        thumbnails[512];
    char        url_target[512];
    char        extra[1024];
} article_item;
#pragma pack()
```
笔者推荐结构体写法：
```
#define MAX_CONTENT_LEN         51200

typedef struct article_item{
    uint64_t    source_id;
    uint64_t    item_id;
    uint64_t    group_id;
    uint32_t    create_time;
    uint32_t    modify_time;
    uint32_t    public_time;
    uint16_t    type;
    uint16_t    duration;
    char        video[512];
    char        original_video[512];
    char        original_thumbnails[512];
    char        category[32];
    char        data_type[32];
    char        url_source[32];
    char        source[32];
    char        title[256];
    char        content[MAX_CONTENT_LEN];
    char        url[512];
    char        thumbnails[512];
    char        url_target[512];
    char        extra[1024];
} article_item;
```

### Version A
```
int main(int argc, char **argv) {
    cout<<getpid()<<endl;
    for (int i = 0; i < 60; ++i) {
        article_item *item = (article_item *)calloc(1000, sizeof(article_item));
        for (int j = 0; j < 1000; ++j) {
            //memset(&item[j], 0, sizeof(article_item));
            memset(&item[j], 0, 100);
        }
    }
    sleep(86400);

    return 0;
}
```

### Version B
```
int main(int argc, char **argv) {
    cout<<getpid()<<endl;
    for (int i = 0; i < 60; ++i) {
        article_item *item = (article_item *)calloc(1000, sizeof(article_item));
        for (int j = 0; j < 1000; ++j) {
            memset(&item[j], 0, sizeof(article_item));
            //memset(&item[j], 0, 100);
        }
    }
    sleep(86400);

    return 0;
}
```

### 笔者的问题
 - `A`和`B`哪个`RES`占用率更高？
 - 如果答不上来可以亲自试一下
 - 尝试思考为什么

## 项目里的问题
&emsp;&emsp;以上例子是笔者对项目里问题的一个简化，原情景是这样的：`Mysql`读取数据的时候，会把数据库里的新闻`content`截断，即`strncpy(article_item.content, ROW[i], MAX_CONTENT_LEN)`，翻看glibc `strncpy`源码，如下：
```
char *
STRNCPY (char *s1, const char *s2, size_t n)
{
  size_t size = __strnlen (s2, n);
  if (size != n)
    memset (s1 + size, '\0', n - size);
  return memcpy (s1, s2, size);
}
libc_hidden_builtin_def (strncpy)
```
可知，`strncpy`并不是末尾补一个`0`，而是末尾全部补`0`，这样就直接造成写整块内存，所以最终内存占用达到了6G，解决方案是自己实现一个`strncpy`函数，如下：
```
static char* safe_strncpy(char *dest, const char *src, size_t n){
    if(NULL == dest) return NULL;
    if(NULL == src) return dest;
    if (strlen(src) >= n) {
        memmove(dest, src, n);
    } else {
        memmove(dest, src, strlen(src));
        dest[strlen(src)] = 0;
    }

    return dest;
}
```

## 结语
好想进BAT……
