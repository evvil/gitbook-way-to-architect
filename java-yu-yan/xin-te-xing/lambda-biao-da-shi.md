# lambda表达式

1、用lambda表达式实现Runnable：`() -> {}`代码块替代了整个匿名类

```java
// Java 8之前：
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("Before Java8, too much code for too little to do");
    }
}).start();

//Java 8方式：
new Thread(() -> System.out.println("In Java8, Lambda expression rocks !!")).start();
```

 你可以使用lambda写出如下代码：

```java
(params) -> expression
(params) -> statement
(params) -> { statements }
```

 2、使用Java 8 lambda表达式进行事件处理

```java
// Java 8之前：
JButton show = new JButton("Show");
show.addActionListener(new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println("Event handling without lambda expression is boring");
    }
});

// Java 8方式：
show.addActionListener((e) -> {
    System.out.println("Light, Camera, Action !! Lambda expressions Rocks");
});
```

 3、使用lambda表达式对列表进行迭代

```java
// Java 8之前：
List features = Arrays.asList("Lambdas", "Default Method", "Stream API", "Date and Time API");
for (String feature : features) {
    System.out.println(feature);
}

// Java 8之后：
List features = Arrays.asList("Lambdas", "Default Method", "Stream API", "Date and Time API");
features.forEach(n -> System.out.println(n));

//或者使用这种方式，即方法引用由::双冒号操作符标示，
features.forEach(System.out::println);
```

 

来源：[Java8 lambda表达式10个示例](http://www.importnew.com/16436.html)





