---
layout:     post
title:      "转载-03丨《Effective Java 第三版》目录汇总"
date:       2020-04-12 20:49:01
author:     "jiefang"
header-style: text
tags:
    - 转载
---
# 《Effective Java 第三版》目录汇总
## 第一章 简介
忽略
## 第二章 创建和销毁对象
[1.考虑使用静态工厂方法替代构造方法](https://www.jianshu.com/p/bcbf22d00823 "1. 考虑使用静态工厂方法替代构造方法") 

[2.当构造方法参数过多时使用builder模式](https://www.jianshu.com/p/89f3cfa975c5 "2. 当构造方法参数过多时使用builder模式") 

[3.使用私有构造方法或枚类实现Singleton属性](https://www.jianshu.com/p/1bc9c189349f "3. 使用私有构造方法或枚类实现Singleton属性") 

[4.使用私有构造方法执行非实例化](https://www.jianshu.com/p/240038800408 "4. 使用私有构造方法执行非实例化") 

[5.使用依赖注入取代硬连接资源](https://www.jianshu.com/p/528fb782cf7a "5. 使用依赖注入取代硬连接资源") 

[6.避免创建不必要的对象](https://www.jianshu.com/p/6c342d119fac "6. 避免创建不必要的对象") 

[7.消除过期的对象引用](https://www.jianshu.com/p/970d3bd09f2d "7. 消除过期的对象引用") 

[8.避免使用Finalizer和Cleaner机制](https://www.jianshu.com/p/45b4df2bd7aa "8. 避免使用Finalizer和Cleaner机制") 

[9.使用try-with-resources语句替代try-finally语句](https://www.jianshu.com/p/a9bb429bc44b "9. 使用try-with-resources语句替代try-finally语句") 


## 第三章 所有对象的通用方法
[10.重写equals方法时遵守通用约定](https://www.jianshu.com/p/787775ea4899 "10. 重写equals方法时遵守通用约定") 

[11.重写equals方法时同时也要重写hashcode方法](https://www.jianshu.com/p/cbb4e362915e "11. 重写equals方法时同时也要重写hashcode方法") 

[12.始终重写toString方法](https://www.jianshu.com/p/fbb552766c86 "12. 始终重写 toString 方法") 

[13.谨慎地重写clone方法](https://www.jianshu.com/p/e84d1fae0809 "13. 谨慎地重写 clone 方法") 

[14.考虑实现Comparable接口](https://www.jianshu.com/p/42080c010ec7 "14.考虑实现Comparable接口") 

## 第四章 类和接口
[15.使类和成员的可访问性最小化](https://www.jianshu.com/p/a4eeb84d9f6e "15. 使类和成员的可访问性最小化") 

[16.在公共类中使用访问方法而不是公共属性](https://www.jianshu.com/p/8eac2bebca2b "16. 在公共类中使用访问方法而不是公共属性") 

[17.最小化可变性](https://www.jianshu.com/p/bf870827fa1f "17. 最小化可变性") 

[18.组合优于继承](https://www.jianshu.com/p/e00c70fed3ec "18. 组合优于继承") 

[19.如果使用继承则设计，并文档说明，否则不该使用](https://www.jianshu.com/p/e7d057b488c2 "19. 如果使用继承则设计，并文档说明，否则不该使用") 

[20.接口优于抽象类](https://www.jianshu.com/p/5e7631cca2a6 "20. 接口优于抽象类") 

[21.为后代设计接口](https://www.jianshu.com/p/400380392a93 "21. 为后代设计接口") 

[22.接口仅用来定义类型](https://www.jianshu.com/p/b98858db1ad5 "22. 接口仅用来定义类型") 

[23.优先使用类层次而不是标签类](https://www.jianshu.com/p/e6d5b9213c22 "23. 优先使用类层次而不是标签类") 

[24.优先考虑静态成员类](https://www.jianshu.com/p/070dc9e58ca5 "24. 优先考虑静态成员类") 

[25.将源文件限制为单个顶级类](https://www.jianshu.com/p/2f52e8765530 "25. 将源文件限制为单个顶级类") 


## 第五章 泛型
[26.不要使用原始类型](https://www.jianshu.com/p/31c6f8d944e1 "26. 不要使用原始类型") 

[27.消除非检查警告](https://www.jianshu.com/p/4c06fa21828c "27. 消除非检查警告") 

[28.列表优于数组](https://www.jianshu.com/p/27a2a1c6a0a3 "28. 列表优于数组") 

[29.优先考虑泛型](https://www.jianshu.com/p/792056198188 "29. 优先考虑泛型") 

[30.优先使用泛型方法](https://www.jianshu.com/p/0bfdcc95aeea "30. 优先使用泛型方法") 

[31.使用限定通配符来增加API的灵活性](https://www.jianshu.com/p/9692c8228910 "31. 使用限定通配符来增加API的灵活性") 

[32.合理地结合泛型和可变参数](https://www.jianshu.com/p/fb61db9bd168 "32. 合理地结合泛型和可变参数") 

[33.优先考虑类型安全的异构容器](https://www.jianshu.com/p/8bc1615c7c01 "33. 优先考虑类型安全的异构容器") 


## 第六章 枚举和注解
[34.使用枚举类型替代整型常量](https://www.jianshu.com/p/9fb00005e2fd "34. 使用枚举类型替代整型常量") 

[35.使用实例属性替代序数](https://www.jianshu.com/p/fa7f16da87f1 "35. 使用实例属性替代序数") 

[36.使用EnumSet替代位属性](https://www.jianshu.com/p/60a01a86d047 "36. 使用EnumSet替代位属性") 

[37.使用EnumMap替代序数索引](https://www.jianshu.com/p/5e8ad5ccde81 "37. 使用EnumMap替代序数索引") 

[38.使用接口模拟可扩展的枚举](https://www.jianshu.com/p/05b80fffe174 "38. 使用接口模拟可扩展的枚举") 

[39.注解优于命名模式](https://www.jianshu.com/p/e96552fbe3e3 "39. 注解优于命名模式") 

[40.始终使用Override注解](https://www.jianshu.com/p/f3ec41f4765b "40. 始终使用Override注解") 

[41.使用标记接口定义类型](https://www.jianshu.com/p/fcf4f54e8151 "41. 使用标记接口定义类型") 


## 第七章  Lambda表达式和Stream流
[42.lambda表达式优于匿名类](https://www.jianshu.com/p/1895987b0533 "42. lambda表达式优于匿名类") 

[43.方法引用优于lambda表达式](https://www.jianshu.com/p/c96b76a9ed88 "43. 方法引用优于lambda表达式") 

[44.优先使用标准的函数式接口](https://www.jianshu.com/p/56e1ef91e34e "44. 优先使用标准的函数式接口") 

[45.明智审慎地使用Stream](https://www.jianshu.com/p/2fb892bba004 "45. 明智审慎地使用Stream") 

[46.在流中优先使用无副作用的函数](https://www.jianshu.com/p/97e2b5b070d2 "46. 在流中优先使用无副作用的函数") 

[47.优先使用Collection而不是Stream来作为方法的返回类型](https://www.jianshu.com/p/9169f54fb703 "47. 优先使用Collection而不是Stream来作为方法的返回类型") 

[48.谨慎使用流并行](https://www.jianshu.com/p/dc6a993b8ab7 "48. 谨慎使用流并行") 

## 第八章 方法
[49.检查参数有效性](https://www.jianshu.com/p/01621268c630 "49. 检查参数有效性") 

[50.必要时进行防御性拷贝](https://www.jianshu.com/p/0da637139be4 "50. 必要时进行防御性拷贝") 

[51.仔细设计方法签名](https://www.jianshu.com/p/4b9a1e05e783 "51. 仔细设计方法签名") 

[52.明智而审慎地使用重载](https://www.jianshu.com/p/78e583144528 "52. 明智而审慎地使用重载") 

[53.明智而审慎地使用可变参数](https://www.jianshu.com/p/001a9e82888f "53. 明智而审慎地使用可变参数") 

[54.返回空的数组或集合不要返回null](https://www.jianshu.com/p/c6f33e26dcc8 "54. 返回空的数组或集合不要返回null") 

[55.明智而审慎地返回Optional](https://www.jianshu.com/p/e54d5a1a969b "55. 明智而审慎地返回Optional") 

[56.为所有已公开的API元素编写文档注释](https://www.jianshu.com/p/c3505882a8a6 "56. 为所有已公开的API元素编写文档注释") 

## 第九章 通用编程
[57.最小化局部变量的作用域](https://www.jianshu.com/p/d66ee8476829 "57. 最小化局部变量的作用域") 

[58.for-each循环优于传统for循环](https://www.jianshu.com/p/0345a628506d "58. for-each循环优于传统for循环") 

[59.熟悉并使用Java类库](https://www.jianshu.com/p/e024fc77a77b "59. 熟悉并使用Java类库") 

[60.需要精确的结果时避免使用float和double类型](https://www.jianshu.com/p/e9be8fa86ca4 "60. 需要精确的结果时避免使用float和double类型") 

[61.基本类型优于装箱的基本类型](https://www.jianshu.com/p/1f70b7722ee6 "61. 基本类型优于装箱的基本类型") 

[62.当有其他更合适的类型时就不用字符串](https://www.jianshu.com/p/a7d44a77ecda "62. 当有其他更合适的类型时就不用字符串") 

[63.注意字符串连接的性能](https://www.jianshu.com/p/29ce30d4d683 "63. 注意字符串连接的性能") 

[64.通过对象的接口引用对象](https://www.jianshu.com/p/0f7b6a2b21fe "64. 通过对象的接口引用对象") 

[65.接口优于反射](https://www.jianshu.com/p/264323730a65 "65. 接口优于反射") 

[66.明智谨慎地使用本地方法](https://www.jianshu.com/p/06412c130302 "66. 明智谨慎地使用本地方法") 

[67.明智谨慎地进行优化](https://www.jianshu.com/p/5662882fde8c "67. 明智谨慎地进行优化") 

[68.遵守普遍接受的命名约定](https://www.jianshu.com/p/ec511eb1989e "68. 遵守普遍接受的命名约定") 

## 第十章 异常
[69.仅在发生异常的条件下使用异常](https://www.jianshu.com/p/5aadfd504e41 "69. 仅在发生异常的条件下使用异常") 

[70.对可恢复条件使用已检查异常，对编程错误使用运行时异常](https://www.jianshu.com/p/49ea113bf450 "70. 对可恢复条件使用已检查异常，对编程错误使用运行时异常") 

[71.避免不必要地使用检查异常](https://www.jianshu.com/p/c0df50c49a81 "71. 避免不必要地使用检查异常") 

[72.赞成使用标准异常](https://www.jianshu.com/p/bf724e43e627 "72. 赞成使用标准异常") 

[73.抛出合乎于抽象的异常](https://www.jianshu.com/p/f33afa9c801b "73. 抛出合乎于抽象的异常") 

[74.文档化每个方法抛出的所有异常](https://www.jianshu.com/p/0e7f9f0f0f23 "74. 文档化每个方法抛出的所有异常") 

[75.在详细信息中包含失败捕获信息](https://www.jianshu.com/p/5f20c36b4c81 "75. 在详细信息中包含失败捕获信息") 

[76.争取保持失败原子性](https://www.jianshu.com/p/fd8a0d7e356b "76. 争取保持失败原子性") 

[77.不要忽略异常](https://www.jianshu.com/p/8eb4f9b77dc8 "77. 不要忽略异常") 

## 第十一章 并发
[78.同步访问共享的可变数据](https://www.jianshu.com/p/228187c203fe "78. 同步访问共享的可变数据") 

[79.避免过度同步](https://www.jianshu.com/p/192a52543451 "79. 避免过度同步") 

[80.EXECUTORS,TASKS,STREAMS优于线程](https://www.jianshu.com/p/1d74de83aa7e "80. EXECUTORS, TASKS, STREAMS 优于线程") 

[81.优先使用并发实用程序替代wait和notify](https://www.jianshu.com/p/2ed76299bdba "81. 优先使用并发实用程序替代wait和notify") 

[82.线程安全文档化](https://www.jianshu.com/p/21c5c50a8c79 "82. 线程安全文档化") 

[83.明智谨慎地使用延迟初始化](https://www.jianshu.com/p/1c2457aa2722 "83. 明智谨慎地使用延迟初始化") 

[84.不要依赖线程调度器](https://www.jianshu.com/p/940c96e53e2b "84. 不要依赖线程调度器") 

## 第十二章 序列化
[85.其他替代方式优于Java本身序列化](https://www.jianshu.com/p/cb8484b36e3e "85. 其他替代方式优于Java本身序列化") 

[86.非常谨慎地实现SERIALIZABLE接口](https://www.jianshu.com/p/09bfb3b14037 "86. 非常谨慎地实现SERIALIZABLE接口") 

[87.考虑使用自定义序列化形式](https://www.jianshu.com/p/d122109dce62 "87. 考虑使用自定义序列化形式") 

[88.防御性地编写READOBJECT方法](https://www.jianshu.com/p/6699a8e47899 "88. 防御性地编写READOBJECT方法") 

[89.对于实例控制，枚举类型优于READRESOLVE](https://www.jianshu.com/p/6efdcfd34c35 "89. 对于实例控制，枚举类型优于READRESOLVE") 

[90.考虑序列化代理替代序列化实例](https://www.jianshu.com/p/925862e0a199 "90. 考虑序列化代理替代序列化实例")