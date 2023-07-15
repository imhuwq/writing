---
title: Notes on Python Machine Learning - Chapter 01
date: 2023-07-15 18:07:00
math: true
categories:
- 笔记
tags: 
- 机器学习
- 学习笔记
---

This is the notes of the first chapter of the book *Python Machine Learning*.  
It introduces three types of machine learning tasks and the common pre-steps of these tasks.  

<!-- more -->

## 1. Three Types of ML

| type | feature | task |
| - | - | - |
| `Supervised Learning` | <pre>Labeled data</br>Direct feedback</br>Predict outcome/future</pre> | <pre>Classification</br>Regression</pre> |
| `Unsupervised Learning` | <pre>No labels</br>No Feedback</br>Find hidden data structure</pre> | <pre>Clustering</br>Dimensionality Reduction</pre> |
| `Reinforcement Learning` | <pre>Decision process</br>Reward System</br>Learn series of actions</pre> | <pre>Interactive problems</pre> |

## 2. Pre-Steps of ML
### 2.1 Process Data
- Extract meaningful features
- Scale features value range
- Dimensionality reduction
### 2.2 Split SubSets
- Train
	- The training set is used to train the model
	- 70-80%
- Validation
	- tune the model's hyperparameters and to make decisions about the model's structure
	- 10-15%
- Test
	- provide an unbiased evaluation of the final model
	- 10-15%
### 2.3 Choose Performance Metrics
- Accuracy
	- $\frac{TP + TN}{TP + TN + FP + FN}$
	- the proportion of predictions that your model got right, among all predictions it made
	- accuracy can be misleading if your classes are imbalanced among different classes
- Precission
	- $\frac{TP}{TP + FP}$
	- the proportion of true positive predictions among **all positive predictions**
	- tend to be higher if the model tends to make **less** true positive predictions
	- it says nothing about the items it failed to label as positive
	- suitable if you want to avoid false positives(identifying spam emails)
- Recall
	- $\frac{TP}{TP + FN}$
	- the proportion of true positive predictions among **all actual positives**
	- tend to be higher if the model tends to make **more** true positive predictions
	- it says nothing about the items it falsely labeled as positive
	- suitable if you want to avoid false negatives(identifying disease)
- F1 Score
	- $2\frac{Precision * Recall}{Precision + Recall}$
	- balance precision and recall
