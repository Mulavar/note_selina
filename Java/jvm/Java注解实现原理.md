## 1. 先验知识

阅读本文需要先熟悉以下知识：

1. 如何编写Java注解
2. Java反射的基本使用
3. Java动态代理
4. 对class文件格式有基本的了解



## 2. 注解的字节码原理

根据JAVA虚拟机规范标准（JAVA SE 8)，注解其实是class文件的**属性**（变长字节数组），根据不同的场景，有不同的字节码格式表示，具体可以分为：

since1.5

- RuntimeVisibleAnnotations：运行时可见注解
- RuntimeInvisibleAnnotations：运行时不可见注解
- RuntimeVisibleParameterAnnotations：运行时可见方法参数注解
- RuntimeInvisibleParameterAnnotations：运行时不可见方法参数注解
- AnnotationDefault：注解默认值

since1.8

- RuntimeVisibleTypeAnnotations：运行时可见类型注解
- RuntimeInvisibleTypeAnnotations：运行时不可见类型注解

其中，反射API调用会涉及到的主要有这几类RuntimeVisibleAnnotations、RuntimeVisibleParameterAnnotations、AnnotationDefault，由于RuntimeVisibleParameterAnnotations的解析逻辑和RuntimeVisibleAnnotations基本一致（两者区别就是前者是使用在方法形参上的注解），所以我们以RuntimeVisibleAnnotations为例。

RuntimeVisibleParameterAnnotations的字节码表示如下：

```java
RuntimeVisibleAnnotations_attribute {
    // 前6个字节固定格式
    u2			 attribute_name_index; 
    u4			 attribute_length;
    // 表示有几个注解
    u2			 num_annotations;
    // 注解数组
    annotation	 annotations[num_annotations];
}	

annotation {
    // 主要指向该的类信息
    u2		type_index;
    // 该注解的属性键值对个数（K：属性名，V：属性值）
    u2		num_element_value_pairs;
    {
        // 指向属性名
        u2				element_name_index;
        // 属性值
        element_value	value;
    }  element_value_pairs[num_element_value_pairs];
}

element_value {
    // 表示是哪种值类型，具体的有
    // 基本类型、String、枚举、注解、Class、数组类型
    u1 tag;
    // 联合体，里面分别定义了每种值类型的格式
    union {
        // 基本类型和String类型
        u2 const_value_index;
        
        // 枚举类型
        {
            u2 type_name_index;
            u2 const_name_index;
        } enum_const_value;
        
        // class 类型
        u2 class_info_index;
        
        // 注解类型
        annotation annotation_values;
        
        // 数组类型
        {
            u2			  num_values;
            element_value values[num_values];
        }
    } value;
}
```

由上图可见，只要按照该格式定义，结合常量池就可以轻松地写出解析代码，也因此第三节我们不会讲结如何解析字节数组。

对于AnnotationDefault，我们知道声明注解可以对其成员属性使用default关键字，此时我们用注解修饰时就不用指定键值对，AnnotationDefault就是为这种场景而生的。

```java
AnnotationDefault_attribute {
    // 前6个字节固定格式
    u2 				attribute_name_index;
    u4 				attribute_length;
    // 见上一个代码块
    element_value 	default_value;
}
```



## 3. JDK注解原理

在该节的源码解读中，会省略部分不必要的源码，只留下最核心的源码讲述注解解析的流程。由于上一节已经详细解剖了RuntimeVisibleAnnotations的字节码表示，所以最内部的字节数组解析逻辑不会详述，按照源码和字节码格式一一对应即可。

**注：该节的源码取自JDK11。**



### 3.1 注解解析流程

以获取注解经常调到的`public <A extends Annotation> A getAnnotation(Class<A> annotationClass)`为入口函数开始讲解。

```java
public <A extends Annotation> A getAnnotation(Class<A> annotationClass) {
    return (A) annotationData().annotations.get(annotationClass);
}
```

该方法一共做了这么几件事：

1. 调用`annotationData()`方法创建`Class.AnnotationData`类实例，`Class.AnnotationData`是一个注解缓存类，用于缓存该类的注解信息。
2. 由于一个类（或方法、注解）可以有多种注解修饰，所以这里使用了一个Map存储**注解类**和**注解对象**的映射，即annotations，类型为Map<Class<? extends Annotation>, Annotation>，这里的K是注解类，V是**匿名的注解代理对象**。

所以下一步我们进入`annotationData()`方法，

