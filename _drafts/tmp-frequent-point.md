# point

## import 和 include 区别

include所做的事情，就是将header文件里面的内容进行一次复制到source中，这样source文件中就可以找到对应方法。

如果间接的多次include了多个header文件，那么就会造成内容定义两次，造成了重复定义，一个解决方法就是header文件中使用条件编译。

```
#ifndef _HEADERNAME_H
#define _HEADERNAME_H

...

#endif
```

使用条件编译后，预处理器仍将整个头文件读入，即使这个头文件所有内容将被忽略。由于这种处理将托慢编译速度，所以如果可能，应该避免出现多重包含。

import 则可以避免这种重复导入的问题。

## extern

## throttle 和 debounce

截流，限制某个操作的频率。

防抖动，超过某个时间才去执行操作。

[ref](https://www.jianshu.com/p/e91775195608)
