C++中class的默认权限是private，struct的默认权限是public除此之外没有别的区别（无论是struct还是class非static的成员变量定义的时候都不能初始化，和java不同）

C语言中struct只能定义成员变量，不能定义函数，而且定义的时候不能初始化，但是可以通过函数指针间接在struct中定义函数

C++中可以定义成员变量和函数，但是定义的时候不能初始化。



```c
int test(){
    cout<<"test()函数运行"<<endl;
    return 0;
}

struct Student{
    int age;
    //C语言中struct中不能定义函数，C++可以，可以通过函数指针让C语言中的结构体使用函数
    int (* run)(void);

};

struct Student stu;
stu.run = test;
stu.run();
```

