---
title: jdk8新增特性尝鲜
date: 2021-07-12 16:46:57
tags:
   - jdk8
---
```java
package com.travel.jdk8.function;

import java.util.Comparator;
import java.util.function.*;

/**
 * 描述 函数接口
 *
 * @author ddshuai
 * @date 2021-05-14 09:18
 **/
public class FunctionExamples {

    /**
     * BiConsumer 函数接口 没有返回值
     *
     * @param a
     * @param b
     * @param biConsumer
     */
    public static void doAccept(String a, int b, BiConsumer<String, Integer> biConsumer) {
        biConsumer.accept(a, b);
    }

    /**
     * BiFunction 两个参数 带返回值
     *
     * @param a
     * @param b
     * @param biFunction
     * @return
     */
    public static String biFunction(String a, String b, BiFunction<String, String, String> biFunction) {
        return biFunction.andThen(a1 -> {
            System.out.println("进来了");
            return a1;
        }).apply(a, b);
    }

    public static void binaryOperator(BinaryOperator<Integer> operator) {
        Integer max = operator.apply(1, 2);
        System.out.println(max);
    }

    public static Comparator<? super Integer> comparator() {
        return (a, b) -> a > b ? 1 : (a.equals(b) ? 0 : -1);
    }

    public static void biPredicate(String a, String b, BiPredicate<String, String> biPredicate) {
        boolean ab = biPredicate.and((a1, b1) -> {
            System.out.println("and 操作");
            return true;
        }).negate().test(a, b);
        System.out.println("biPredicate: " + ab);
    }

    public static boolean booleanSupplier(BooleanSupplier booleanSupplier) {
        return booleanSupplier.getAsBoolean();
    }

    public static void consumer(Consumer<? super Object> consumer,Object t){
        consumer.andThen((t1)->{
            System.out.println("再执行后置逻辑");
        }).accept(t);
    }

    public static double doubleBinaryOperator(double left,double right,DoubleBinaryOperator doubleBinaryOperator){
        return doubleBinaryOperator.applyAsDouble(left,right);
    }

    public static void main(String[] args) {

        /*BiConsumer*/
        doAccept("a", 1, (a, b) -> System.out.println(a + b));

        /*BiFunction*/
        System.out.println(biFunction("a", "b", (a, b) -> a + b));

        /*BinaryOperator*/
        binaryOperator(BinaryOperator.maxBy(comparator()));
        binaryOperator(BinaryOperator.minBy(comparator()));

        /*BiPredicate*/
        biPredicate("1", "w", (a, b) -> {
            System.out.println("第一层判断");
            return true;
        });

        /*BooleanSupplier*/
        System.out.println(booleanSupplier(()-> false));

        /*Consumer*/
        consumer((t-> System.out.println("先执行业务自定义逻辑")),"ssssss");

        /*DoubleBinaryOperator*/
        System.out.println(doubleBinaryOperator(0.01,0.1, Double::sum));
    }
}

```
