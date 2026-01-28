---
title: "Generics in Java"
date: 2024-07-26T15:32:53+08:00
draft: true
---

# Generics

Java 里的泛型（Generics）分别和 **class、interface** 在一起时组成了 Generic types，和**方法**在一起时组成了 Generic methods。

## Generics Types

定义一个 Generics class ，语法如下：

```java
class name <T1, T2, Tn> {
  // ...
}
```

其中，T1,T2,Tn 叫做 `Type parameters` 或者叫做 `Type variables`。有了泛型后，写类型无关的代码时便省去了重复代码、强转类型等这些 error prone 的代码。如下：

```java
public class DisruptorMessage<T> {
    private T message;

    public T getMessage() {
        return message;
    }

    public void setMessage(T message) {
        this.message = message;
    }

    public static void main(String[] args) {
        DisruptorMessage<String> msg1 = new DisruptorMessage<>();
        msg1.setMessage("Hello world");

        DisruptorMessage<Integer> msg2 = new DisruptorMessage<>();
        msg2.setMessage(2);
    }
}
```

上面示例里定义了一个泛型类 `DisruptorMessage<T>`，参数化了变量 message 的类型。

```java
public class DisruptorEventHandler {
    private DisruptorMessage<String> msg;

    public DisruptorMessage<String> getMsg() {
        return msg;
    }

    public void setMsg(DisruptorMessage<String> msg) {
        this.msg = msg;
    }
}
```

上面代码示例在 DisruptorEventHandler 里定义了成员变量 msg，它的类型是 `DisruptorMessage<String>` ，此时 `String` 是 `T` 的一个具体参数，`String` 叫做 `DisruptorMessage<T>` 的 `Type arguments`（以示区别，`T` 叫做 `DisruptorMessage<T>` 的 `Type Parameter`）。

和上面类似，同样可以用来定义一个 `Generics interface` ，不多做赘述。

## Generics Methods

定义一个 Generics method ，语法如下：

```java
public class DisruptorProducer {
    public <T> T formatMsg(T input) {
        // ...
        return input;
    }
}
```

## 类型擦除 Type erasure

对于泛型的实现，不同的编程语言实现不同，Java 采用的方法叫 Type erasure 泛型擦除。泛型擦除是指，在编译时期把代码中的泛型类型移除掉，Java 语言在编译时把参数类型转成了第一个边界，或者是 Object 类型，在 JVM 里是没有泛型信息的。如下：

```java
// 编译前 data 是 T 类型
public class Node<T> {
    private T data;
    private Node<T> next;

    public Node(T data, Node<T> next) {
        this.data = data;
        this.next = next;
    }

    public T getData() { return data; }
    // ...
}

// 编译后 data 是 Object 类型
public class Node {
    private Object data;
    private Node next;

    public Node(Object data, Node next) {
        this.data = data;
        this.next = next;
    }

    public Object getData() { return data; }
    // ...
}
```

关于泛型的使用、类型擦除的优缺点网上有很多介绍文章，笔者不做过多介绍。引起笔者好奇的是，前面提到了 Java 实现泛型是在编译期将泛型类型转为 Object 类型，那如果有个变量是泛型类型，那在运行时有办法知道它具体是什么类型的吗？答案是有的。

## JDK 自带方法获取泛型参数真实类型

在 Java 里，由于运行时已没有泛型信息，获取某个对象的真实类型只能采用反射。

不过在介绍如何利用反射获取真实参数类型前，需要先认识下 `java.lang.reflect.Type`，Type 是 Java 里所有类型的超级接口（superinterface），它可以代表任何类型。看下面代码：

```java
public class TypePoc {
    // Primitive type 基本类型
    private int a;
    // Raw type 原始类型
    private DisruptorMessage msg;
    // Array type 数组
    private String[] list;
    // Parameterized Type 参数化类型
    private List<String> strList;

    public static void main(String[] args) throws NoSuchFieldException {
        Type aType = TypePoc.class.getDeclaredField("a").getGenericType();
        Type msgType = TypePoc.class.getDeclaredField("msg").getGenericType();
        Type listType = TypePoc.class.getDeclaredField("list").getGenericType();
        Type strListType = TypePoc.class.getDeclaredField("strList").getGenericType();

        System.out.println(aType);
        System.out.println(msgType);
        System.out.println(listType);
        System.out.println(strListType);
    }
}
```

