---
layout: post
title: Spring通过RestTemplate使用POST方式请求指定接口
date: 2022-01-19
tag: java,Spring,RestTemplate
---

考虑到java开发或者运维过程中，需要调用指定的接口，java代码样例如下：

使用POST方式调用；
```
	public static void main(String[] args) {
		RestTemplate restTemplate = new RestTemplate();
		//  解决中文乱码
		restTemplate.getMessageConverters().set(1,new StringHttpMessageConverter(StandardCharsets.UTF_8));
		// 构建请求头
		HttpHeaders requestHeaders = new HttpHeaders();
		requestHeaders.setBearerAuth("tokenString");
		requestHeaders.setContentType(MediaType.APPLICATION_JSON);
		// 入参为json体
		String reqJsonStr = "{\"code\":\"testCode\", \"group\":\"testGroup\",\"content\":\"testContent\", \"order\":1}";
		HttpEntity<String> requestEntity = new HttpEntity<>(null, requestHeaders);
		// 发起请求
		ResponseEntity<String> json =restTemplate.exchange("https://xxx1/yyy1/zzz1/postmethod?userId=12345", HttpMethod.GET, requestEntity,String.class);
		System.out.println(json);
	}
```


使用GET方式调用：
```
	public static void main(String[] args) {
		RestTemplate restTemplate = new RestTemplate();
		restTemplate.getMessageConverters().set(1,new StringHttpMessageConverter(StandardCharsets.UTF_8));
		HttpHeaders requestHeaders = new HttpHeaders();
		requestHeaders.setBearerAuth("tokenString");
		HttpEntity<String> requestEntity = new HttpEntity<>(null, requestHeaders);
		ResponseEntity<String> json =restTemplate.exchange("https://xxx2/yyy2/zzz2/gettmethod?userId=12345", HttpMethod.GET, requestEntity,String.class);
	}

```