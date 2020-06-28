
## gettimeofday

```cpp
int gettimeofday (struct timeval * tv, struct timezone * tz);
// gettimeofday()会把目前的时间有tv 所指的结构返回，当地时区的信息则放到tz 所指的结构中。
// 返回值：成功则返回0，失败返回－1，错误代码存于errno

// timeval 结构定义为：
struct timeval{
    long tv_sec;  //秒
    long tv_usec;  //微秒
};

// timezone 结构定义为：
struct timezone {
    int tz_minuteswest;  //和Greenwich 时间差了多少分钟
    int tz_dsttime;  //日光节约时间的状态
};
```

## backtrace

```cpp
#include <execinfo.h>

int backtrace(void **buffer, int size);
// 返回函数的调用栈，buffer存放各个函数的返回地址，size则指定了最大的调用层级；

char **backtrace_symbols(void *const *buffer, int size);
// 根据返回的backtrace返回的butter地址，来获取具体的函数名；

void backtrace_symbols_fd(void *const *buffer, int size, int fd);
// 和上一个函数类似，只是将具体的函数名放到一个指定的文件中；

// 注意，在编译的时候需要加上-rdynamic选项让链接器将所有符号添加到动态符号表中，这样才能将函数地址翻译成函数名。另外，这个选项不会处理static函数，所以，static函数的符号无法得到。
```

```cpp
#include <iostream>
#include <execinfo.h>

void show() {
    int MaxBacktraceLimit = 10;
    void **backTracePtr = (void **)malloc(sizeof(void*) * MaxBacktraceLimit);

    if (backTracePtr != NULL) {
        size_t nptrs = backtrace(backTracePtr, MaxBacktraceLimit);
        printf("backtrace() returned %zu addresses\n", nptrs);

        char **strings = backtrace_symbols(backTracePtr, nptrs);
        if (strings != NULL) {
            for (int j = 0; j < nptrs; j++)
                  printf("%s\n", strings[j]);

            free(strings);
        }
    }

    abort();
}


int main(int argc, const char * argv[]) {

    show();
    return 0;
}
```
