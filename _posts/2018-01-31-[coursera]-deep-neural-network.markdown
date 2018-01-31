---
layout: post
title: "[Coursera] Deep neural network - Week 2"
use_math: true
---

## Minibatch gradient descent

On batch size:
- Choosing too small batch size, you lose advantage of vectorization, cost is going to be noisy. In the extreme case where batch size = 1, it's call stochastic gradient descent
- Choosing too large batch size, the iteration takes too long. However if dataset is small, it's best just to use batch gradient descent
- Choosing something in between, it's the best of both world. Typical mini batch size ranges from 64, 128, 256, ... (in general power of 2 works best)

### Exponential Weighted Averages

A simple example with temperature in London as in the lecture:

$$ V_n = \beta * V_{n-1} + \alpha * x_n $$

where \\(x_n\\) is the temperature reading of today