运行上面代码，会打印下面日志：

```bash
int
class com.xx.DisruptorMessage
class [Ljava.lang.String;
java.util.List<java.lang.String>
```

可以看到 Type 可以用来承接所有类型，**Type 是反射里的很重要的 interface，有了 Type 后通过反射获取类型才能行通**。

### 反射获取类成员变量的类型信息

利用反射获取类的成员变量类型信息可以分为 3 步：

1. 根据 class.getDeclaredField("字段名") 拿到 Field。
2. 使用 Field.getGenericType() 方法取字段的 Type 。
3. 调用 Typte 的 getActualTypeArguments() 方法拿到 "子 Type"。

如下代码示例：

```java
public class DisruptorProducer {
    private DisruptorMessage<Map<String, List<String>>> msg;

    public static void main(String[] args) throws NoSuchFieldException {
        Field msgField = DisruptorProducer.class.getDeclaredField("msg");
        ParameterizedType firstType = (ParameterizedType) msgField.getGenericType();
        System.out.printf("1 layer: %s, 1 layer size: %s%n", firstType, 1);

        ParameterizedType secondType = (ParameterizedType) firstType.getActualTypeArguments()[0];
        System.out.printf("2 layer: %s, 2 layer size: %s%n", secondType, firstType.getActualTypeArguments().length);

        Type thirdType1 = secondType.getActualTypeArguments()[0];
        System.out.printf("3 layer.1: %s, 3 layer size: %s%n", thirdType1, secondType.getActualTypeArguments().length);

        ParameterizedType thirdType2 = (ParameterizedType) secondType.getActualTypeArguments()[1];
        System.out.printf("3 layer.2: %s, 3 layer size: %s%n", thirdType2, secondType.getActualTypeArguments().length);

        Type fourthType = thirdType2.getActualTypeArguments()[0];
        System.out.printf("4 layer: %s, 4 layer size: %s%n", fourthType, thirdType2.getActualTypeArguments().length);
    }
}
```

运行上面代码会打印日志如下：

```bash
1 layer: com.xxx.DisruptorMessage<java.util.Map<java.lang.String, java.util.List<java.lang.String>>>, 1 layer size: 1
2 layer: java.util.Map<java.lang.String, java.util.List<java.lang.String>>, 2 layer size: 1
3 layer.1: class java.lang.String, 3 layer size: 2
3 layer.2: java.util.List<java.lang.String>, 3 layer size: 2
4 layer: class java.lang.String, 4 layer size: 1
```

通过下面的图能更直观反映上面示例中"泛型嵌套"的层级关系。

{{< image-resize "images/layer.png" >}}

可以看到，通过不断地获取某个字段的子 Type 来一层层剥开类型，就可以拿到这个字段的所有类型。

### 反射获取方法的入参、出参的类型信息

使用反射获取类方法的泛型类型信息可以分成 4 步：

1. 使用 class.getDeclaredMethods() 方法获取所有方法。
2. 使用 method.getGenericParameterTypes() 获取方法的入参。
3. 使用 method.getGenericReturnType() 获取方法的出参。
4. 2、3 步获取到的都是 Type 类型实例，使用 Type.getActualTypeArguments() 也就可以一层层剥开类型了，拿到方法出、入参的所有类型。

```java
public class MyClass {
    public List<String> setStringList(List<String> list) {
        return list;
    }

    public static void main(String[] args) {
        Method[] methods = MyClass.class.getDeclaredMethods();
        System.out.printf("MyClass has %d methods.%n", methods.length);

        for (Method method : methods) {
            Type[] genericParameterTypes = method.getGenericParameterTypes();
            System.out.printf("method-[%s] has %d parameters.%n", method, genericParameterTypes.length);

            for (int i = 0; i < genericParameterTypes.length; i++) {
                Type paramType = genericParameterTypes[i];
                if (paramType instanceof ParameterizedType pType) {
                    System.out.printf("No.%d parameter type is %s.%n", i, pType);

                    Type[] argTypes = pType.getActualTypeArguments();
                    System.out.printf("No.%d parameter type has %d subType.%n", i, argTypes.length);
                    for (int j = 0; j < argTypes.length; j++) {
                        System.out.printf("No.%d parameter type No.%d subType is %s.%n", i, j, argTypes[j]);
                    }
                }
            }

            Type returnType = method.getGenericReturnType();
            System.out.printf("\nmethod-[%s] return type is %s.%n", method, returnType);
            if (returnType instanceof ParameterizedType rType) {
                Type[] argTypes = rType.getActualTypeArguments();
                for (int i = 0; i < argTypes.length; i++) {
                    System.out.printf("No.%d return subType is %s.%n", i, argTypes[i]);
                }
            }
            System.out.println("\n-----------------------------\n");
        }
    }
}
```

