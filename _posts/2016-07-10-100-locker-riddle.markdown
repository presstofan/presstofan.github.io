---
title:  "The 100 Lockers Riddle"
date:   2016-07-10 19:07:42 +0100
categories:
  - Python
tags:
  - Python
  - Ted Riddle
  - Practice Project
---

In this post, I tried to crack the 100 locker riddle using very simple python code. Again, this is a simple practice of python syntax and functions.

> Your rich, eccentric uncle just passed away, and you and your 99 nasty relatives have been invited to the reading of his will. He wanted to leave all of his money to you, but he knew that if he did, your relatives would pester you forever. Can you solve the riddle he left for you and get the inheritance? Lisa Winer shows how.

TED talk webpage can be found [here][ted-talk].

<iframe width="560" height="315" src="https://www.youtube.com/embed/c18GjbnZXMw" frameborder="0" allowfullscreen></iframe>
<br><br>
Below is the code:

```python
# create 100 closed locker (-1)
locker = [-1] * 100
heir = -1
```

```python
for n in range(1,101):
    s = locker[n-1::n]
    new_s = [x * heir for x in s]
    locker[n-1::n] = new_s
```




[ted-talk]: http://ed.ted.com/lessons/can-you-solve-the-locker-riddle-lisa-winer
