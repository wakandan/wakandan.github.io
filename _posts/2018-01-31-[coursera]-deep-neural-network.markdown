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

$$ V_n = \beta * V_{n-1} + (1-\beta)* x_n $$

where \\(x_n\\) is the temperature reading of today. This formula is approximately the same as taking the average of the last m days, where m is 

$$ m = 1/(1-\beta) $$

This has the effect that when \\(beta\\) is closer to 1, or \\(m\\) is larger; you are taking average over a wider range of data, the avg line hence become smoother and most recent value has less affect on the average

To implement EWA we just need to keep track of the current \\(V\\) and keep updating it with

$$ V := \beta * V + (1-\beta) * x_n $$

EWA (Exponentially weighted avg) are used in machine learning in favor of simple avg because it is simple to implement & have extremely small memory footprint

### Gradient descent with momentum

The EWA above is used to update the weights and biases in gradient descent. Instead of doing:

$$ W := W - learning\_rate * dW $$

we do

$$ V_{dW} = \beta * V_{dW} + (1-\beta) * dW $$

$$ W:= W - learning\_rate * V_{dW} $$

similarly,

$$ V_{db} = \beta * V_{db} + (1-\beta) * db $$

$$ b:= b - learning\_rate * V_{db} $$

We can think \\(V_{db}\\) as the velocity, \\(\beta\\) as the friction and \\(dW\\) as the acceleration of the learning

#### Implementation details

The most common value for \\(\beta\\) is 0.9. It's approximately averaging the value of last 10 interations

\\(V_{dW}\\) is initialized to 0

\\(V_{db}\\) is initialized to 0

Sometimes you can see 

$$ V_{dW} = \beta * V_{dW} + dW $$

where the part \\(1-\beta\\) is removed. When this is used, the \\(learning\\_rate\\) is adjusted accordingly, making it less intuitive. Therefore the original form is always preferred.

## RMSprop (Root mean square prop)

Objective of RMSprop is to slow download learning biases and accelerate learning weights. The steps are as follow:
- calculate dW, db as normal
- compute

$$ S_{dW} = \beta * S_{dW} + (1-\beta)*dW^2$$

$$ S_{db} = \beta * S_{db} + (1-\beta)*db^2$$

- Assign

$$ W:= W - \alpha * \dfrac{dW}{\sqrt{S_{dW}}}$$

$$ b:= b - \alpha * \dfrac{db}{\sqrt{S_{db}}}$$

where \\(\alpha\\) is the learning rate
