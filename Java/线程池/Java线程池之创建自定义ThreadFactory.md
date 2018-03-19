# Java线程池之创建自定义ThreadFactory

[TOC]


工厂模式是最常用的模式之一，在创建线程的时候，我们当然也能使用工厂模式来生产Thread，这样就能替代默

认的new THread，而且在自定义工厂里面，我们能创建自定义化的Thread，并且计数，或则限制创建Thread的数量，

给每个Thread设置对应的好听的名字，或则其他的很多很多事情，总之就是很爽，下面我们来展示一个简单的Thread

工厂模式来创建自己的Thread。



```java
package cn.usr.alarm.broadcast.util;

import javafx.concurrent.Task;

import java.util.ArrayList;
import java.util.Date;
import java.util.Iterator;
import java.util.List;
import java.util.concurrent.ThreadFactory;

/**
 * @Package: cn.usr.alarm.broadcast.util
 * @Description: TODO
 * @author: Rock 【shizhiyuan@usr.cn】
 * @Date: 2018/3/19 0019 11:10
 */
public class RecorderThreadFactory implements ThreadFactory {

    private int counter;
    private String name;
    private List<String> stats;

    public RecorderThreadFactory(String name) {
        counter = 0;
        this.name = name;
        stats = new ArrayList<String>();
    }

    @Override
    public Thread newThread(Runnable run) {
        Thread t = new Thread(run, name + "-Thread-" + counter);
        counter++;
        stats.add(String.format("UsrCloud Alarm thread [%d] with name %s on%s\n" ,t.getId() ,t.getName() ,new Date()));
        return t;
    }

    public String getStas() {
        StringBuffer buffer = new StringBuffer();
        Iterator<String> it = stats.iterator();
        while(it.hasNext()) {
            buffer.append(it.next());
            buffer.append("\n");
        }
        return buffer.toString();
    }

    public static void main(String[] args) {
        RecorderThreadFactory factory = new RecorderThreadFactory("MyThreadFactory");
        Task task = new Task() {
            @Override
            protected Object call() throws Exception {
                return null;
            }
        };
        Thread thread = null;
        for(int i = 0; i < 10; i++) {
            thread = factory.newThread(task);
            thread.start();
        }
        System.out.printf("Factory stats:\n");
        System.out.printf("%s\n",factory.getStas());
    }
}


```