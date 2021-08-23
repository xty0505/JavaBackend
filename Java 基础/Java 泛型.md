## Java 泛型的实现方法

Java 的泛型是伪泛型，在编译期间所有的泛型信息都会被擦除。编译器在生成泛型对象的字节码时不会包含泛型中的类型信息，这个过程称为**类型擦除**

`List<Object>` 和 `List<Integer>` 等在编译后都会变成 `List`，JVM 只能看到 List 看不到泛型附加的类型信息。Java 编译器会尽可能在编译时发现出错的地方，但是仍然无法避免在运行时出现的类型转换异常。

### 原始类型相等

```Java
public class Test {

    public static void main(String[] args) {

        ArrayList<String> list1 = new ArrayList<String>();
        list1.add("abc");

        ArrayList<Integer> list2 = new ArrayList<Integer>();
        list2.add(123);

        System.out.println(list1.getClass() == list2.getClass());
    }

}
```

上述程序输出应该为true, 两个 list.getClass() 相等。

### 通过反射增加其他类型元素

```Java
public class Test {

    public static void main(String[] args) throws Exception {

        ArrayList<Integer> list = new ArrayList<Integer>();

        list.add(1);  //这样调用 add 方法只能存储整形，因为泛型类型的实例为 Integer

        list.getClass().getMethod("add", Object.class).invoke(list, "asd");

        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }

}
```

直接调用 add 只能存储整型。反射调用 add 可以存储字符串，说明 list 的 Integer 泛型被擦除了。



## 类型擦除后保留的原始类型

**原始类型**就是擦除类型后真正在字节码中的类型。泛型的类型变量会被其**限定类型**（？extends A，则 A 为限定类型）替换，若无限定类型则为 Object

```Java
public class Pair<T> {  
    private T value;  
    public T getValue() {  
        return value;  
    }  
    public void setValue(T  value) {  
        this.value = value;  
    }  
}  
```

Pair 的原始类型为

```Java
public class Pair {  
    private Object value;  
    public Object getValue() {  
        return value;  
    }  
    public void setValue(Object  value) {  
        this.value = value;  
    }  
}
```

如果 Pair 声明为

```Java
public class Pair<T extends Comparable> {}
```

则原始类型为 Comparable



待用泛型方法时可以指定泛型也可以不指定泛型。

- 不指定泛型的情况下，泛型变量的类型是参数类型的同一父类的最小级，Object 最后兜底
- 指定泛型的情况下，该方法的参数类型必须是该泛型的实例的类型或者其子类

```Java
public class Test {  
    public static void main(String[] args) {  

        /**不指定泛型的时候*/  
        int i = Test.add(1, 2); //这两个参数都是Integer，所以T为Integer类型  
        Number f = Test.add(1, 1.2); //这两个参数一个是Integer，一个是Float，所以取同一父类的最小级，为Number  
        Object o = Test.add(1, "asd"); //这两个参数一个是Integer，一个是Float，所以取同一父类的最小级，为Object  

        /**指定泛型的时候*/  
        int a = Test.<Integer>add(1, 2); //指定了Integer，所以只能为Integer类型或者其子类  
        int b = Test.<Integer>add(1, 2.2); //编译错误，指定了Integer，不能为Float  
        Number c = Test.<Number>add(1, 2.2); //指定为Number，所以可以为Integer和Float  
    }  

    //这是一个简单的泛型方法  
    public static <T> T add(T x,T y){  
        return y;  
    }  
}
```

### 

## 类型擦除带来的问题及解决方法

### 引用先检查再编译

```java
public class Test {  

    public static void main(String[] args) {  

        ArrayList<String> list1 = new ArrayList();  
        list1.add("1"); //编译通过  
        list1.add(1); //编译错误  
        String str1 = list1.get(0); //返回类型就是String  

        ArrayList list2 = new ArrayList<String>();  
        list2.add("1"); //编译通过  
        list2.add(1); //编译通过  
        Object object = list2.get(0); //返回类型就是Object  

        new ArrayList<String>().add("11"); //编译通过  
        new ArrayList<String>().add(22); //编译错误  

        String str2 = new ArrayList<String>().get(0); //返回类型就是String  
    }  

}  
```

类型检查针对于**引用**

### 不允许泛型类型继承

```Java
ArrayList<String> list1 = new ArrayList<Object>(); //编译错误  
ArrayList<Object> list2 = new ArrayList<String>(); //编译错误
```

- 第一种情况下，为了避免 Object 转换为 String 类型报错 ClassCastException，因此编译器不允许这样的引用赋值
- 第二种情况，虽然 String 能够转换为 Object，但是这违背了泛型设计的初衷，本来就是为了解决类型转换，但是现在却又要 将 String 强转换为 Object



#### 自动类型转换

泛型类型既然都被擦除，转换为原始类型，为什么在获取是不用强制转换

Java 编译器会将在返回时自动强转换，由于之前检查的存在，不会报错。



#### 类型擦除与多态的冲突







