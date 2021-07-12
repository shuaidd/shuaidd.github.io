---
title: jdk8新增特性尝鲜
date: 2021-07-12 16:46:57
tags:
   - jdk8
---

### 函数接口
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
### 方法引用
```java
package com.travel.jdk8.method;

import java.util.function.BiFunction;

/**
 * jdk8新特性  方法引用
 */
public class MethodReferencesExamples {

    public static <T> T mergeThings(T a, T b, BiFunction<T, T, T> merger) {
        return merger.apply(a, b);
    }

    public static String appendStrings(String a, String b) {
        return a + b;
    }

    public String appendStrings2(String a, String b) {
        return a + b;
    }

    public static void main(String[] args) {

        MethodReferencesExamples myApp = new MethodReferencesExamples();

        // Calling the method mergeThings with a lambda expression
        System.out.println(MethodReferencesExamples.
            mergeThings("Hello ", "World!", (a, b) -> a + b));

        // Reference to a static method
        System.out.println(MethodReferencesExamples.
            mergeThings("Hello ", "World!", MethodReferencesExamples::appendStrings));

        // Reference to an instance method of a particular object
        System.out.println(MethodReferencesExamples.
            mergeThings("Hello ", "World!", myApp::appendStrings2));

        // Reference to an instance method of an arbitrary object of a
        // particular type
        System.out.println(MethodReferencesExamples.
            mergeThings("Hello ", "World!", String::concat));
    }
}

```
### stream
```java
package com.travel.jdk8.stream;

import java.util.function.IntConsumer;

class Averager implements IntConsumer
{
    private int total = 0;
    private int count = 0;

    public double average() {
        return count > 0 ? ((double) total)/count : 0;
    }

    @Override
    public void accept(int i) { total += i; count++; }

    public void combine(Averager other) {
        total += other.total;
        count += other.count;
    }
}
```

```java
package com.travel.jdk8.stream;

import java.util.List;

public class BulkDataOperationsExamples {

    public static void main(String... args) {

        // Create sample data

        List<Person> roster = Person.createRoster();

        // 1. Print names of members, for-each loop

        System.out.println("Members of the collection (for-each loop):");
        for (Person p : roster) {
            System.out.println(p.getName());
        }

        // 2. Print names of members, forEach operation

        System.out.println("Members of the collection (bulk data operations):");
        roster
                .stream()
                .forEach(e -> System.out.println(e.getName()));

        // 3. Print names of male members, forEach operation

        System.out.println(
                "Male members of the collection (bulk data operations):");
        roster
                .stream()
                .filter(e -> e.getGender() == Person.Sex.MALE)
                .forEach(e -> System.out.println(e.getName()));

        // 4. Print names of male members, for-each loop

        System.out.println("Male members of the collection (for-each loop):");
        for (Person p : roster) {
            if (p.getGender() == Person.Sex.MALE) {
                System.out.println(p.getName());
            }
        }

        // 5. Get average age of male members of the collection:

        double average = roster
                .stream()
                .filter(p -> p.getGender() == Person.Sex.MALE)
                .sorted()
                .mapToInt(Person::getAge)
                .average()
                .getAsDouble();

        System.out.println(
                "Average age of male members (bulk data operations): " +
                        average);
    }
}
```

