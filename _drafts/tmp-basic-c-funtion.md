
> 这里记录下阅读源代码期间，遇到的C函数。

# 字符串

# 文件

- open

{% highlight cpp %}
int open(const char *pathname, int flags)
{% endhighlight %}

flags常用参数：
O_RDONLY: 只读的方式打开；
O_WRONLY: 只写的方式打开；
O_RDWR: 读写的方式打开；
> 上面三种只可选其一；
O_CRET: 不存在进行创建；

返回值：
0打开成功，返回一个文件描述符；-1表示失败。

- write

{% highlight cpp %}
ssize_t	 write(int fd, const void * buf, size_t nbyte)
{% endhighlight %}

将buf内存的nbyte字节写入fd所打开的文件中。

返回值：
实际写入文件的字节数；写入发生错误返回-1；

- read

{% highlight cpp %}
ssize_t	 read(int fd, void * buf, size_t nbyte) 
{% endhighlight %}

将fd所打开的文件中nbyte个字节写入buf所指的内存中。

返回值：
实际读取文件的字节数；读取发生错误返回-1；

---

使用write和read，需要将返回值和传入的nbyte进行比较，查看是否符合。

每一个打开的文件有一个读写位置，open后读写位置指向文件开始处。当调用write和read后，读写位置会进行移动。

- lseek

{% highlight cpp %}
off_t  lseek(int fildes, off_t offset, int whence);
{% endhighlight %}

控制文件的读写位置。fildes就是open返回的文件描述符。

whence参数取值如下：
SEEK_SET : 此时offset为新的读写位置；
SEEK_CUR : 将当前的读写位置往后增加offset；
SEEK_END : 将读写位置指向文件末尾，在增加offset，此时offset可能为负数；

{% highlight cpp %}
lseek(fd, 0, SEEK_SET); // 读写位置移动到文件头
lseek(fd, 0, SEEK_END); // 读写位置移动到文件尾
lseek(fd, 0, SEEK_CUR); // 获取当前读写位置
{% endhighlight %}

返回值：
当前读写位置，也就是距离文件头多少个字节；发生错误返回-1；

- realpath

{% highlight cpp %}
char *realpath(const char *path, char *resolved_path)
{% endhighlight %}

将path相对路径转成绝对路径后存到resolved_path指向的字符数组中。

返回值:
成功返回resolved_path;  失败返回NULL；

# 字符串

- strrchr

{% highlight cpp %}
char	*strrchr(const char *__s, int __c);
{% endhighlight %}

查找字符c在字符串str末次出现的位置，也就是从str末尾开始查找，找到返回对应的位置即可。未找到则返回NULL。

- strcpy, strncpy, strlcpy

{% highlight cpp %}
char src[] = "longest src";
char dst[5];
strcpy(dst, src); // buffer overflow
{% endhighlight %}

strcpy实现为了效率没有进行检查dst是否有足够的空间，所以src过长则导致了缓冲区溢出。

--- 

{% highlight cpp %}
char src[] = "longestsrc";
char dst1[5];
strncpy(dst1, src, sizeof(dst1)); // 此时只拷贝num个字符，不会添加 '\0', 输出乱码；
{% endhighlight %}

strncpy 增加了一个size参数，指定拷贝的字节数，但是没有处理`\0`，需要手动处理，如下所示:

{% highlight cpp %}
char src[] = "longestsrc";
char dst2[5];
strncpy(dst2, src, sizeof(dst2)-1);
dst2[sizeof(dst2)-1] = '\0'; // dst2 = "long"
{% endhighlight %}

---

{% highlight cpp %}
char src[] = "longestsrc";
char dst2[5];
auto rs = strlcpy(dst2, src, sizeof(dst2)); // dst2 = "long"
if (rs >= sizeof(dst2)) {
	printf("truncation occurred.\n");
}
{% endhighlight %}

strlcpy 自动处理`\0`。strlcpy 返回的值是strlen(src);

- strcat, strlcat

{% highlight cpp %}
char src[] = "botwin";
char dst[] = "nancy";

strcat(dst, src);
{% endhighlight %}

和strcpy类似，strcat没有检查dst的大小。

---

{% highlight cpp %}
char src[] = "botwin";
char dst[20] = "nancy";

strlcat(dst, src, sizeof(dst));
{% endhighlight %}

strlcat 通过第三个参数进行了检查。

# 内存

- memcpy, memmove

memcpy 和 memmove都是从src开始将n个字节个字节移动到dest开始的位置。

区别在于，memmove可以处理src和dest重叠的问题。所以速度也会有一定降低。

```cpp
char s[] = "1234567890";
char* src = s;
char* dest = s+2;

memcpy(dest, src, 5); //  "12121890"
memmove(dest, src, 5); // "12345890"
```

```cpp
void* my_memcpy(void* dest, const void* src, size_t n)
{
    char*      d = (char*) dest;
    const char*  s = (const char*) src;
    while (n--)
       *d++ = *s++;
    return dest;
}

void* my_memmove(void* dest, const void* src, size_t n)
{
    char*     d  = (char*) dest;
    const char*  s = (const char*) src;

    if (s>d)
    {
         // start at beginning of s
         while (n--)
            *d++ = *s++;
    }
    else if (s<d)
    {
        // start at end of s
        d = d+n-1;
        s = s+n-1;

        while (n--)
           *d-- = *s--;
    }
    return dest;
}
```

# References

[https://www.cnblogs.com/edwardcmh/archive/2013/06/04/3117628.html](https://www.cnblogs.com/edwardcmh/archive/2013/06/04/3117628.html)
