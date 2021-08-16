# Java 异常

## 异常类层次

![image-20210304100649589](..\pic\image-20210304100649589.png)

异常主要分为两类

- Exception：程序可以处理的异常
- Error：属于程序无法处理的错误，不能通过 catch 捕获。如 OutOfMemoryError、StackOverFlowError 等1

## Exception

Exception 又分为两类

- 受查异常：指在编译时如果没有 catch/throw 处理的话就无法通过编译。除了 `RuntimeException` 及其子类外，其他的都属于受查异常。
- 非受查异常：编译过程中即使不处理也可以通过。常见如空指针、除零、数组越界异常等。