---
title: Notes on Python Machine Learning - Chapter 02
date: 2023-07-15 18:07:00
math: true
categories:
- 笔记
tags: 
- 机器学习
- 学习笔记
---

This is the notes of the second chapter of the book *Python Machine Learning*.  
In this chapter, three linear classification algorithms with detailed explanation and python implementation are taught. 
- perceptron classification
- adaline with batch gradient descent
- adaline with stochastic gradient descent

<!-- more -->
## 1. Perceptron Classification
### 1.1 Main Concepts
- All features affects the output linearly with respective weights
- The training goal is to find proper weights that minimize the sum of samples' errors
- The training process is to update weights on every example:
	- compute predict value $y^{i}$:
       $$
	\begin{aligned}
		p^i &= X[i].dot(W[1:]) + W[0]\\
		y^{i} &= \begin{cases}
				1, &p^i \ge 0 \\
				-1, &p^i \lt 0
			   \end{cases}
	\end{aligned}
	
	$$

	- update the weights according to error $e^i$ and learning rate $\alpha$:
	 $$
	\begin{aligned}
		e^i &= Y^i - y^i\\
		W &= W + \alpha e^i
	\end{aligned}
	$$
	- here is why the updation works:
		- if $e^i = 0$, then $w^i$ gets no updation;
		- if $e^i \gt 0$, which means $y^i \lt Y^i$, then $w^i$ is increased by $\alpha e^i x^i$, making the predict $p^i$ closer to $Y^i$ next time we encounter the same example, because $x^i w^i$ is becoming more positive;
		- if $e^i \lt 0$, which means $y^i \gt Y^i$, then $w^i$ is decreased by $\alpha e^i x^i$, making the predict $p^i$ closer to $Y^i$ next time we encounter the same example, because $x^i w^i$ is becoming more negative.
- The trainning process stops after n epoches.

### 1.2 Python Implementation
```python
import numpy as np


class Perceptron:
    """
    The Perception classifier.
    """

    def __init__(self, learning_rate: float = 0.01, epochs: int = 1, seed: float = 1.0):
        """
        Initialize a perceptron classifier.
        """
        self.learning_rate = learning_rate
        self.epochs = epochs
        self.rgen = np.random.RandomState(seed)
        self.weights = None
        self.errors = None

    def _net_input(self, Xi: np.ndarray):
        return Xi.dot(self.weights[1:]) + self.weights[0]

    def predict(self, Xi: np.ndarray):
        pi = self._net_input(Xi)
        return np.where(pi >= 0.0, 1, -1)

    def fit(self, X: np.ndarray, Y: np.ndarray):
        self.weights = self.rgen.normal(loc=0.0, scale=0.01, size=X.shape[1] + 1)
        self.errors = []

        for i in range(self.epochs):
            errors = 0
            for Xi, Yi in zip(X, Y):
                yi = self.predict(Xi)
                update = self.learning_rate * (Yi - yi)
                self.weights[0] += update
                self.weights[1:] += Xi * update
                errors += int(update != 0.0)
            self.errors.append(errors)

```


## 2. Adaptive Linear Neurons Classification with Batch Gradient Descent
### 2.1 Main Concepts
- All features affects the output linearly with respective weights
- The training goal is to find the $W$ that minimize the output of cost function $J(W) = \frac{1}{2}\sum_{i=1}^{n}(Y[i] - X[i] \cdot W)^2$ 
- The training process is to update weights on every iteration of the whole dataset:
	- We assume that $J(W)$ is minimized if $J(w)$ is minimized for every $w$ in $W$;
	- To minimize $J(w)$, we constantly decrease $w$ by grediant descent of $J(w)$, untill we hit the global minimun;
	- To compute the gradient , we need to compute the partial derivative of the cost function with respect to each weight $w$ :
	 $$
	\begin{aligned}
	J(W) &= \frac{1}{2}\sum_{i=1}^{n}(Y[i]-X[i] \cdot W)^2\\
	\frac{\partial J}{\partial w_j} &= \frac{\partial}{\partial w_j}\frac{1}{2}\sum_{i=1}^{n}(Y[i] - X[i] \cdot W)^2\\
	&=\sum_{i=1}^{n}\frac{1}{2}2(Y[i] - X[i] \cdot W)(0-(x[_1^iw_1] + ... + x^i_jw_j + ... x^nw)\prime)\\
	&=\sum_{i=1}^{n}(Y[i] - X[i] \cdot W) \cdot -x_j^i\\
	&=-\sum_{i=1}^{n}(Y[i] - X[i] \cdot W) \cdot x_j^i
	\end{aligned}
	$$
### 2.2 Python Implementation
```python
import numpy as np


class AdalineGD:
    """
    The Adaptive Linear Neuron Classifier with Gradient Descent.
    """

    def __init__(self, learning_rate: float = 0.01, epochs: int = 1, seed: float = 1.0):
        """
        Initialize a AdalineGD classifier.
        """
        self.learning_rate = learning_rate
        self.epochs = epochs
        self.rgen = np.random.RandomState(seed)
        self.weights = None
        self.costs = None

    def _net_input(self, X: np.ndarray):
        return X.dot(self.weights[1:]) + self.weights[0]

    def predict(self, Xi: np.ndarray):
        return np.where(self._net_input(Xi) >= 0.0, 1, -1)

    def fit(self, X: np.ndarray, Y: np.ndarray):
        self.weights = self.rgen.normal(loc=0.0, scale=0.01, size=X.shape[1] + 1)
        self.costs = []

        for i in range(self.epochs):
            errors = Y - self._net_input(X)
            self.weights[0] += self.learning_rate * sum(errors)
            self.weights[1:] += self.learning_rate * X.T.dot(errors)
            costs = (errors ** 2).sum() / 2.0
            self.costs.append(costs)

```
## 3. Adaline With SGD
### 3.1 Main Concepts
- It's an improvement for Adaline classifier with gradient descent, which update weights on every example 
- It has the following:
	- suitable for large datasets
	- suitable for online learning
	- better at escaping local minima
- It must shuffle the datasets in every epoch to avoid cycles
### 3.2 Python Implementation
```python
import numpy as np


class AdalineSGD:
    """
    The Adaptive Linear Neuron Classifier with Stochastic Gradient Descent.
    """

    def __init__(self, learning_rate: float = 0.01, epochs: int = 1, seed: float = 1.0):
        """
        Initialize a AdalineSGD classifier.
        """
        self.learning_rate = learning_rate
        self.epochs = epochs
        self.rgen = np.random.RandomState(seed)
        self.weights = None
        self.costs = None

    def _net_input(self, X: np.ndarray):
        return X.dot(self.weights[1:]) + self.weights[0]

    def _shuffle(self, X: np.ndarray, Y: np.ndarray):
        indices = self.rgen.permutation(len(Y))
        return X[indices], Y[indices]

    def predict(self, Xi: np.ndarray):
        return np.where(self._net_input(Xi) >= 0.0, 1, -1)

    def fit(self, X: np.ndarray, Y: np.ndarray):
        self.weights = self.rgen.normal(loc=0.0, scale=0.01, size=X.shape[1] + 1)
        self.costs = []

        for i in range(self.epochs):
            costs = []
            Xs, Ys = self._shuffle(X, Y)
            for Xi, Yi in zip(Xs, Ys):
                error = Yi - self._net_input(Xi)
                self.weights[0] += self.learning_rate * error
                self.weights[1:] += self.learning_rate * Xi * error
                costs.append((error ** 2) / 2.0)
            cost = sum(costs) / len(costs)
            self.costs.append(cost)

```