可以感觉到，利用反射实现稍微复杂点的逻辑就难以理解了，运行上面代码会打印日志：

```bash
MyClass has 2 methods.
method-[public static void com.xx.MyClass.main(java.lang.String[])] has 1 parameters.

method-[public static void com.xx.MyClass.main(java.lang.String[])] return type is void.

-----------------------------

method-[public java.util.List com.xx.MyClass.setStringList(java.util.List)] has 1 parameters.
No.0 parameter type is java.util.List<java.lang.String>.
No.0 parameter type has 1 subType.
No.0 parameter type No.0 subType is class java.lang.String.

method-[public java.util.List com.xx.MyClass.setStringList(java.util.List)] return type is java.util.List<java.lang.String>.
No.0 return subType is class java.lang.String.

-----------------------------
```

综上，可以看到通过 JDK 自带方法运行时获取泛型真实类型，需要**重复**经过这些步骤：

- z = class.getDeclaredXxx()
- type = z.getGenericXxxType()
- innerType = type.getActualTypeArguments()

除此以外，还需要写类型强转、指定数组索引下标等代码，当这些黑魔法大量出现时代码就会难以读懂，且容易出错 ☹️。好在 Spring framework 提供了更好用的 ResolvableType 来帮助我们达到相同的目的。

## Spring ResolvableType 获取泛型参数真实类型

Spring 项目里大量使用了泛型，如果没有 ResolvableType 加持，项目肯定会更复杂、难懂。ResolvableType 没有 public 构造方法，可以通过下面几种方法获得到 ResolvableType 实例。

{{<image-resize "images/获得ResolvableType的方法.png">}}

归纳下支持如下几种类型：

- class
- instance
- field
- Constructor
- Method
- MethodParameter
- ArrayComponent
- Type 等

看到构建 ResolvableType 使用的参数，大多都是 `java.lang.reflect` 包下面的类型，和使用 JDK 自带方法来获取泛型参数真实类型需要的入参类型差不多，故也可以猜测 ResolvableType 大概也是利用反射实现功能。

### ResolvableType 用法

下面介绍几种常用的 ResolvableType 使用方法。

#### 获取父类和实现的接口真实类型

```java
// 定义一个接口
public interface Draw<T> {
    T get();
}

// 定义一个类
public abstract class AbstractDraw<T> {
    public T member;
}

// 定义实现类
public class ResolvableTypePoc extends AbstractDraw<String> implements Draw<Long> {
    @Override
    public Long get() {
        return 0L;
    }

    public static void main(String[] args) {
        // 获取父类泛型类型
        ResolvableType[] generics = ResolvableType.forClass(ResolvableTypePoc.class).getSuperType().getGenerics();
        for (ResolvableType generic : generics) {
            System.out.println(generic.getType());
        }

        // 获取实现的接口泛型类型
        ResolvableType[] interfaces = ResolvableType.forClass(ResolvableTypePoc.class).getInterfaces();
        for (ResolvableType anInterface : interfaces) {
            ResolvableType[] generics1 = anInterface.getGenerics();
            for (ResolvableType resolvableType : generics1) {
                System.out.println(resolvableType.getType());
            }
        }
    }
}
```

运行上面代码会打印如下日志：

```bash
class java.lang.String
class java.lang.Long
```



#### 获取字段真实类型

```java
public class ResolvableTypePoc extends AbstractDraw<String> implements Draw<Long> {
    @Override
    public Long get() {
        return 0L;
    }

    public static void main(String[] args) throws NoSuchFieldException {
        ResolvableType memberType = ResolvableType.forField(AbstractDraw.class.getField("member"),
                ResolvableTypePoc.class);
        System.out.println(memberType);
    }
}
```

