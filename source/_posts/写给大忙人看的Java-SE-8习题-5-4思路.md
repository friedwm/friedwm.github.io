---
title: 写给大忙人看的Java SE 8习题 5.4思路
date: 2019-06-26 23:09:19
categories:
tags:
---

题目要求：输入年月，按真正日历的格式打印某月日历，星期一为一周第一天。

解题思路：
很自然的想法是计算每一行的内容（包括前置的空白），然后按行输出，我觉得换算略麻烦，而且存储起来也没必要。

换个思路，只用计算出前导空白的天数（其实是上个月末尾几天），然后按顺序1~月底输出，每7天换个行就可以了。由此有了以下代码：



```java
// 辅助类，封装重复操作
public class Repeater {
  private int count;
  private Repeater(int count) {
    this.count = count;
  }

  static Repeater of(int count) {
    return new Repeater(count);
  }

  public void repeat(Consumer<Integer> consumer) {
    for (int i = 0; i < count; i++) {
      consumer.accept(i);
    }
  }
}

// 主类
public class ex4_1 {

  public static void main(String[] args) {
    printCalendar(2019, 5);
  }

  private static void printCalendar(int year, int month) {
    String[] header = {"一", "二", "三", "四", "五", "六", "日"};
    CalenderPrinter printer = new CalenderPrinter();
    // 打印头
    Arrays.stream(header).forEach(printer::print);

    LocalDate firstDay = LocalDate.of(year, month, 1);
    int dayOfWeek = firstDay.getDayOfWeek().getValue();
    
    // 输出前导的空白
    Repeater.of(dayOfWeek - 1).repeat(i -> {
      printer.print("");
    });

    // 计算当月天数
    int totalDaysOfMonth = firstDay.with(TemporalAdjusters.lastDayOfMonth()).getDayOfMonth();
    Repeater.of(totalDaysOfMonth).repeat(i -> {
      // repeater从0开始循环，因此+1
      printer.print(String.valueOf(i + 1));
    });
  }

  // 打印类，记忆打印的字符串个数，每7个输出一个换行
  private static class CalenderPrinter {
    private int printed;

    void print(String str) {
      System.out.printf("%-10s\t", str);
      ++printed;
      if (printed % 7 == 0) {
        System.out.println("");
      }
    }
  }
}
```