```java
private AnnotationData annotationData() {
    while (true) { // retry loop
        AnnotationData annotationData = this.annotationData;
        int classRedefinedCount = this.classRedefinedCount;
        //1. 判断注解缓存是否失效
        if (annotationData != null &&
            annotationData.redefinedCount == classRedefinedCount) {
            return annotationData;
        }
        //2. 如果失效，则创建新的注解缓存
        AnnotationData newAnnotationData = createAnnotationData(classRedefinedCount);

        if (Atomic.casAnnotationData(this, annotationData, newAnnotationData)) {
            //3. 利用cas算法更新注解缓存
            return newAnnotationData;
        }
    }
}
```

进一步进入`createAnnotationData`方法，

```java
/**
 * 
 */
private AnnotationData createAnnotationData(int classRedefinedCount) {
    // 1.解析修饰当前类的注解
    Map<Class<? extends Annotation>, Annotation> declaredAnnotations =
        AnnotationParser.parseAnnotations(getRawAnnotations(), getConstantPool(), this);
    // 2.继承修饰父类的注解（前提是该注解可修饰，具体可了解元注解Inherited）
    Class<?> superClass = getSuperclass();
    Map<Class<? extends Annotation>, Annotation> annotations = null;
    if (superClass != null) {
        // 3.父类递归解析注解
        Map<Class<? extends Annotation>, Annotation> superAnnotations =
            superClass.annotationData().annotations;
        for (Map.Entry<Class<? extends Annotation>, Annotation> e : superAnnotations.entrySet()) {
            Class<? extends Annotation> annotationClass = e.getKey();
            if (AnnotationType.getInstance(annotationClass).isInherited()) {
                if (annotations == null) { // lazy construction
                    annotations = new LinkedHashMap<>((Math.max(
                        declaredAnnotations.size(),
                        Math.min(12, declaredAnnotations.size() + superAnnotations.size())
                    ) * 4 + 2) / 3
                                                     );
                }
                annotations.put(annotationClass, e.getValue());
            }
        }
    }
    if (annotations == null) {
        // 4.假如没有需要继承的注解，则declaredAnnotations和annotations一致
        annotations = declaredAnnotations;
    } else {
        // 5.有继承的注解，则当前类和父类冲突的注解需要被当前类覆盖
        annotations.putAll(declaredAnnotations);
    }
    // 6.构建Class.AnnotationData实例
    return new AnnotationData(annotations, declaredAnnotations, classRedefinedCount);
}
```

根据这段代码可以看到核心的解析逻辑在第一步`Map<Class<? extends Annotation>, Annotation> declaredAnnotations = AnnotationParser.parseAnnotations(getRawAnnotations(), getConstantPool(), this);`中，因此进入该方法继续查看，

```java
/**
 * 该方法声明在类AnnotaionParser中
 * @param rawAnnotations 内容按一定格式阻止的字节数组，其格式可参考Classfile标准的RunvisibleAnnotations属性结构
 * @param constPool 表示的是Classfile标准中的常量池
 * @param container 在解析复杂嵌套的注解时用到，这里不作阐述
 */
public static Map<Class<? extends Annotation>, Annotation> parseAnnotations(
    byte[] rawAnnotations,
    ConstantPool constPool,
    Class<?> container) {
    // 1.没有注解修饰时
    if (rawAnnotations == null)
        return Collections.emptyMap();

    try {
        // 2.解析注解
        return parseAnnotations2(rawAnnotations, constPool, container, null);
    } catch(/**错误处理*/) {
    }
}
```

进一步进入`parseAnnotations2(rawAnnotations, constPool, container, null);`方法，

```java
private static Map<Class<? extends Annotation>, Annotation> parseAnnotations2(
    byte[] rawAnnotations,
    ConstantPool constPool,
    Class<?> container,
    Class<? extends Annotation>[] selectAnnotationClasses) {
    Map<Class<? extends Annotation>, Annotation> result =
        new LinkedHashMap<Class<? extends Annotation>, Annotation>();
    ByteBuffer buf = ByteBuffer.wrap(rawAnnotations);
    // 1. 读取num_annotations（具体查看2.注解的字节码格式一节）的值，确定有多少个注解
    int numAnnotations = buf.getShort() & 0xFFFF;
    for (int i = 0; i < numAnnotations; i++) {
        // 2. 进一步解析单个注解
        Annotation a = parseAnnotation2(buf, constPool, container, false, selectAnnotationClasses);
        if (a != null) {
            Class<? extends Annotation> klass = a.annotationType();
            // 3. 保证解析的注解无重复
            if (AnnotationType.getInstance(klass).retention() == RetentionPolicy.RUNTIME &&
                result.put(klass, a) != null) {
                throw new AnnotationFormatError(
                    "Duplicate annotation for class: "+klass+": " + a);
            }
        }
    }
    return result;
}
```