运行上面代码会打印如下日志：

```bash
java.lang.String
```

`ResolvableType.forField(AbstractDraw.class.getField("member"), ResolvableTypePoc.class)` 这个方法第一参数是 Field 类型代表要查找的字段，第二个参数是 Class 类型代表实现类，方法作用是找出在指定的**实现类里 Field 变量的真实类型**是什么，因为 AbstractDraw 里的成员变量是泛型 T，AbstractDraw 的不同继承者传入的 Type argument 可能不同。如果不传第二个参数 `ResolvableType.forField(AbstractDraw.class.getField("member"))` 得到的结果就是类型通配符 ` ?`。

#### 获取嵌套类型真实类型

```java
public class ResolvableTypePoc extends AbstractDraw<String> implements Draw<Long> {
    private List<Map<String, Set<Long>>> list;

    @Override
    public Long get() {
        return 0L;
    }

    public static void main(String[] args) throws NoSuchFieldException {
        ResolvableType firstType = ResolvableType.forField(ResolvableTypePoc.class.getDeclaredField("list"));
        System.out.println("1st " + firstType);

        ResolvableType secondGeneric = firstType.getGeneric(0);
        System.out.println("2nd " + secondGeneric);

        ResolvableType thirdGeneric1 = secondGeneric.getGeneric(0);
        System.out.println("3rd 1 " + thirdGeneric1);

        ResolvableType thirdGeneric2 = secondGeneric.getGeneric(1);
        System.out.println("3rd 2 " + thirdGeneric2);

        ResolvableType fourthGeneric = thirdGeneric2.getGeneric(0);
        System.out.println("4th " + fourthGeneric);
    }
}
```

运行上面的代码会打印日志：

```bash
1st java.util.List<java.util.Map<java.lang.String, java.util.Set<java.lang.Long>>>
2nd java.util.Map<java.lang.String, java.util.Set<java.lang.Long>>
3rd 1 java.lang.String
3rd 2 java.util.Set<java.lang.Long>
4th java.lang.Long
```

层级图如下：

{{<image-resize "images/layer2.png">}}

👍 可以明显感觉到，ResolvableType 带来的简便性不是一点点，代码更易懂、清晰，也少了 error prone，看上去也不是黑魔法了。

### ResolveType 是怎么做到的？

以 `ResolvableType.forField` 方法为例，从源码入手，看看 ResolveableType 的实现秘密。

```java
// 此方法是 ResolvableType 提供的公用方法，用来解析 Field 的 Type。
public static ResolvableType forField(Field field) {
	Assert.notNull(field, "Field must not be null");
	return forType(null, new FieldTypeProvider(field), null);
}
```

方法里用到了 `FieldTypeProvider`，源码里看到 `FieldTypeProvider` 实现了 `TypeProvider`  接口。TypeProvider 是用来做什么的呢？

```java
interface TypeProvider extends Serializable {

    /**
     * Return the (possibly non {@link Serializable}) {@link Type}.
     */
    @Nullable
    Type getType();

    /**
     * Return the source of the type, or {@code null} if not known.
     * <p>The default implementation returns {@code null}.
     */
    @Nullable
    default Object getSource() {
       return null;
    }
}
```

可以看到，它提供了 getType() 方法，getType() 返回类型是 Type，前面提到过 Type 是所有类型的 super interface，我们可以猜测 TypeProvider 可以用来更统一、便捷的获取某个类型的 Type 信息。Spring 项目里已有 `FieldTypeProvider`、`MethodParameterTypeProvider` 等 class 实现了 TypeProvider 类型。`FieldTypeProvider` 的作用应该是提供了更统一的方法来获取字段 Field 们的 Type 信息，`MethodParameterTypeProvider` 应该是提供了更统一的方法来获取方法的入参们的 Type 信息。FieldTypeProvider 源码如下：