```java
package com.travel.jdk8.stream;

import java.util.List;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.function.Consumer;
import java.util.function.IntConsumer;
import java.util.function.Function;
import java.util.function.BinaryOperator;
import java.util.Comparator;
import java.util.function.UnaryOperator;
import java.util.function.Predicate;
import java.util.GregorianCalendar;
import java.util.Collection;
import java.util.Collections;
import java.lang.Iterable;
import java.util.function.Supplier;
import java.util.Set;
import java.util.Map;
import java.util.HashSet;
import java.util.HashMap;
import java.util.List;
import java.util.ArrayList;
import java.util.stream.Collector;
import java.util.stream.Collectors;
import java.util.stream.IntStream;
import java.time.chrono.IsoChronology;
import java.lang.Number;
import java.util.stream.*;
import java.util.Optional;
import java.util.OptionalInt;
import java.util.concurrent.ConcurrentMap;

public class ParallelismExamples {

    public static void main(String... args) {

        // Create sample data

        List<Person> roster = Person.createRoster();

        System.out.println("Contents of roster:");
        roster
            .stream()
            .forEach(p -> p.printPerson());
        System.out.println();

        // 1. Average age of male members in parallel

        double average = roster
            .parallelStream()
            .filter(p -> p.getGender() == Person.Sex.MALE)
            .mapToInt(Person::getAge)
            .average()
            .getAsDouble();

        System.out.println("Average age of male members in parallel: " +
            average);

        // 2. Concurrent reduction example

        ConcurrentMap<Person.Sex, List<Person>>
            byGenderParallel =
            roster
                .parallelStream()
                .collect(Collectors.groupingByConcurrent(Person::getGender));

        List<Map.Entry<Person.Sex, List<Person>>>
            byGenderList =
            new ArrayList<>(byGenderParallel.entrySet());

        System.out.println("Group members by gender:");
        byGenderList
            .stream()
            .forEach(e -> {
                System.out.println("Gender: " + e.getKey());
                e.getValue()
                    .stream()
                    .map(Person::getName)
                    .forEach(f -> System.out.println(f)); });

        // 3. Examples of ordering and parallelism

        System.out.println("Examples of ordering and parallelism:");
        Integer[] intArray = {1, 2, 3, 4, 5, 6, 7, 8 };
        List<Integer> listOfIntegers =
            new ArrayList<>(Arrays.asList(intArray));

        System.out.println("listOfIntegers:");
        listOfIntegers
            .stream()
            .forEach(e -> System.out.print(e + " "));
        System.out.println("");

        System.out.println("listOfIntegers sorted in reverse order:");
        Comparator<Integer> normal = Integer::compare;
        Comparator<Integer> reversed = normal.reversed();
        Collections.sort(listOfIntegers, reversed);
        listOfIntegers
            .stream()
            .forEach(e -> System.out.print(e + " "));
        System.out.println("");

        System.out.println("Parallel stream");
        listOfIntegers
            .parallelStream()
            .forEach(e -> System.out.print(e + " "));
        System.out.println("");

        System.out.println("Another parallel stream:");
        listOfIntegers
            .parallelStream()
            .forEach(e -> System.out.print(e + " "));
        System.out.println("");

        System.out.println("With forEachOrdered:");
        listOfIntegers
            .parallelStream()
            .forEachOrdered(e -> System.out.print(e + " "));
        System.out.println("");

        // 4. Example of interference

        try {
            List<String> listOfStrings =
                new ArrayList<>(Arrays.asList("one", "two"));

            // This will fail as the peek operation will attempt to add the
            // string "three" to the source after the terminal operation has
            // commenced.

            String concatenatedString = listOfStrings
                .stream()

                // Don't do this! Interference occurs here.
                .peek(s -> listOfStrings.add("three"))

                .reduce((a, b) -> a + " " + b)
                .get();

            System.out.println("Concatenated string: " + concatenatedString);

        } catch (Exception e) {
            System.out.println("Exception caught: " + e.toString());
        }

        // 5. Stateful lambda expressions examples

        List<Integer> serialStorage = new ArrayList<>();

        System.out.println("Serial stream:");
        listOfIntegers
            .stream()

            // Don't do this! It uses a stateful lambda expression.
            .map(e -> { serialStorage.add(e); return e; })

            .forEachOrdered(e -> System.out.print(e + " "));
        System.out.println("");

        serialStorage
            .stream()
            .forEachOrdered(e -> System.out.print(e + " "));
        System.out.println("");

        System.out.println("Parallel stream:");
        List<Integer> parallelStorage = Collections.synchronizedList(
            new ArrayList<>());
        listOfIntegers
            .parallelStream()

            // Don't do this! It uses a stateful lambda expression.
            .map(e -> { parallelStorage.add(e); return e; })

            .forEachOrdered(e -> System.out.print(e + " "));
        System.out.println("");

        parallelStorage
            .stream()
            .forEachOrdered(e -> System.out.print(e + " "));
        System.out.println("");
    }
}
```

```java
package com.travel.jdk8.stream;

import java.util.List;
import java.util.ArrayList;
import java.time.chrono.IsoChronology;
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;
import java.time.Period;

public class Person {

    public enum Sex {
        MALE, FEMALE
    }

    String name;
    LocalDate birthday;
    Sex gender;
    String emailAddress;

    Person(String nameArg, LocalDate birthdayArg,
        Sex genderArg, String emailArg) {
        name = nameArg;
        birthday = birthdayArg;
        gender = genderArg;
        emailAddress = emailArg;
    }

    public int getAge() {
        return birthday
            .until(IsoChronology.INSTANCE.dateNow())
            .getYears();
    }

    public void printPerson() {
      System.out.println(name + ", " + this.getAge());
    }

    public Sex getGender() {
        return gender;
    }

    public String getName() {
        return name;
    }

    public String getEmailAddress() {
        return emailAddress;
    }

    public LocalDate getBirthday() {
        return birthday;
    }

    public static int compareByAge(Person a, Person b) {
        return a.birthday.compareTo(b.birthday);
    }

    public static List<Person> createRoster() {

        List<Person> roster = new ArrayList<>();
        roster.add(
            new Person(
            "Fred",
            IsoChronology.INSTANCE.date(1980, 6, 20),
            Person.Sex.MALE,
            "fred@example.com"));
        roster.add(
            new Person(
            "Jane",
            IsoChronology.INSTANCE.date(1990, 7, 15),
            Person.Sex.FEMALE, "jane@example.com"));
        roster.add(
            new Person(
            "George",
            IsoChronology.INSTANCE.date(1991, 8, 13),
            Person.Sex.MALE, "george@example.com"));
        roster.add(
            new Person(
            "Bob",
            IsoChronology.INSTANCE.date(2000, 9, 12),
            Person.Sex.MALE, "bob@example.com"));

        return roster;
    }

}
```

