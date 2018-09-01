---
title:  "Bringing Raspberry Pi to Classroom"
last_modified_at: 2018-03-20T16:00:58-04:00
categories: 
  - Edtech
tags:
  - RaspberryPi
toc: true
---

There is so much going on in technology these days. Technology has brought unprecedented changes in our daily life, retooling the way we communicate, the way we shop, the way we make our living and more. The things that were considered as science-fiction a few years ago, is now a real thing. But, compare the classrooms back in the 90s and now, do you see any change? For the major part, it’s more or less the same. It’s a no-brainer that we have been scared to adopt new technology to our classrooms. It is said that Socrates was scared of this new technology called “writing” which he thought would erode the memorizing power of human. There was a time when people were intimidated by the use of the calculator in the class, for it may jeopardize the calculating power of the human brain. Nevertheless, we cannot overlook the fact of how technology can be leveraged to extend the knowledge imparting process, for education today is not about what-you-know, but what you can do with what-you-know. With growing MOOCs and learning materials available over the Internet, that are not only adding the new dimension in learning today, but also making the learning process more fun. Also, the students today are more adept in using technology, so taking technology out of the learning equation would be alienating the student of their abilities.

At [OLE Nepal](http://olenepal.org), we strive to bring the best new technology to our classrooms. It’s never easy to embrace a new piece of technology, for teachers are resistant to the change, with factors like power cuts and budget making up a huge share of challenge. So we set our selection of the technology based on the 3 prime constraints; low-powered, portable, and low cost. And for these traits, Raspberry Pi steps up as the knight in shining armor.

![](https://cdn-images-1.medium.com/max/800/1*xnQWux2pqa2H1qPDi-f4_w.jpeg)

Produced in Cambridge, [Raspberry Pi](https://en.wikipedia.org/wiki/Raspberry_Pi) primarily designed to demystify the technology in the classrooms for the learners, is a credit-card sized computer that costs only $35. The device plugs into a computer monitor or TV, and uses a standard keyboard and mouse. You can use this mini computer just like you would your desktop computer to do everything from browsing the Internet and playing high-definition video, to making spreadsheets, word-processing, or playing games.


## How is OLE Nepal using Raspberry Pi?

![Typical E-Pustakalaya lab setup](https://cdn-images-1.medium.com/max/800/1*_0sm75N6aVaoBRVWxfGAlQ.png)

Predominantly we build Pustakalaya Server, which is typically a mini PC, hosting educational contents major section which is the [E-Pustakalaya](http://www.olenepal.org/E-Pustakalaya/) , which is digital library of more than 7000+ books of different genre build on the FEDORA (or Flexible Extensible Digital Object Repository Architecture) digital asset management (DAM) architecture upon which institutional repositories, digital archives, and digital library systems are built. We also have added to it our home-brewed, curriculum-based interactive teaching material; [E-Paath](http://www.olenepal.org/E-Paath/), off-line Khan Academy videos, Open-street map, Nepali Sabdakosh, [PheT simulations](https://phet.colorado.edu/en/simulations/) and much more, with a regular update to books and educational content. Basically, Pustakalaya Server is the offline version of the pustakalaya.org. It’s more like bringing the Internet to your classroom.

![](https://cdn-images-1.medium.com/max/800/1*DE2qHIwaXbZspeF_4Z3s7A.png)

After successful implementation of the [XO](http://wiki.laptop.org/go/The_OLPC_Wiki) and desktops computer as the client machine, Raspberry Pi, with its amazing community, is a pertinent technology to bring in our classrooms. We have been tinkering the popular Debian-based Linux-distro, Ubuntu-Mate. With its active popular community, it is best OS to our option. We have preloaded E-Paath content into the OS itself. Since most of the activities in our E-Paath are currently flash based; flash support was an important feature to have in the OS. Debian-based OS have much better flash support, it was another reason why Ubuntu-Mate was used. We also have loaded [BalPaathmala](http://pustakalaya.org/external-content/static/balpathmala/), which is a small repository of the books, into the OS hosted on Apache web server. We have been customizing the OS to make it more educational with the interactive games. Our work so far is just a tip of an iceberg.

![Raspberry Pi Desktop](https://cdn-images-1.medium.com/max/800/1*nO4BiJ7KvfrrB_WaVLWafA.png)



![Class 7–8 EPaath interface](https://cdn-images-1.medium.com/max/800/1*0zby0zK0tdk-35nBfX0xkg.png)


Some of the features we are currently working on:

* Synchronization with Server.
* Student activity statistics collection in Server.
* Auto-running mount scripts.
* Using docker or lxc containers for the installation and upgrades.
* Awstats collection 



## Where is this being implemented?

We have initiated a pilot program at Gorakhnath Secondary School, Kirtipur,Nepal, where 18 Raspberry Pi with preloaded educational content was deployed. It was a challenging experience as it was a-first-of-its-kind of deployment for us. We had a 5-day training for the teachers about pedagogy as well as the technical aspects of using the Raspberry Pi in the classroom. The primary purpose of the training was to inculcate, amongst the teachers, a culture of referring to additional reference materials. Looking at the excitement of the teachers, its just seems that ‘direction’ and ‘training’ were what was stopping technology from getting into their classrooms. We also installed the battery backup system for an uninterrupted flux of the lab. We are regularly providing technical support for the class as this is a pilot project for us.

![Students taking EPaath Class](https://cdn-images-1.medium.com/max/800/1*vhdX4aNtKnP8Au-tOV3Kkg.jpeg)


![Raspberry Pi Lab Setup](https://cdn-images-1.medium.com/max/800/1*RoCBe-ivsq11PeuxVbo1vQ.jpeg)


Originally published in [OLE-Blog](http://blog.olenepal.org/index.php/archives/2067). If you like what you read please follow.

