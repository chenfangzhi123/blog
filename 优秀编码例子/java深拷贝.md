
[TOC]

在java开发的过程中我们很多时候会有深拷贝需求，比如将一个请求体拷贝多次，修改成多个不同版笨，分别发给不同的服务，在比如维护不同的缓存时。还有些时候并不需要深拷贝，只是简单的类型转换，比如到将do对象转换为dto对象返回给前端，其中两者的字段基本相同，只是类名不一样。本文主要罗列了下自己总结的拷贝方式和适合的场景（深浅拷贝原理文章很多，本文不再解释）。


拷贝过程中用到的Bean定义：

```java

@Data
public class Source {
    String a;
    Filed1 filed1;
    Filed1 filed2;
    List<Filed1> fileds;


    @NoArgsConstructor
    @AllArgsConstructor
    @Data
    public static class Filed1 {
        String id;
    }
}

```
## 深拷贝


### 1. 手动new


```java
    Source source = getSource();
    Source target = new Source();
    target.setFiled1(new Source.Filed1(source.getFiled1().getId()));
    target.setFiled2(new Source.Filed1(source.getFiled2().getId()));

    if (source.getFileds() != null) {
        ArrayList<Source.Filed1> fileds = new ArrayList<>(source.getFileds().size());
        for (Source.Filed1 filed : source.getFileds()) {
            fileds.add(new Source.Filed1(filed.getId()));
        }
        target.setFileds(fileds);
    }
```

手动new非常简单，但是非常繁琐不利于后期的维护，每次修改类定义的时候需要修改相应的copy方法，不过性能非常高。

### 2. clone方法

```java

    //  Source类
    public Source clone() {
        Source clone = null;
        try {
            clone = (Source) super.clone();
            clone.setFiled1(filed1.clone());
            clone.setFiled2(filed2.clone());
            //列表的克隆
            if (fileds != null) {
                ArrayList<Filed1> target = new ArrayList<>(this.fileds.size());
                for (Filed1 filed : this.fileds) {
                    target.add(filed.clone());
                }
                clone.setFileds(target);
            }
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return clone;
    }
    // Filed1类
    public Filed1 clone() throws CloneNotSupportedException {
            return (Filed1) super.clone();
    }

```

在重写clone方法的时候，如果类的字段类型是String和Integer等不可变类型，那么source实例对应的字段是可以复用的，以为这个字段值不能被修改。如果字段类型是可变类型则也需要重写，如Source中Filed1字段类型不是不可变类型，则也需要重写clone方法，另外注意重写clone方法的类必须实现Cloneable类（`public class Source implements  Serializable`），否则会抛出CloneNotSupportedException。

### 3. java自带序列化

```java
    ByteArrayOutputStream out = new ByteArrayOutputStream();
    new ObjectOutputStream(out).writeObject(source);

    ObjectInputStream in = new ObjectInputStream(new ByteArrayInputStream(out.toByteArray()));
    Source target = (Source) in.readObject();

    // spring中封装了下可以直接使用
    // Source target = (Source) SerializationUtils.deserialize(SerializationUtils.serialize(source));
```


这个方法很多书中都有提起，因为序列化来实现深度拷贝代码比较简单，可扩展性好，后期添加字段无需修改实现，不过类需要继承标记接口Serializable（`public class Source implements Serializable`）。不过这个方法没有什么实际用途，因为确实性能非常低。

### 4. json序列化

```java
public class JsonCopy {
    private static ObjectMapper mapper = new ObjectMapper();


    public static String encodeWithoutNull(Object obj) throws Exception {
        return mapper.writeValueAsString(obj);
    }

    public static <T> T decodeValueIgnoreUnknown(String str, Class<T> clazz) throws Exception {
        return mapper.readValue(str, clazz);
    }


    // 一千万次  15.3秒
    public static <T> T copy(T source, Class<T> tClass) throws Exception {
        return decodeValueIgnoreUnknown(encodeWithoutNull(source), tClass);
    }
}
```

一个简单的工具类，利用了Jackson库，性能一般，不过扩展性好，比java自带序列化很大提升。

### 性能测试

我在自己的机器上用每种方法实现Source对象的一千万次拷贝，测试了时间。结果如下：
  
|  类型   | 测试结果  |
|  ----  | ----  |
| 手动new  | 一千万次 774毫秒 |
| clone方法  | 一千万次  827毫秒 |
| java自带序列化  | 一千万次 109.7秒 |
| json序列化  | 一千万次  15.3秒 |



### 深拷贝总结

从可扩展性和性能方面的考虑，如果注重性能，那么使用手动new和clone方法，如果注重扩展性那么使用java自带序列化和json序列化。平时的使用中，优先使用json序列化，因为大部分场景下cpu不是瓶颈，在一些热点代码中改用重写clone方法。使用clone方法和手动New两个性能和可维护性都类似，只不过看你的喜好，我是认为clone比较符合Java风格，将对象的clone方法写在那个类中。


## 浅拷贝


### 1. spring BeanUtils（Apache BeanUtils）

```java
Source source = getSource();
Source target = new Source();
BeanUtils.copyProperties(source, target);
```
spring的BeanuUtils和Apache BeanUtils原理都类似，都是利用反射获取了对象的字段，逐个赋值，性能方面其实也是比较好了，虽然利用了反射，但是内部缓存了反射的结果，后面在复制的时候可以直接取缓存的结果。反射的性能损耗在获取Class信息那一块，在调用的开销和普通调用的类似，Jvm也会使用Jit进行优化。

### 2. mapstruct

```java

@Mapper
public interface SourceMapper {

    SourceMapper INSTANCE = Mappers.getMapper(SourceMapper.class);

    Source copy(Source car);
}

Source target = SourceMapper.INSTANCE.copy(source);

```

mapstruct和lombok的原理类型，在编译期根据你的注解生成所需要的方法，所以他的性能理论上和手写是一样的，现在springboot也可以和他很好的结合，如果遇到了对象拷贝的性能瓶颈可以考虑用下这个类库。不过遗憾的是他并不支持深拷贝。https://github.com/mapstruct/mapstruct/issues/695


### 性能测试

|  类型   | 测试结果  |
|  ----  | ----  |
| BeanUtils  |一千万次  1825毫秒 |
| mapstruct  | 一千万次  235毫秒 |


### 浅拷贝总结

浅拷贝也可以看到可以复制不同对象的实例字段，这是序列化和Clone方法等不具备的优势，在转化Bean的时候十分有用。在一般情况下，推荐使用Spring的BeanUtils类，不用引入额外的依赖，性能也够用。如果在高并发的场景下，可以考虑通过mapstruct进行优化，两者会有一个数量级的差距。