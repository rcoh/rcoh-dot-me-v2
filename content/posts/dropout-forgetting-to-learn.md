---
title: "Dropout, Deep Learning, and Forgetting to Learn"
date: 2018-07-12T11:01:16-07:00
draft: true
enableMath: true
tags: ["deep-learning", "python"]
---
Before I started studying deep learning in earnest, I was under the mistaken impression that the neural networks of the 1980s coupled with the CPUs of the 2010s lead to the incredible results that have been coming out of the last few years. In reality, that's simply false. Sure, modern processing power pays a critical role -- you simply can't process million-image datasets like [ImageNet](www.imagenet.org) without a GPU -- but they're only a small part of the picture. The deep learning results we're getting today actually stem from a series of shockingly simple innovations that yield huge performance improvements. In this post we'll explore "dropout", a simple enhancement, obvious in hindsight, that yields huge gains when training nearly _all_ types of neural networks. 

This post is targeted at people just getting their toes wet with deep learning -- I don't assume any knowledge except the absolute basics.

## The World Before Dropout
First, a quick primer on neural networks work, feel free to skip ahead. At their core, neural networks are built on simple pieces. Most modern neural network networks are composed of linear functions, `\(ax+b\)`, and "activations" `\(\max(0, x)\)`. When people talk about "weights", they're referring to `\(a\)` and `\(b\)` in `\(ax+b\)`. 

![An Extremely Simple Neural Network](/images/overfitting.svg)
![BST Read Performance](/images/basic-net.png)

Above is an incredibly simple neural network architecture, capable of learning simple regressions. It contains a "fully connected linear layer" followed by an activation. The training phase involves examining each of the weights in turn and determining how a small change to the weight impacts the output. If the target output is bigger, make a change to the weight that increases the target output. If the target output is smaller, do the opposite. Doing this for millions of weights simultaneously on vectors instead of scalars requires vector calculus (the "hard math" part of deep learning) but the core concept is the same.
 
Compared to other machine learning methods, neural networks have a massive number of free parameters. Typical architectures for image classification typically have 10-100 _million_ free parameters. Because of this, neural networks have a strong tendency to overfit the data. This is essentially equivalent to memorizing the training set instead learning its patterns. Just like human learning, memorization is useless if you need to be able to learn things you've never seen before. Ideally, there would be a way to train a neural network in way that prevented it from overfitting. If you squash the high dimensional space into two dimensional space, neural networks can very effectively learn the red function, minimizing error for a small set of (training) inputs. We want to train a neural network to learn the green line, equally good on the training inputs, but on general inputs as well.

## Enter Dropout
Developed by Nitish Srivastava, Geoffrey Hinton, et al. in 2013, dropout is one of the first generalizable techniques to combat overfitting. Dropout is simply ignoring, at random, the results of some neurons in the network each time you train a minibatch (subset) of your data. When we evaluate our performance on the validation data, we don't ignore any nodes. There is one wrinkle: Suppose we randomly ignore the output of 50% of the nodes (in Pytorch this would be `nn.Dropout(p=0.5)`). On average, the total output of the layer will be 50% less, confounding the neural network when run without dropout. Since neural networks are simply a linear combination, however, we can multiply the outputs at each layer by 2x to compensate.

Dropout was a big step forward when it was [originally proposed](http://jmlr.org/papers/volume15/srivastava14a/srivastava14a.pdf) resulting in new state-of-the-art results on most major machine vision problems. Since then, nearly every mainstream architecture uses dropout in most layers.