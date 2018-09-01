---
title:  "Let’s boost the web:Does speed of your website really matter?"
last_modified_at: 2018-03-20T16:00:58-04:00
categories: 
  - web
tags:
  - Web Developemt
  - Javascript
toc: true
toc_label: "Methods"
---

It’s a no-brainer: slow web performance torments every internet user. Before I dig deep into unveiling the bottlenecks of your web performance, it’s really important to estimate the cost of your performance, and there is no better way to articulate the cost of performance better than the old story of Amazon reporting increased revenue of 1% for every 100 milliseconds improvement to their site speed. source: [Amazon](http://sites.google.com/site/glinden/Home/StanfordDataMining.2006-11-28.ppt).

![](/assets/images/boostweb/giphy.gif)

From the user perspective, see how facebook’s instant article intrigues you to click the article more reflexively than any other article taking ages to load. Though there is the cost for performance enhancement; development cost but it won’t cost you a fortune to assimilate trivial practices which mitigate web load time of your website.

## 1. Reduce HTTP requests

Primarily every web pages comprise of the images, stylesheets, flash ,scripts etc. For each of these components HTTP requests are made, so adding more number of images and scripts prolongs the avail of the contents. These round trips are the main reason for slow web pages.Most developers choose to add images instead of the CSS (though sometimes it may not be feasible) but this may add much more load for the web server and web performance may be hindered. Choosing images against CSS can stack you much of the performance tax and from the user perspective, faster web is much delightful than sluggish but attractive designs for user experience (UX) does matters. 
 There are multiple ways we can reduce this 

* [Sprites](https://css-tricks.com/css-sprites/) are the cool way to reduce image numbers.
* Combine multiple style sheets into one. 
* Reduce scripts and put them at the bottom of the page.

## 2. Progressive rendering

Though lightening speed would be great but till we reach there giving something for the user so that they can get busy, can help remove the anxiety of sluggishness. For this progressive rendering is the killer ( but under appreciated ) weapon.


![](https://cdn-images-1.medium.com/max/800/1*wkTkbzLrrrhy1MNtW9xulA.png)



Though the page may be heavier but putting something for them on the plate may trick them to think page as faster.Amazon does an excellent job with progressive rendering. Who knows, had they focused on mere fast loading they could have been out of business.

![](https://cdn-images-1.medium.com/max/800/1*PwIgDTWHCeHbdiOUbQ0qSA.jpeg)

## 3. Add a Load Balancer and Reverse Proxy Server

Load balancing is a technique aiming at distributing workload in a computer network, in order to optimally utilize resources, avoid overload and maximize throughput. Instead of adding multiple tasks to a single server we add a load balancer server to distribute the load across multiple servers. Even if your application is written poorly load balancer can enhance the user experience.

A load balancer server is simply a reverse proxy server; it receives the internet traffic and forwards the requests to another server. Load balancer acts as the relay station to route the traffic.The trick is load balancer supports two or more application server with the proper choice of an algorithm the traffic can be routed to the next server. Most popularly, round-robin approach is used in which the load balancer forward the traffic to next server in the list. Load balancers can lead to much better improvements in performance because they prevent one server from being overloaded while other servers wait for traffic.

![](https://cdn-images-1.medium.com/max/800/1*5oMXVu17JA7oEs11YJgWXA.png)

Popular web servers like Apache and NGINX has inbuilt load balancing capabilities. A brief tutorial on Load Balancing can be found [here](http://websistent.com/configure-apache-web-server-load-balancing/).

## 4. Caching

As mentioned earlier too much of HTTP requests costs the web performance; hence, it would enhance the performance if we could store the most frequently used contents in a way that it could be easily be accessed. Apache and NGINX come with sophisticated and scalable caching mechanism. [Caching](https://www.digitalocean.com/community/tutorials/how-to-configure-apache-content-caching-on-ubuntu-14-04) can be broadly divided into three categories which can be broken down as:

* File Caching: The most basic caching strategy, this simply opens files or file descriptors when the server starts and keeps them available to speed up access.
* Key-Value Caching: Mainly used for SSL and authentication caching, key-value caching uses a shared object model that can store items which are costly to compute repeatedly.
* Standard HTTP caching: The most flexible and generally useful caching mechanism, this three-state system can store responses and validate them when they expire. This can be configured for performance or flexibility depending on your specific needs.


At browser level when someone visits your website for the first time they need to download the web contents but caching enhances the subsequent browsing for it cuts off HTTP requests . As [Tenni Theurer](http://yuiblog.com/blog/2007/01/04/performance-research-part-2/), formerly of Yahoo, explains it…


> The first time someone comes to your website, they have to download the HTML document, stylesheets, javascript files and images before being able to use your page. That may be as many as 30 components and 2.4 seconds.


![](https://cdn-images-1.medium.com/max/800/1*Mc5doNxCamCwNCCjRlxX1Q.png)

> Once the page has been loaded and the different components stored in the user’s cache, only a few components needs to be downloaded for subsequent visits. In Theurer’s test, that was just three components and .9 seconds, which shaved nearly 2 seconds off the load time.

![](https://cdn-images-1.medium.com/max/800/1*VAqawvqJjJ6f8st_R2m-Aw.png)

## 5. Reduce DNS Lookups

To put in laymen’s term DNS is server responsible for converting the domain main into its corresponding IP address like converting google.com to 120.89.97.42. DNS lookup takes place even before the web page loads so the size of the page do not affect the load time.Typically you can use your ISP’s DNS server but however other options as OpenDNS and Google’s DNS is also available; you can test and switch as suitable for you.

## 6. Optimize Images and Minify resources

Though adding images to web pages are not encouraged but many times its really needed to. So image should be compressed to best possible size. There are variety of image compressing sites as [ImageOptim](https://imageoptim.com/), [PNG Gauntlet](https://pnggauntlet.com/), [JPEG Optimizer](http://jpeg-optimizer.com/) and [TinyPNG](https://tinypng.com/). Images should be sized correctly. That is, if you are displaying a 200×200 image, then don’t display a 400×400 at 200×200 size. There is no point in sending a bigger image if it is not going to be displayed, or if it’s not going to fit on a device’s screen.

Minification is the process of reducing file size by removing unnecessary characters such as whitespace, new line characters, comments. The unnecessary characters call for unnecessary HTTP requests so minification is very important . In many cases file sizes can drop by significant level through minification, thus speeding up content delivery and processing. You should minify all your JavaScript, CSS, and HTML files. There are many minification tools to try for different file types, for example, these for HTML:www.willpeavy.com/minifier and kangax.github.io/html-minifier.

## What else can you do ?

I run a simple test on WebPage test with pustakalaya.org to get the result as :

![](https://cdn-images-1.medium.com/max/800/1*Ay5GhA_iIG8CRE5fBBvBLA.png)

As you can see in the figure the span in green as Time to First Byte (TTFB) takes up a significant part of the server response. That may have hit your nerve, so it’s very important to minimize this time. This time, varies according to your location from the server for it represents the route of the traffic. Particularly Waiting (TTFB) represents a number of things, like:

* Server response time
* Time to transfer bytes (a.k.a latency of the round-trip)


The Internet can be pretty fast depending on the location and network , you are using, which is dictated by the distance the packets have to travel. According to [Ilya Grigorik](http://chimera.labs.oreilly.com/books/1230000000545/ch01.html#PROPAGATION_LATENCY), we’re already quite close to the speed of light so we shouldn’t expect more speed there unless we bend physics. The way to speed things up, then, is to shorten distances. Depending on the user you have to locate the server if a bulk of the users is from Nepal than it wouldn’t be wise to place your server in United states.

So the bottom line its hard to focus on these things with approaching deadline and many bugs to fix but in the long run these practices can definitely boost your result.







