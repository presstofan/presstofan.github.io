---
title:  "The Bridge Riddle"
date:   2016-07-10 10:07:42 +0100
categories:
  - Python
tags:
  - python
  - ted-riddle
---

In this post, I tried to crack the bridge riddle using very simple python code.

> Taking that internship in a remote mountain lab might not have been the best idea. Pulling that lever with the skull symbol just to see what it did probably wasn’t so smart either. But now is not the time for regrets because you need to get away from these mutant zombies...fast. Can you use math to get you and your friends over the bridge before the zombies arrive?

TED talk webpage can be found [here][ted-talk].

<iframe width="560" height="315" src="https://www.youtube.com/embed/7yDmGnA8Hw0" frameborder="0" allowfullscreen></iframe>
<br><br>
Below is the code:

```python
import random

# A function that randomly pick K object from a list of unknown size
def random_subset( iterator, K ):
    result = []
    N = 0

    for item in iterator:
        N += 1
        if len( result ) < K:
            result.append( item )
        else:
            s = int(random.random() * N)
            if s < K:
                result[ s ] = item

    return result
```

```python
# set pars
t_dict = dict([("lab", 2),
              ("jan", 5),
              ("you", 1),
              ("prof", 10)])
t = 100
```

```python
while t > t_dead:
    set_danger = ["lab", "jan", "you", "prof"]
    set_safe =[]

    c1 = random_subset(set_danger, 2)
    t1 = max(t_dict[c1[0]],t_dict[c1[1]])
    set_danger = list(set(set_danger) - set(c1))
    set_safe = c1

    c2 = random_subset(set_safe, 1)
    t2 = t_dict[c2[0]]
    set_danger = set_danger + c2
    set_safe = list(set(set_safe) - set(c2))

    c3 = random_subset(set_danger, 2)
    t3 = max(t_dict[c3[0]],t_dict[c3[1]])
    set_danger = list(set(set_danger) - set(c3))
    set_safe = set_safe + c3

    c4 = random_subset(set_safe, 1)
    t4 = t_dict[c4[0]]
    set_danger = set_danger + c4
    set_safe = list(set(set_safe) - set(c4))

    c5 = set_danger
    t5 = max(t_dict[c5[0]],t_dict[c5[1]])

    t = t1 + t2 + t3 + t4 + t5

    if t <= t_dead:
        print "You survived!"
        print "Solution: c1:%s, c2:%s, c3:%s, c4:%s, c5:%s" % (c1, c2, c3, c4, c5)
        print "Total Time: %d" % t
    else: continue
```




[ted-talk]: https://ed.ted.com/lessons/can-you-solve-the-bridge-riddle-alex-gendler
