---
layout: post
title: Representing State in Rest
comments: true
---

> Representing state is a complex thing. [...] When you ask the clients to infer state from the fields alone, they often infer things differently. 

> We tell the clients that if there is no paid date, then it has not been paid, and the same logic applies for sent. [...] Inferring states from dates or other arbitrary fields is awful, and it always goes wrong. 

> The data can be stored like this in the database, but exposing it in the contract is begging for trouble, as clients will always end up with a slightly different picture of the current state.

> Hypermedia As The Engine Of Application State" is [...] what makes a REST API so powerful [...] as a collection of affordances (potential actions that can [be] taken).

[https://philsturgeon.uk/api/2017/06/19/representing-state-in-rest-and-graphql](https://philsturgeon.uk/api/2017/06/19/representing-state-in-rest-and-graphql)
