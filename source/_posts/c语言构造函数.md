---
title: c语言构造函数
categories: c语言
tags:
  - c语言
---
说起这个属性，要从fio的vpp说起，引擎的注册函数fio_libaio_register与反注册函数fio_libaio_unregister都没有其他函数调用，而fio又没有以动态库的形式将这两个函数供别的地方使用,但是这两个函数有宏定义fio_init和fio_exit来修饰。这个两个个宏定义为：
```
#define fio_init __attribute__((constructor))  
#define fio_exit __attribute__((destructor))
```
以如下构造函数为例,这是vpp中的代码：

```
#define VLIB_DECLARE_INIT_FUNCTION(x, tag)                      \
vlib_init_function_t * _VLIB_INIT_FUNCTION_SYMBOL (x, tag) = x; \
static void __vlib_add_##tag##_function_##x (void)              \
    __attribute__((__constructor__)) ;                          \
static void __vlib_add_##tag##_function_##x (void)              \
{                                                               \
 vlib_main_t * vm = vlib_get_main();                            \
 static _vlib_init_function_list_elt_t _vlib_init_function;     \
 _vlib_init_function.next_init_function                         \
    = vm->tag##_function_registrations;                         \
  vm->tag##_function_registrations = &_vlib_init_function;      \
 _vlib_init_function.f = &x;                                    \
}
```
这个两个属性是gcc提供的属性，在dpdk中也有体现。若函数被设定为constructor属性，则该函数会在main（）函数执行之前被自动的执行。若函数被设定为destructor属性，则该函数会在main（）函数执行之后或者exit（）被调用后被自动的执行。通过如下测试代码，能更加清晰地认识到这两个属性的作用：
```
#include <stdio.h>  
#include <stdlib.h>  
void __attribute__((constructor)) con_func()  
{  
    printf("befor main: constructor is called..\n");  
}  
void __attribute__((destructor)) des_func()  
{  
    printf("after main: destructor is called..\n");  
}  
int main()  
{  
    printf("main func..\n");  
    return 0;  
}
```
结果：
```
befor main: constructor is called..  
main func..  
after main: destructor is called..
```
