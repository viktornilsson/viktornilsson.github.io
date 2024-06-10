---
layout: post
title: "Mastering Caching Layers"
categories: cache
---

## Introduction
In this tutorial, we'll explore different caching strategies to improve the performance of your web application. Caching is a crucial technique for reducing server load, decreasing response times, and enhancing the overall user experience. We'll cover three main caching layers: Browser caching, CDN caching, and Server caching.

## Prerequisites
- Basic knowledge of web development.
- A frontend project.
- A CDN provider (Cloudflare, Fastly etc).

## Steps

### 1. Browser Caching
Browser caching is the first line of defense for improving performance. By caching static assets like CSS, JavaScript, and images on the client-side, we can significantly reduce the number of requests to the server and improve page load times.

### 2. CDN Caching
A Content Delivery Network (CDN) can significantly improve performance by caching your application's content closer to your users. A CDN acts as a reverse proxy, caching your application's responses and serving them directly to users from its global edge network.

*Response headers with insights from Fastly*
![Cache Hit](/images/cache_hit.png)

### 3. Server Caching
While browser caching and CDN caching handle static assets and HTML responses, server caching can significantly improve the performance of dynamic content and API responses. You can use in-memory caching solutions like Redis or Memcached as a shared cache across multiple server instances.

This is also the go-to solution for personalized data that needs to be behind authorization.


### 4. Orchestration
Using these principals standalone is pretty straight forward. But when combining them things usually starts to get a bit more complex.

![Cache flow](/images/cache_flow.png)

#### 4.1 Product Data
A real world scenario could be fetching product data including real-time stock levels. To be able to handle this we need to think about all layers.

1. Don't cache it in the browser, because we are not able to invalidate or purge cache browser content. Once cached it will be there until as long as the max-age property where set to.
Fastly has a way to achieve this, https://www.fastly.com/documentation/reference/http/http-headers/Surrogate-Control/.

2. We want to cache it either way in the CDN, but we want to be able to purge/invalidate the object.
This can be done with surrogate-keys in Fastly. You tag your content, and then you can purge those tags from your backend applications. https://www.fastly.com/documentation/reference/http/http-headers/Surrogate-Key/

For example you can set:

```
Cache-Control: public, max-age=86400
Surrogate-Key: product-62952
```

And then purge the key "product-62952" when your stock import has been run.

#### 4.2 Private Data
Another example is private data, could be data like address, wishlist, preferences etc.
This data we absolutely don't want to store in the browser or in the CDN.

If your for example has some endpoint similar to this, with Authentication on the server side.
`api/customer/adress?john.doe@gmail.com`
Only the authorized applications will be able to access this content.

But if you then let the CDN cache that URL, everyone will be able to access it. Which is pretty bad.
The CDN will never handle the authorization logic by default.

*Don't cache it in Fastly/Browser*
```
Cache-Control: private, no-cache, no-store, must-revalidate, max-age=0
```

So be careful with the headers you use on this kind of data. And if you need to cache it, then Server Caching is the way to go.

#### 4.3 Unique data per URL
To make it as easy as possible for yourselves, try to have totally unique data per url.
What I mean be that is to not have clear interfaces, the input data needed comes via the url, like query parameters etc. 

We don't use hidden cookies, special logic with body parameters or something that can impact the uniqueness of the content returned.

This will make everything so much easier.

If you request `pdp/product/shirt-62952`, you know the response will be exactly the same every time.

#### GraphQL
One pretty well used example where this gets more tricky is the usage of GraphQL. On GraphQL requests the url is always the same, the request parameters comes in the body instead. So to be able to cache this you need to do custom stuff to get a unique identifier from the body request. 


### 5. Monitoring
To be able to understand what's cached and not it's almost necessary to monitor your requests/responses somehow.
Fastly for example has built in support for this. https://docs.fastly.com/en/guides/logging-endpoints

![Fastly logs](/images/cache_logs.png)

### 6. Conclusion
In this tutorial, we covered three main caching layers: Browser caching, CDN caching, and Server caching. By implementing these caching strategies, you can significantly improve the performance of your web application, reduce server load, and enhance the overall user experience.

Remember to monitor your application's performance and adjust caching configurations as needed. Additionally, consider implementing cache invalidation strategies to ensure that cached content remains up-to-date when changes occur.