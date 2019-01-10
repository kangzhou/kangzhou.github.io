---
title: 算法：HashMap排序
date: 2016-06-30
categories: "Java"
tags: "算法"
---
![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1508492533211&di=de3abd21f5b11532b18ebae729c43827&imgtype=0&src=http%3A%2F%2Fimg.mp.itc.cn%2Fupload%2F20160914%2F38b0c776230244d7a7b7431c85bde0ac_th.jpg)
# 概述
面试的时候遇到过这样一道算法题，已知HashMap map,User类中有int age，String name属性。请根据User中的age进行降序排序。我们知道HashMap是没有顺序的，这里应该怎么处理呢？回来后想想应该用LinkedHashMap，LinkedHashMap是有顺序的而且是继承HashMap。
<!-- more -->

# 具体实现
```java
public class TestMap {
	public static void main(String[] args) {
		TestMap t = new TestMap();
		HashMap<Integer, User> hash = new HashMap<>();
		//根据User的年龄进行排序
		hash.put(1, new User(22, "张三"));
		hash.put(2, new User(19, "李四"));
		hash.put(3, new User(23, "王五"));
		hash.put(4, new User(18, "Tom"));
		
		HashMap<Integer, User> ha = t.Test(hash);
		System.out.println("-----------"+ha);
	}
	
	private HashMap<Integer, User> Test(HashMap<Integer, User> hash){
		Set<Entry<Integer, User>> en = hash.entrySet();
		List<Entry<Integer, User>> list = new ArrayList<>(en);
		//把排序好的放进List里面
		Collections.sort(list, new Comparator<Entry<Integer, User>>() {
			@Override
			public int compare(Entry<Integer, User> o1, Entry<Integer, User> o2) {
				
				return o1.getValue().getAge()-o2.getValue().getAge();
//				return o1.getKey()-o2.getKey();
			}
			
		});
		//再把排序好的LIst遍历到LinkedHashMap
		LinkedHashMap<Integer, User> lm = new LinkedHashMap<>();
		
		for (Entry<Integer, User> entry : list) {
			
			lm.put(entry.getKey(),entry.getValue());
		}
		return lm;
	}
	
	static class User{
		
		public User(int age, String name) {
			this.age = age;
			this.name = name;
		}
		int age;
		String name;
		public int getAge() {
			return age;
		}
		public void setAge(int age) {
			this.age = age;
		}
		public String getName() {
			return name;
		}
		public void setName(String name) {
			this.name = name;
		}
	}
}
```
