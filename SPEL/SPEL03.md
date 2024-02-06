## rootObject

在表达式中直接写name和getName()，这时候Expression是无法解析的，因为其不知道name和getName()对应什么意思

```java
@Test
public void test06() {
	 ExpressionParser parser = new SpelExpressionParser();
	 parser.parseExpression("name").getValue();
	 parser.parseExpression("getName()").getValue();
}
```

当表达式是基于某一个对象时，我们可以把对应的对象作为一个rootObject传递给对应的Experssion进行取值

```java
@Test
public void test07() {
    Object user = new Object() {
        public String getName() {
        	return "abc";
        }
    };
    ExpressionParser parser = new SpelExpressionParser();
    Assert.assertTrue(parser.parseExpression("name").getValue(user, String.class).equals("abc"));
    Assert.assertTrue(parser.parseExpression("getName()").getValue(user, String.class).equals("abc"));
}
```

<a name="ILGZV"></a>

## 设置上下文

通过指定EvaluationContext我们可以让name和getName()变得有意义，指定了EvaluationContext之后，Expression将根据对应的EvaluationContext来进行解析

```java
@Test
public void test06() {
    Object user = new Object() {
    public String getName() {
        	return "abc";
        }
    };
    EvaluationContext context = new StandardEvaluationContext(user);
    ExpressionParser parser = new SpelExpressionParser();
    Assert.assertTrue(parser.parseExpression("name").getValue(context, String.class).equals("abc"));
    Assert.assertTrue(parser.parseExpression("getName()").getValue(context, String.class).equals("abc"));
}
```

<a name="NlhDs"></a>

## 设置变量

```java
@Test
public void test14() {
    Object user = new Object() {
        public String getName() {
        	return "abc";
        }
    };
    EvaluationContext context = new StandardEvaluationContext();
    //1、设置变量
    context.setVariable("user", user);
    ExpressionParser parser = new SpelExpressionParser();
    //2、表达式中以#varName的形式使用变量
    Expression expression = parser.parseExpression("#user.name");
    //3、在获取表达式对应的值时传入包含对应变量定义的EvaluationContext
    String userName = expression.getValue(context, String.class);
    //表达式中使用变量，并在获取值时传递包含对应变量定义的EvaluationContext。
    Assert.assertTrue(userName.equals("abc"));
}
```

<a name="Cjn7t"></a>

## #root

#root在表达式中永远都指向对应EvaluationContext的rootObject对象

```java
@Test
public void test14_1() {
    Object user = new Object() {
        public String getName() {
        	return "abc";
        }
    };
    EvaluationContext context = new StandardEvaluationContext(user);
    ExpressionParser parser = new SpelExpressionParser();
 	Assert.assertTrue(parser.parseExpression("#root.name").getValue(context).equals("abc"));
}
```

<a name="zk8lM"></a>

## #this

#this永远指向当前对象，其通常用于集合类型，表示集合中的一个元素

```java
@Test
public void test14_2() {
    ExpressionParser parser = new SpelExpressionParser();
    List<Integer> intList = (List<Integer>)parser.parseExpression("{1,2,3,4,5,6}").getValue();
    EvaluationContext context = new StandardEvaluationContext(intList);
    //从List中选出为奇数的元素作为一个List进行返回，1、3、5。
    List<Integer> oddList = (List<Integer>)parser.parseExpression("#root.?[#this%2==1]").getValue(context);
    for (Integer odd : oddList) {
    	Assert.assertTrue(odd%2 == 1);
    }
}
```

<a name="uYrbR"></a>

## 注册方法

StandardEvaluationContext允许我们在其中注册方法，然后在表达式中使用对应的方法，注册的方法必须是一个static类型的公有方法。注册方法是通过StandardEvaluationContext的registerFunction(funName,method)方法进行。参数1表示需要在表达式中使用的方法名称，参数2表示需要注册的java.lang.reflect.Method。在表达式中可以使用类似与#funName(params...)的形式来使用对应的方法

```java
static class MathUtils {
    public static int plusTen(int i) {
    	return i+10;
    }
}
 
@Test
public void test15() throws NoSuchMethodException, SecurityException {
    ExpressionParser parser = new SpelExpressionParser();
    //1、获取需要设置的java.lang.reflect.Method，需是static类型
    Method plusTen = MathUtils.class.getDeclaredMethod("plusTen", int.class);
    StandardEvaluationContext context = new StandardEvaluationContext();
    //2、注册方法到StandardEvaluationContext，第一个参数对应表达式中需要使用的方法名
    context.registerFunction("plusTen", plusTen);
    //3、表达式中使用注册的方法
    Expression expression = parser.parseExpression("#plusTen(10)");
    //4、传递包含对应方法注册的StandardEvaluationContext给Expression以获取对应的值
    int result = expression.getValue(context, int.class);
    Assert.assertTrue(result == 20);
}
```

<a name="HiJU9"></a>

## 赋值

SPEL支持给表达式赋值，其是通过Expression的setValue()方法进行的，在赋值时需要指定rootObject或对应的EvaluationContext

```java
@Test
public void test() {
    ExpressionParser parser = new SpelExpressionParser();
    Date d = new Date();
    
    Expression expression = parser.parseExpression("date");
    //设日期为1号 此处为date，其原理是通过调用Date类的setDate方法，必须指定是set开头的方法
    expression.setValue(d, 1);
    Object value = expression.getValue(d);
    System.out.println(value);

    //其原理是通过调用Date类的setYear方法
    expression = parser.parseExpression("year");
    expression.setValue(d, 2023);
    value = expression.getValue(d);
    System.out.println(value);
}
```

对于List而言，在进行赋值时是通过元素的索引进行的，且对应的索引是必须存在的

```java
@Test
public void test09() {
    ExpressionParser parser = new SpelExpressionParser();
    List<Integer> list = new ArrayList<Integer>(1);
    list.add(0);//添加一个元素0
    EvaluationContext context = new StandardEvaluationContext();
    //添加变量以方便表达式访问
    context.setVariable("list", list);
    //设置第一个元素的值为1
    Expression expression = parser.parseExpression("#list[0]");
    expression.setValue(context, 1);
    int first = (Integer) expression.getValue(context);
    System.out.println(first);
}
```

对于Map的赋值是通过key进行的，对应的key在Map中可以先不存在

```java
@Test
public void test10() {
    ExpressionParser parser = new SpelExpressionParser();
    Map<String, Integer> map = new HashMap<>();
    EvaluationContext context = new StandardEvaluationContext();
    //添加变量以方便表达式访问
    context.setVariable("map", map);
    //设置第一个元素的值为1
    Expression expression = parser.parseExpression("#map['key1']");
    expression.setValue(context, 1);
    int first = (Integer) expression.getValue(context);
    System.out.println(first);
}
```

<a name="aAoEG"></a>