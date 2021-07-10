# Java 8 新特性

## Lambda 表达式

### 匿名内部类

Java 8 以前，实现程序的可扩展性往往使用匿名内部类传递用于扩展的代码，例如实现一个能从 List 中筛选出指定对象的函数 filter

- 定义 filter 函数

```java
List<T> filter(List<T> list, FilterProcessor filterProcessor){
    List<T> result = new ArrayList<T>();
    for(T t : list){
        if(filterProcessor.process(t))
            result.add(t);
    }
    return list;
}
```

- 定义 FilterProcessor 接口

```java
interface FilterProcessor<T>{
    boolean process(T);
}
```

- 匿名内部类实现具体逻辑，如筛选出 30 岁以上的 Person

```java
List<Person> result = filter(list, new FilterProcessor<Person>(){
    boolean process(Person person){
        if(person.getAge()>=30)
            return true;
        return false;
    }
});
```

### 策略模式

**策略模式**：如果某类问题大部分处理过程是一致的，只有核心步骤存在差异，就可以抽象出一个接口，不同的代码放在接口的实现类中，要使用时将该实现类的对象传递给该函数即可。

### Lambda

使用内部匿名类实现策略模式代码比较冗余不易阅读，Lambda 表达式就是为了简化实现策略模式，本质是将一个函数的代码作为一个参数或变量进行传递，即**函数式编程**。

- 定义函数式接口，只能有一个抽象函数

```java
@FunctionalInterface
interface FilterProcessor<T>{
    boolean process(T t);
}
```

- 实现筛选函数

```java
List<T> filter(List<T> list, FilterProcessor filterProcessor){
    List<T> result = new ArrayList<T>();
    for(T t : list){
        if(filterProcessor.process(t))
            result.add(t);
    }
    return list;
}
```

- 传递 Lambda 表达式

```java
List<Person> result = filter(list, (Person person)->person.getAge()>=30);
```

## Stream API