```java
static class FieldTypeProvider implements TypeProvider {

    private final String fieldName;

    private final Class<?> declaringClass;

    private transient Field field;

    public FieldTypeProvider(Field field) {
       this.fieldName = field.getName();
       this.declaringClass = field.getDeclaringClass();
       this.field = field;
    }

    @Override
    public Type getType() {
       return this.field.getGenericType();
    }

    @Override
    public Object getSource() {
       return this.field;
    }
  
  	// 重写了 getType 和 getSource 方法，getType() 是 FieldTypeProvider 核心的方法，后面会用到。
  	// 能看到，FieldTypeProvider 的 getType() 实现很简单，直接调用了 field.getGenericType() 方法。

    private void readObject(ObjectInputStream inputStream) throws IOException, ClassNotFoundException {
       inputStream.defaultReadObject();
       try {
          this.field = this.declaringClass.getDeclaredField(this.fieldName);
       }
       catch (Throwable ex) {
          throw new IllegalStateException("Could not find original class structure", ex);
       }
    }
}
```

FieldTypeProvider 的实现比较简单，看完 FieldTypeProvider 后，进一步看 forType() 方法，下面代码的 forType() 是上面提到的 forType() 方法的重载。

```java
static ResolvableType forType(
       @Nullable Type type, @Nullable TypeProvider typeProvider, @Nullable VariableResolver variableResolver) {
  	// 从调用处能看到，
  	// 传入的 type = null, variableResolver = null, 
  	// typeProvider 是 FieldTypeProvider 的实例。
  	// type = null、typeProvider 是刚被创建，所以下面的判断条件成立。

    if (type == null && typeProvider != null) {
       type = SerializableTypeWrapper.forTypeProvider(typeProvider);
    }
    if (type == null) {
       return NONE;
    }

    ...
}
```

因为 forType 方法传入的参数 type 的值是 null，但 forType 方法里后面又**需要用到 type 参数**，所以代码里使用`SerializableTypeWrapper.forTypeProvider()` 方法来尝试获取到 type 的值。可以往下多看几行代码，如果使用 forTypeProvider 还不能解析到 type，那么 forType 方法就返回默认值 NONE。所以我们能知道 forTypeProvider 方法也不是 100% 能解析到 type。到此我们需要先看看 SerializableTypeWrapper.forTypeProvider 方法做了什么事。

```java
static Type forTypeProvider(TypeProvider provider) {

    Type providedType = provider.getType();
    // 传入的 TypeProvider 是 FieldTypeProvider 的实例，
  	// 此处 provider.getType() 调用的是 FieldTypeProvider 重写的 getType() 方法，
  	// 重写的 getType() 方法里直接调用了 field.getGenericType()。
  	// 实际上就是获取到了传入的 Field 的 Type。
  
  
    if (providedType == null || providedType instanceof Serializable) {
       // No serializable type wrapping necessary (e.g. for java.lang.Class)
       return providedType;
    }
    if (NativeDetector.inNativeImage() || !Serializable.class.isAssignableFrom(Class.class)) {
       // Let's skip any wrapping attempts if types are generally not serializable in
       // the current runtime environment (even java.lang.Class itself, e.g. on GraalVM native images)
       return providedType;
    }

    // Obtain a serializable type proxy for the given provider...
    Type cached = cache.get(providedType);
    if (cached != null) {
       return cached;
    }
  	// 从 chache 里取出缓存的 providedType 对应的 Type，如果缓存不为 null 则立即返回。
  	// chache 里 key 和 value 分别是什么呢？想要回答这个问题需要先往下看。  

    for (Class<?> type : SUPPORTED_SERIALIZABLE_TYPES) {
      // SUPPORTED_SERIALIZABLE_TYPES 是由 GenericArrayType.class, ParameterizedType.class, 
  		// TypeVariable.class, WildcardType.class 组成的固定数组。
      
       if (type.isInstance(providedType)) {
         // type.isInstance() 由 Class 类提供，它的作用和 instanceof 操作符一样。
         // 即当 providedType 是上面 4 个 class 里的 *某个 class 的实例* 时，才执行下面代码。
         
          ClassLoader classLoader = provider.getClass().getClassLoader();
          Class<?>[] interfaces = new Class<?>[] {type, SerializableTypeProxy.class, Serializable.class};
          InvocationHandler handler = new TypeProxyInvocationHandler(provider);
          cached = (Type) Proxy.newProxyInstance(classLoader, interfaces, handler);
          cache.put(providedType, cached);
          return cached;
         
         // 上面代码片段里，变量 classLoader、interface、handler 都是给 Proxy.newProxyInstance(arg1, arg2, arg3) 
         // 方法使用，Proxy.newProxyInstance 方法是 JDK 自带的字节码生成工具，为了保持篇幅的整洁和聚焦，就不展开介绍
         // Proxy.newProxyInstance 的用法和原理了，此处知道是利用 JDK 自带的字节码生成工具创建了一个对象 Object，并被
         // 强转成了 Type 类型即可，感兴趣的读者可以自行研究其原理。
         
         // 通过 Proxy.newProxyInstance 获得到 cached 后将其放入 cache 里，方法返回 cached。
         // 到此我们知道了 cache 的 key 和 value 分别是什么，key 是 TypeProvider.getType() 方法返回的 Type 类型对象，
         // value 是通过 Proxy.newProxyInstance() 方法创建的代理对象。
       }
    }
    throw new IllegalArgumentException("Unsupported Type class: " + providedType.getClass().getName());
}
```

