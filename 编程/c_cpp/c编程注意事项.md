## c编程注意事项

[www.zhaoch.top](http://www.zhaoch.top) > [编程](http://www.zhaoch.top/编程) > [c_cpp](http://www.zhaoch.top/编程/c_cpp)

## 列了一下c编程的注意事项，有些时自己总结的，有些是一些公司的规范

### 空指针检查
主要是对外提供的程序入口，很多开源库本身内部函数也没有处处检查指针，能在程序中用合理的架构保证更好。

### 释放指针设置为NULL
防止野指针

### 循环体内不修改循环变量
容易造成死循序，高手请忽略，高可靠场景中最好做到

### 循环设置循环上限
对某些异常情况会造成死循环，或者长时间没有相应。例如：

    while(n) {
        // do sth
        n = n->next
    }

可以改成

    for (int i = 0; i < LOOP_LIMIT && n != NULL; i++, n= n->next) {
         // do sth
    }

### 循环变量的数据类型有效范围不小于循环上限的表示范围

错误示例：

    uint32 max = 100000000000;
    for(uint16 i =0; i < max; i++) {
        // do sth
    }

### 内存访问做检查

### 数组访问做长度检查

### 加减乘防止正溢负溢

### 除法,求余防止除零操作

### 圈复杂度不超过10

### 代码行数不超过50

### 变量必须被正确的初始化

### warning清零

### 禁止递归调用(消息递归)
容易死循环，容易栈溢出

### 不使用goto

### 所有代码块使用括号
保持代码风格统一，防止不必要的猜测，防止多行宏引起的错误

可能并不是你想要的结果

    if (x == 1)
        if (y == 2)
            printf("11111")
    else
        printf("22222")

多行宏引起的错误

    #define haha(x) x += 1; \
                    x *= x;

    if (x < 0)
        haha(x);


### 不建议使用可变参数
缺乏类型校验，造成随机很难定位的问题

### 用const和inline代替宏定义
宏定义缺乏类型校验


以上仅是建议，不强制

## The End

+ My [github location](https://github.com/GhostZCH/)
+ View Source of this website [GhostZch.github.io](https://github.com/GhostZCH/GhostZch.github.io/)
+ Commit [issues](https://github.com/GhostZCH/GhostZch.github.io/issues) to discuss with me and others