```java
package com.travel.jdk8.stream;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class ReductionExamples {

    public static void main(String... args) {

        // Create sample data

        List<Person> roster = Person.createRoster();

        System.out.println("Contents of roster:");

        roster
            .stream()
            .forEach(p -> p.printPerson());

        System.out.println();

        // 1. Average age of male members, average operation

        double average = roster
            .stream()
            .filter(p -> p.getGender() == Person.Sex.MALE)
            .mapToInt(Person::getAge)
            .average()
            .getAsDouble();

        System.out.println("Average age (bulk data operations): " +
            average);

        // 2. Sum of ages with sum operation

        Integer totalAge = roster
            .stream()
            .mapToInt(Person::getAge)
            .sum();

        System.out.println("Sum of ages (sum operation): " +
            totalAge);

        // 3. Sum of ages with reduce(identity, accumulator)

        Integer totalAgeReduce = roster
            .stream()
            .map(Person::getAge)
            .reduce(
                0,
                (a, b) -> a + b);

        System.out.println(
            "Sum of ages with reduce(identity, accumulator): " +
            totalAgeReduce);

        // 4. Average of male members with collect operation

        Averager averageCollect = roster.stream()
            .filter(p -> p.getGender() == Person.Sex.MALE)
            .map(Person::getAge)
            .collect(Averager::new, Averager::accept, Averager::combine);

        System.out.println("Average age of male members: " +
            averageCollect.average());

        // 5. Names of male members with collect operation

        System.out.println("Names of male members with collect operation: ");
        List<String> namesOfMaleMembersCollect = roster
            .stream()
            .filter(p -> p.getGender() == Person.Sex.MALE)
            .map(p -> p.getName())
            .collect(Collectors.toList());

        namesOfMaleMembersCollect
            .stream()
            .forEach(p -> System.out.println(p));

        // 6. Group members by gender

        System.out.println("Members by gender:");
        Map<Person.Sex, List<Person>> byGender =
            roster
                .stream()
                .collect(
                    Collectors.groupingBy(Person::getGender));

        List<Map.Entry<Person.Sex, List<Person>>>
            byGenderList =
            new ArrayList<>(byGender.entrySet());

        byGenderList
            .stream()
            .forEach(e -> {
                System.out.println("Gender: " + e.getKey());
                e.getValue()
                    .stream()
                    .map(Person::getName)
                    .forEach(f -> System.out.println(f)); });

        // 7. Group names by gender

        System.out.println("Names by gender:");
        Map<Person.Sex, List<String>> namesByGender =
            roster
                .stream()
                .collect(
                     Collectors.groupingBy(
                         Person::getGender,
                         Collectors.mapping(
                             Person::getName,
                             Collectors.toList())));

        List<Map.Entry<Person.Sex, List<String>>>
            namesByGenderList =
                new ArrayList<>(namesByGender.entrySet());

        namesByGenderList
            .stream()
            .forEach(e -> {
                System.out.println("Gender: " + e.getKey());
                e.getValue()
                    .stream()
                    .forEach(f -> System.out.println(f)); });

        // 8. Total age by gender

        System.out.println("Total age by gender:");
        Map<Person.Sex, Integer> totalAgeByGender =
            roster
                .stream()
                .collect(
                     Collectors.groupingBy(
                         Person::getGender,
                         Collectors.reducing(
                             0,
                             Person::getAge,
                             Integer::sum)));

        List<Map.Entry<Person.Sex, Integer>>
            totalAgeByGenderList =
            new ArrayList<>(totalAgeByGender.entrySet());

        totalAgeByGenderList
            .stream()
            .forEach(e ->
                System.out.println("Gender: " + e.getKey() +
                    ", Total Age: " + e.getValue()));

        // 9. Average age by gender

        System.out.println("Average age by gender:");
        Map<Person.Sex, Double> averageAgeByGender =
            roster
                .stream()
                .collect(
                     Collectors.groupingBy(
                         Person::getGender,
                         Collectors.averagingInt(Person::getAge)));

        for (Map.Entry<Person.Sex, Double> e : averageAgeByGender.entrySet()) {
            System.out.println(e.getKey() + ": " + e.getValue());
        }
    }
}
```
