# Stream

在开始之前，先来看一下Stream操作的一些分类，Stream相关的操作按照过程可以分为两种：
- 中间操作：中间操作按照有无状态又可以分为两种
    - 无状态：指的是之前的元素处理对当前的元素处理没有影响，比如filter()、map()等等操作
    - 有状态：指的是之前的元素对当前的处理有影响，一般都需要拿到所有的元素之后，才能进行这一步操作，比如sorted()、distinct()等等
- 结束操作：结束状态按照是否短路也可以分为两种
    - 短路操作：表示需要所有的元素都处理完才能得到结果，比如collect()、forEach()等等操作
    - 非短路操作：表示只需要遇到某些符合条件的元素就可以得到结果，比如anyMatch()、findAny()、findFirst()等等操作

## 一个简单的例子

我们经常会遇到如下的场景：

```java
public class StreamDemo {

    public static List<String> cal(final List<Integer> integerList) {
        // 假设输入的 integerList 有 100W 个元素
        // 那么在某个瞬间，会同时存在 integerList、doubles、strings 这三个对象
        // 并且每个列表里的元素都是 100W 个，加起来就是 300W 个
        // 就会引起内存爆炸
        final List<Double> doubles = new ArrayList<>();
        for (final Integer i : integerList) {
            doubles.add(i * Math.PI);
        }
        final List<String> strings = new ArrayList<>();
        for (final Double d : doubles) {
            strings.add(String.valueOf(d));
        }
        return strings;
    }

    public static List<String> cal01(final List<Integer> integers) {
        // 优化后的写法如下，同样输入是 100W 个元素的情况下
        // 这种写法所占用的内存差不多是 200W
        final List<String> strings = new ArrayList<>();
        for (final Integer i : integers) {
            strings.add(String.valueOf(i * Math.PI));
        }
        return strings;
    }

    public static List<String> cal02(final List<Integer> integers) {
        // 用 Stream 的写法如下，内存占用也是 200W
        // cal01 和 cal02 对内存改善的作用差不多，但用了 Stream API 的代码显然更加的简洁
        return integers.stream()
                .map(i -> String.valueOf(i * Math.PI))
                .collect(Collectors.toList());
    }
}

```