注意下，forTypeProvider 方法的返回值有两种，**一种是从 provider.getType() 得到的** ，**另一种是用字节码生成工具生成的**。

看完 SerializableTypeWrapper.forTypeProvider 源码后，回到 forType 方法内容，继续往下看。

```java
static ResolvableType forType(
       @Nullable Type type, @Nullable TypeProvider typeProvider, @Nullable VariableResolver variableResolver) {

    if (type == null && typeProvider != null) {
       type = SerializableTypeWrapper.forTypeProvider(typeProvider);
    }
    if (type == null) {
       return NONE;
    }
  
  	// 上面都已经看过，接着往下读。

    // For simple Class references, build the wrapper right away -
    // no expensive resolution necessary, so not worth caching...
    if (type instanceof Class) {
       return new ResolvableType(type, null, typeProvider, variableResolver); // 1️⃣
    }
  	// 当通过 forTypeProvider 方法得到的 type 是 Class 类型时，forType 方法直接返回 new ResolvableType。 
  	// 此处用到了 ResolvableType 的私有构造方法。

    // Purge empty entries on access since we don't have a clean-up thread or the like.
    cache.purgeUnreferencedEntries();
  	// 方法 purgeUnreferencedEntries 是由 ConcurrentReferenceHashMap 提供，作用是清除 Map 中已经
  	// 被 GC 回收调的 entries

    // Check the cache - we may have a ResolvableType which has been resolved before...
    ResolvableType resultType = new ResolvableType(type, typeProvider, variableResolver);
    ResolvableType cachedType = cache.get(resultType);
    if (cachedType == null) {
       cachedType = new ResolvableType(type, typeProvider, variableResolver, resultType.hash);
       cache.put(cachedType, cachedType);
    }
    resultType.resolved = cachedType.resolved;
    return resultType;
}
```

ResolvableType 有多个私有构造方法，上面 1️⃣ 处使用的构造方法源码如下：

```java
private ResolvableType(Type type, @Nullable ResolvableType componentType,
       @Nullable TypeProvider typeProvider, @Nullable VariableResolver variableResolver) {

    this.type = type;
    this.componentType = componentType;
    this.typeProvider = typeProvider;
    this.variableResolver = variableResolver;
    this.hash = null;
    this.resolved = resolveClass();
  
  // 方法入参 type 是经过 SerializableTypeWrapper.forTypeProvider 计算得到。
  // 方法入参 typeProvider 实例是 FieldTypeProvider 类型。
  
  // 调用 resolveClass() 方法初始化 resolved 变量。
}

@Nullable
private Class<?> resolveClass() {
    if (this.type == EmptyType.INSTANCE) {
       return null;
    }  
    if (this.type instanceof Class<?> clazz) {
       return clazz;
    }  
    // 当 type 是 Class<?> 实例时
  
    if (this.type instanceof GenericArrayType) {
       Class<?> resolvedComponent = getComponentType().resolve();
       return (resolvedComponent != null ? Array.newInstance(resolvedComponent, 0).getClass() : null);
    }
    return resolveType().resolve();
}
```





参考链接

https://docs.oracle.com/javase/tutorial/java/generics/index.html

https://en.wikipedia.org/wiki/Generic_programming

https://en.wikipedia.org/wiki/Generics_in_Java

https://en.wikipedia.org/wiki/Type_erasure & https://en.wikipedia.org/wiki/Type_inference

https://en.wikipedia.org/wiki/Reification_(computer_science)

