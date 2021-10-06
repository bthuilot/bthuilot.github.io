---
layout: post
title:  "mandelbrot-in-haskell"
---

## Building a Mandelbrot generator in Haskell


```haskell
fib :: Integer -> Integer
fib 0 = 1
fib 1 = 1
fib n = 1 + fib(n - 1) + fib (n - 2)
```


```python
def fib(n: Int):
    if n < 2:
        return 0
    return n + fib(n - 1) + fib(n - 2)
```