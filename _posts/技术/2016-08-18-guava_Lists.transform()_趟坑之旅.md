---
layout: post
title: Guava Lists.transform()趟坑之旅
category: 学习
tags: study
keywords:
description:
---


# 问题描述

 下面的测试类,是对遇到问题的简化版本, 对下面的测试, 你感觉输出会是什么?

        public class GuavaListsTransformTest {

            @Test
            public void transform_test() {

                // 假如你从别的地方获取了一个数据集
                List<Data> dataList = getDatas();

                // 现在你需要对这个数据集的数据进行修改
                for (Data data : dataList) {
                    data.setKey("second key");
                }

                System.out.println(dataList);

            }

            private List<Data> getDatas() {
                List<String> keyList = Lists.newArrayList("first key");
                // 获取数据集合时,使用这种方式返回的
                return Lists.transform(keyList, new Function<String, Data>() {
                    @Override
                    public Data apply(String input) {
                        return new Data(input);
                    }
                });
            }

            private static class Data {
                private String key;

                private Data(String key) {
                    this.key = key;
                }

                public String getKey() {
                    return key;
                }

                public void setKey(String key) {
                    this.key = key;
                }

                @Override
                public String toString() {
                    return "Data{" +
                            "key='" + key + '\'' +
                            '}';
                }
            }

        }

## 结果输出

> [Data{key='first key'}]

这他喵的就见鬼了啊, 我明明就是直接对list里面的元素进行处理的, 修改了里面数据的值,为啥`dataList`中数据没有变?

在for循环中打日志,发现,没问题啊,值是被改了的啊...
>Data{key='second key'}

再到for循环外面再打日志,发现值又变回去了.

多么痛的领悟,渐渐的都开始怀疑人生了.为啥在for循环里面打出来的数据是被改了的,在for循环外面就没变?

## 造成原因

摸爬滚打,一步步搞, 发现了 guava Lists.transform() 返回的List里面的猫腻.

        @Override public ListIterator<T> listIterator(final int index) {
              return new TransformedListIterator<F, T>(fromList.listIterator(index)) {
                @Override
                T transform(F from) {
                  return function.apply(from);
                }
              };
            }

foreach语句在调用这个List的`iterrator()`时,返回的是上面这样一个迭代器,
也就是说, 每一次for循环的时候,都会去重新调用转换函数,生成一个新的对象返回(它并不会保存已转换的对象),
所以你在foreach语句中对集合里面的数据进行修改,是没什么卵用的.

## 总结

如果外围调用者可能会对返回的集合数据做修改, 慎用 guava Lists.transform() 这种 延迟转换 的方法, 不然可就是大坑一个啊 :(
