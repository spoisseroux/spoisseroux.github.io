---
layout: post
title:  "[Java] CPU scheduler simulator"
date:   2020-11-03 19:48:10
---

a single core CPU process scheduler simulation created in Java from scratch

-----------------------------------------------------------

**parameter files**

![sim2](https://media.giphy.com/media/IpjjD6aBrlxlpIz1eQ/giphy.gif)

the parameter files were formatted to specify these parameters (in milliseconds):
- total simulation time
- quantum time
- context switch
- average process length
- average creation length
- IO bound process time
- average IO service time

length of time from average times were randomly generated from a standard deviation

-----------------------------------------------------------

**running params1.txt**

![sim1](https://media.giphy.com/media/erZuhUa1Rey9jYqOdj/giphy.gif)

-----------------------------------------------------------

**results of params1.txt**

![sim3](https://media.giphy.com/media/FXQXOLMK04jjzxgciq/giphy.gif)

The results show the overall statistics, as well as the statistics for CPU-bound and IO-bound processes individually


[[code]](https://github.com/spoisseroux/cs371scheduler/tree/main/cs371scheduler)