进入`parseAnnotation2`，

```java
private static Annotation parseAnnotation2(ByteBuffer buf,
                                              ConstantPool constPool,
                                              Class<?> container,
                                              boolean exceptionOnMissingAnnotationClass,
                                              Class<? extends Annotation>[] selectAnnotationClasses) {
        int typeIndex = buf.getShort() & 0xFFFF;
        Class<? extends Annotation> annotationClass = null;
        String sig = "[unknown]";
        try {
            try {
                // 1. 读取typeIndex（该注解类名在常量池中的位置下标），并得到对应注解类
                sig = constPool.getUTF8At(typeIndex);
                annotationClass = (Class<? extends Annotation>)parseSig(sig, container);
            } catch (IllegalArgumentException ex) {
                // support obsolete early jsr175 format class files
                annotationClass = (Class<? extends Annotation>)constPool.getClassAt(typeIndex);
            }
        } catch (/**错误处理*/) {
        }
    	//......
        AnnotationType type = null;
        try {
            // 2. 创建一个AnnotationType实例，
            // AnnotationType是在运行环境中对一个注解类型的表示
            type = AnnotationType.getInstance(annotationClass);
        } catch (/**错误处理*/) {
        }

    	// 3.1 根据AnnotationType解析得到该注解的所有映射值<memberName, value>
        Map<String, Class<?>> memberTypes = type.memberTypes();
        Map<String, Object> memberValues =
            new LinkedHashMap<String, Object>(type.memberDefaults());

        int numMembers = buf.getShort() & 0xFFFF;
        for (int i = 0; i < numMembers; i++) {
            // 3.2 根据memberNameIndex解析对应的注解属性名
            int memberNameIndex = buf.getShort() & 0xFFFF;
            String memberName = constPool.getUTF8At(memberNameIndex);
            // 3.3 根据注解属性名获取属性类型（可知在第2步时，初始化创建
            // AnnotationType的过程中已经绑定了该映射关系,
            // 具体查看2.注解的字节码格式一节可知有哪几种类型
            Class<?> memberType = memberTypes.get(memberName);

            if (memberType == null) {
                skipMemberValue(buf);
            } else {
                // 3.4 对不同的类型使用不同的方式进行解析，不再展开
                Object value = parseMemberValue(memberType, buf, constPool, container);
                if (value instanceof AnnotationTypeMismatchExceptionProxy)
                    ((AnnotationTypeMismatchExceptionProxy) value).
                        setMember(type.members().get(memberName));
                memberValues.put(memberName, value);
            }
        }
    	// 4. 核心的一步，创建代理对象。
        return annotationForMap(annotationClass, memberValues);
    }
```

至此，我们基本了解了JDK解析注解的流程，但这里仍有两点没有说清：

1. 如何处理注解中带有default关键字的属性；
2. 如何实现代理；



### 3.2 如何处理注解中的默认属性值

