---
layout: post
title:  "RESTful with Spring Boot"
author: "Shori"
comments: false
tags: Spring
---

### Notice: Chinese Content

This article is a walkthrough of [this](https://spring.io/guides/gs/rest-service/) spring.io guide. I feel the official spring guides are perfectly suitable for beginners. And, as the guide itself is already in English, I will write this in Chinese. Okay, without further ado, let's get started.

<br>

# Building a RESTful Web Service

那么我们会搭一个 Hello world 的 RESTful 的 web 应用。那么他的功能和[前文](https://lishpr.github.io/2021-01-25/spring-boot-1)一致，就是接受到`http://localhost:8080/greeting`的`GET`请求，然后返回一个 JSON ，
```js
{"id":1,"content":"Hello, World!"}
```
我们也可以改变这个 `GET` 请求的 payload ，比如说添加 `name` field ，即
`http://localhost:8080/greeting?name=User` ，那么我们的回复也会相应更改为
```js
{"id":1,"content":"Hello, User!"}
```
反正就是这么一个简单的东西。那么开始吧！

## 建立资源类 Resource Representation Class

我们的目标是，针对到`/greeting`的`GET`请求，回复一个`200 OK`，然后主文是一个 JSON ，包括了
```js
{"id":1,"content":"Hello, World!"}
```

那么我们为了要构建这个 JSON ，我们需要一个资源类。那么这个类就是一个简单的 POJO （字面意思：简单而令人怀旧的 Java 对象），诸如：

```java
package com.example.restservice;

public class Greeting {

	private final long id;
	private final String content;

	public Greeting(long id, String content) {
		this.id = id;
		this.content = content;
	}

	public long getId() {
		return id;
	}

	public String getContent() {
		return content;
	}
}
```

<br>

## 建立资源控制器 Resource Controller

```java
package com.example.restservice;

import java.util.concurrent.atomic.AtomicLong;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class GreetingController {

	private static final String template = "Hello, %s!";
	private final AtomicLong counter = new AtomicLong();

	@GetMapping("/greeting")
	public Greeting greeting(@RequestParam(value = "name", defaultValue = "World") String name) {
		return new Greeting(counter.incrementAndGet(), String.format(template, name));
	}
}
```

这个控制器看上去很简洁，但是有很多事情在内部发生。我们一一解释：

首先是`@GetMapping("/greeting")`，保证了我们到`/greeting`的 HTTP GET 请求会被传送到`greeting()`函数中。那么同理， Spring 中也有`@PostMapping`，以及所有 RESTful 通用的`@RequestMapping`（用例：@RequestMapping(method=GET)）。

`@RequestParam`则将请求中的`name`参数与`greeting()`函数的`name`参数绑定。同时制定了一个缺省值。

最后，我们创建一个新的`Greeting` POJO ，并返回之。注意到，具有`@RestController`的类中的函数返回的值都默认将变成 JSON 格式。

<br>

## 运行

执行`./mvnw spring-boot:run`， maven 会自动搭建并运行代码。这样我们的应用就做好啦！我们可以访问`http://localhost:8080/greeting`来看看效果。