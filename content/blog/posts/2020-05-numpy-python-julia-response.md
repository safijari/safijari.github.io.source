+++
title = "A Response to \"Is Numpy Faster than Python?\""
tags = ["python", "numba", "numpy", "julia"]
draft = true
+++

So I woke up this morning to my google feed recommending the article [Is NumPy Faster than Python?](https://towardsdatascience.com/is-numpy-faster-than-python-e8a7363d8276)
and the first thought through my head was "uhhh ... why is that even a question?". I later realized that
this was a follow up to the author's previous article called ["Should You Jump Python's Ship and Move to Julia?"](https://towardsdatascience.com/should-you-jump-pythons-ship-and-move-to-julia-ccd32e7d25d9).

Upon more investigation it was clear that the author is a `Julia` evangelist with seemingly very little experience
in `Python`. Now this is ... fine. I have a great deal of respect for `Julia` and the community around it and the
language has come a long way from when I first used it in grad school (as I noticed during my lengthy exploration
of the language in a past video).

{{< youtube TNoShNPoEak>}}

In face in the above video I did a comparison of how well a naive `Julia` implementation stacks up against a
naive Numba implementation of the same simulation (they were about the same). I was then [schooled by](https://discourse.julialang.org/t/how-to-optimize-the-following-code/33209) some
folks much more knowledgable about `Julia` than I was after my 4 hours session with it and they demonstrated
that with a bit of care `Julia` could become sufficiently faster than Numba. The fact that such a simple looking
language could be made so performant says a lot about its potential and I'm eagerly looking forward to what
becomes of it.

But this post is not about `Julia`.

This post is more of a response to the "Is Numpy Faster than Python" question, but perhaps more importantly
it serves as an assertion that **"Even if you wanted to, you likely cannot jump `Python`'s ship just yet, so
how about we talk about how to make the most of this unfortunately less than performant language?"**

OK that's a bit of a mouthful, sorry about that. Let's get to it.


## The base code {#the-base-code}

What follows is code taken verbatim from the aforementioned article (formatting my own,
I just ran [black](https://pypi.org/project/black/) on it). First, the pure `Python` implementation which was compared with `Julia`:

```python
def dot(x, y):
    lst = []
    for i, w in zip(x, y):
        lst.append(i * w)
    return lst


def sq(x):
    x = [c ** 2 for c in x]
    return x


class LinearRegression:
    def __init__(self, x, y):
        # a = ((∑y)(∑x^2)-(∑x)(∑xy)) / (n(∑x^2) - (∑x)^2)
        # b = (x(∑xy) - (∑x)(∑y)) / n(∑x^2) - (∑x)^2
        if len(x) != len(y):
            pass
        # Get our Summations:
        Σx = sum(x)
        Σy = sum(y)
        # dot x and y
        xy = dot(x, y)
        # ∑dot x and y
        Σxy = sum(xy)
        # dotsquare x
        x2 = sq(x)
        # ∑ dotsquare x
        Σx2 = sum(x2)
        # n = sample size
        n = len(x)
        # Calculate a
        self.a = (((Σy) * (Σx2)) - ((Σx * (Σxy)))) / ((n * (Σx2)) - (Σx ** 2))
        # Calculate b
        self.b = ((n * (Σxy)) - (Σx * Σy)) / ((n * (Σx2)) - (Σx ** 2))

    def predict(self, xt):
        xt = [self.a + (self.b * i) for i in xt]
        return xt
```

The code is meant to do `Linear Regression` and the API is kind of similar to what you might expect
from `scikit-learn` (except the initializer takes the place of the `fit` function). If I came across this code
in a review I would make the following remarks:

-   The "if not this then pass" in the initializer is problematic. Kindly replace with an assert or, if you'd prefer not to raise an exception, then let's talk more about your usecase and how you might redesign this.
-   Use `Numpy` arrays and ops rather than `Python` lists and ops.
-   There is already an implementation of this in `scikit-learn`, is there any reason why you can't use that (e.g. don't want to pull in a dependency)?

Now the author _did_ rewrite the code with `Numpy` (again, formatting my own and I have removed the comments):

```python
import numpy as np


class npLinearRegression:
    def __init__(self, x, y):
        if len(x) != len(y):
            pass
        Σx = sum(x)
        Σy = sum(y)
        xy = np.multiply(x, y)
        Σxy = sum(xy)
        x2 = np.square(x)
        Σx2 = sum(x2)
        n = len(x)
        self.a = (((Σy) * (Σx2)) - ((Σx * (Σxy)))) / ((n * (Σx2)) - (Σx ** 2))
        self.b = ((n * (Σxy)) - (Σx * Σy)) / ((n * (Σx2)) - (Σx ** 2))

    def predict(self, xt):
        xt = [self.a + (self.b * i) for i in xt]
        return xt
```

but it's a bit of an unfortunate implementation (and only gives about a 50% speedup over pure `Python` and is 5x or so slower than `Julia`). Let's
pretend this is the update to the previous code that was submitted after my pretend review of the above code.
My next review would likely be as follows:

-   You appear to ultimately still be operating over `=Python=` lists and there's probably a lot of casting back and forth between `ndarray` and `List`.
-   `np.multiply` and `np.square` aren't really needed (if `x` and `y` are passed as `numpy` arrays, that is).
-   The `predict` function is completely unchanged.

At this point I might offer the reviewee to do some pair programming with me so I can demonstrate some of the updates
this code can dearly benefit from. At the end of that review the code would likely look something like this:

```python
import numpy as np
class NpLinearRegression:
    def __init__(self, x: np.ndarray, y: np.ndarray):
        assert len(x) == len(y), "x and y arrays must have the same number of elements along the first axis"
        Σx = x.sum()
        Σy = y.sum()
        Σxy = (sx*sy).sum()
        Σx2 = (x**2).sum()
        n = len(x)

        self.a = ((Σy) * (Σx2)) - ((Σx * (Σxy))) / ((n * (Σx2)) - (Σx ** 2))
        self.b = ((n * (Σxy)) - (Σx * Σy)) / ((n * (Σx2)) - (Σx ** 2))

    def predict(self, xt: np.ndarray) -> np.ndarray:
        return self.a + (self.b * xt)
```
