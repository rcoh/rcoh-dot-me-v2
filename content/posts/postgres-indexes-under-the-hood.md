---
title: "Postgres Indexes Under the Hood"
date: 2018-01-21T20:32:06-08:00
draft: true
tags: ["postgres", "databases", "algorithms"]
---
More than many commonly used primitives, I find database indexes to be among the least well-understood by many software engineers. *todo fix the mouthful*. In this post, I hope to clear the haze a bit and do a deep dive into how indexes work in Postgres. 

### Indexes in Postgres
Postgres actually offers 4 different kinds of indexes for different use cases. In this post I'll be focusing on the "normal" index, the kind you get by default when you type `create index`. These indexes 