针对第一个问题，假设一个注解声明如下：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface Color {
    String color() default "red";
}
```

使用它修饰某个类：

```java
@Color
class Bird{}
```

此时，Bird的classfile结构的RunvisibleAnnotations属性是不会包含该<color, red>的映射信息的，这个信息实际存储在Color的classfile中的AnnotationDefault属性中，因此实际是在初始化创建AnnotaionType——`AnnotationType.getInstance(annotationClass)`中去处理的，在`AnnotationType.getInstance(annotationClass)`中有一行`result = new AnnotationType(annotationClass)`负责实际创建对象，逻辑如下：

```java
private AnnotationType(final Class<? extends Annotation> annotationClass) {
    // 1. 获取该注解类所有的方法，而在方法中有一个成员变量annotationDefault,
    // 就是上面提到的classfile中的AnnotationDefault属性，也是一个字节数组
    Method[] methods =
        AccessController.doPrivileged(new PrivilegedAction<>() {
            public Method[] run() {
                // Initialize memberTypes and defaultValues
                return annotationClass.getDeclaredMethods();
            }
        });

    // 2. 注解属性名 => 注解类型 的映射
    memberTypes = new HashMap<>(methods.length+1, 1.0f);
    // 3. 注解属性名 => 注解默认值 的映射
    memberDefaults = new HashMap<>(0);
    members = new HashMap<>(methods.length+1, 1.0f);

    for (Method method : methods) {
        if (Modifier.isPublic(method.getModifiers()) &&
            Modifier.isAbstract(method.getModifiers()) &&
            !method.isSynthetic()) {
            if (method.getParameterTypes().length != 0) {
                throw new IllegalArgumentException(method + " has params");
            }
            String name = method.getName();
            Class<?> type = method.getReturnType();
            memberTypes.put(name, invocationHandlerReturnType(type));
            members.put(name, method);
			// 4. 在这里获得注解属性的默认值，进一步点进去，
            // 可以看到里面也调用了AnnotationParser.parseMemberValue方法
            // 解析字节数组annotationDefault
            Object defaultValue = method.getDefaultValue();
            if (defaultValue != null) {
                memberDefaults.put(name, defaultValue);
            }
        }
    }

    // 5. 其他初始化
}
```



### 3.3 注解的代理实现

针对第二个问题，在3.1里面可以看到最后一行代码是`annotationForMap(annotationClass, memberValues)`，返回了一个Annotation类型的对象，即注解的代理对象，其运用的是Java动态代理的知识。

```java
public static Annotation annotationForMap(
    final Class<? extends Annotation> type,	
    final Map<String, Object> memberValues)
{
    return AccessController.doPrivileged(new PrivilegedAction<Annotation>() {
        public Annotation run() {
            // 生成代理对象，对该注解所有的方法调用最后都会交给
            // AnnotationInvocationHandler对象处理
            return (Annotation) Proxy.newProxyInstance(
                type.getClassLoader(), new Class<?>[] { type },
                new AnnotationInvocationHandler(type, memberValues));
        }});
}
```

至此，我们可以明白，其实所有对注解的方法调用，最终都是交给了`AnnotationInvocationHandler`，在生成`AnnotationInvocationHandler`对象时，我们会传入：

- type：注解类型
- memberValues：注解属性名=>注解属性值的映射

下面具体看一下`AnnotationInvocationHandler`的部分核心代码—`invoke()`方法，

```java
class AnnotationInvocationHandler implements InvocationHandler, Serializable {
    private final Class<? extends Annotation> type;
    private final Map<String, Object> memberValues;
    
    public Object invoke(Object proxy, Method method, Object[] args) {
        String member = method.getName();
        int parameterCount = method.getParameterCount();

        // 1. 处理内置继承的Object类和Annotation接口的方法
        if (parameterCount == 1 && member == "equals" &&
                method.getParameterTypes()[0] == Object.class) {
            return equalsImpl(proxy, args[0]);
        }
        if (parameterCount != 0) {
            throw new AssertionError("Too many parameters for an annotation method");
        }

        if (member == "toString") {
            return toStringImpl();
        } else if (member == "hashCode") {
            return hashCodeImpl();
        } else if (member == "annotationType") {
            return type;
        }

        // 2. 处理注解属性名对应的方法
        Object result = memberValues.get(member);

        if (result == null)
            throw new IncompleteAnnotationException(type, member);

        if (result instanceof ExceptionProxy)
            throw ((ExceptionProxy) result).generateException();

        if (result.getClass().isArray() && Array.getLength(result) != 0)
            result = cloneArray(result);

        return result;
    }
}
```

可以看到，`invoke`里主要包含了两部分逻辑：

1. 对内置继承方法的特殊处理，这一部分`AnnotationInvocationHandler`已有实际的实现方法；
2. 对注解属性对应的方法的处理，则直接从`memberValues`获取对应的属性值。





## 4. 小结

梳理下来，其实可以发现，注解的实现原理主要就是两部分：

1. 按照字节码格式解析；
2. 将对注解的方法调用通过代理的方式传递给代理对象。

而classfile除了`RuntimeVisibleAnnotations`属性以外，还有`RuntimeInVisibleAnnotations`（针对只在编译期出现的注解），`RuntimeVisibleParameterAnnotations`（针对方法参数运行时可见的注解），`RuntimeInVisibleParameterAnnotations`（针对方法参数只在编译期可见的注解）等注解属性，其字节码格式都大同小异，更多是JDK层逻辑代码稍微的变动。

