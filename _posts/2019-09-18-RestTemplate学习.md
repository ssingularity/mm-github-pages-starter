---
title: "RestTemplate学习"
date: 2019-09-18
categories:
  - 工具使用
tags:
  - 工具使用
  - RestTemplate
header:
  teaser: /assets/images/RestTemplate.jpg
---
![image](/assets/images/RestTemplate.jpg)
在RestTemplate的使用中，我们尽量使用exchange方法，因为通过它可以完成GET，POST所有动作，并把接口统一化了

在使用RestTemplate的过程中，我们需要先定义一个HttpHeaders，再定义一个类型为E的payload最后使用HttpHeader以及payload生成HttpEntity<E>

其中需要注意的是，如果我们没有设置HttpHeader中的Content-Type那么默认会是application/x-www-form-urlencoded，这时候的payload只能是MultiValueMap类型，并且会被@RequestParam接收

如果想要使用@RequestBody接收的话，需要在HttpHeader中定义HttpHeaders.CONTENT_TYPE(CONTENT_TYPE)为application/json，这个时候就可以使用任意类

如果既想要接收@RequestBody又想要接收@RequestParam，那么一个方法就是直接把RequestParam放在url?后面

在使用RestTemplate.exchange(URL, POST, HttpEntity, XXX.class)之后，我们就可以得到一个Response<XXX>的返回并且可以通过它拿到Headers以及Data
构建parameter:
- 可以使用UriComponentsBuilder方便地把Paramter放到Uri中

- 可以使用参数化uri，比如：
``` 
restTemplate.exchange("http://my-rest-url.org/rest/account/{account}?name={name}",
    HttpMethod.GET,
    httpEntity,
    clazz,
    "my-account",
    "my-name"
);
```
后面的Object...或者替换成map也可以

同时可以使用HttpEntity<?> entity = new HttpEntity<>(header)构建没有body的HttpEntity

在使用RestTemplate的时候需要注意一点：
- RestTemplate是同步阻塞的，如果想要异步非阻塞的则考虑WebClient
