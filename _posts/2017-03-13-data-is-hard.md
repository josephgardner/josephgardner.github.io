---
layout: post
title: The Hardest Part About Microservices is your Data
comments: true
---
> Of the reasons we attempt a microservices architecture, chief among them is allowing your teams to [...] be autonomous, capable of making decisions about how to best implement and operate their services, and free to make changes as quickly as the business may desire. 

> To gain this autonomy, [...] don’t share a single database across services because then you run into conflicts like competing read/write patterns, data-model conflicts, coordination challenges, etc. But a single database does afford us a lot of safeties and conveniences: ACID transactions, single place to look, well understood (kinda?), one place to manage, etc. So when building microservices how do we reconcile these safeties with splitting up our database into multiple smaller databases?

[http://blog.christianposta.com/microservices/the-hardest-part-about-microservices-data/](http://blog.christianposta.com/microservices/the-hardest-part-about-microservices-data/)