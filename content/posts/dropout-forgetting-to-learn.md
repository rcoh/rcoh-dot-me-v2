---
title: "Dropout, Deep Learning, and Forgetting to Learn"
date: 2018-07-12T11:01:16-07:00
draft: true
enableMath: true
tags: ["deep-learning"]
---
Dropout is the counterintuitive process of disabling certain neurons when training a neural network. In many cases, using dropout decreases overfitting and improves the accuracy of the network. More details in a moment.

Before I started studying deep learning in earnest, I was under the mistaken impression that the neural networks of the 1980s coupled with the GPUs of the 2010s lead to the incredible results that have been coming out of deep learning over the last few years. In reality, that's not the case. Sure, modern processing power pays a critical role -- you can't process million-image datasets like [ImageNet](www.imagenet.org) without a GPU -- but GPUs only represent a small part of the picture. The deep learning results we're getting today actually stem from a series of fairly simple innovations that yield huge improvements in the efficacy of using neural networks for machine learning problems. Dropout seems obvious in hindsight but it yields huge gains when training nearly _all_ types of neural networks.[^itdid]

This post is targeted at people just getting their toes wet with deep learning -- I don't assume any knowledge except the absolute basics.

## The World Before Dropout
First, a quick primer on how neural networks work, feel free to skip ahead. At their core, neural networks are built on simple pieces. Most modern neural network networks are composed of linear functions, `\(ax+b\)`, and "activations" `\(\max(0, x)\)`. When people talk about "weights", they're referring to `\(a\)` and `\(b\)` in `\(ax+b\)`. 

![An Extremely Simple Neural Network](/images/basic-net.svg)

Above is an incredibly simple neural network architecture, capable of learning simple regressions. It contains a "fully connected linear layer" followed by an activation. The training phase involves examining each of the weights in turn and determining how a small change to the weight impacts the output. If the target output is bigger, make a change to the weight that increases the target output. If the target output is smaller, do the opposite. Doing this for millions of weights simultaneously, on vectors instead of scalars requires vector calculus (the "hard math" part of deep learning) but the core concept is the same. For a longer intro with a bit more detail, [Jay Allamar's](http://jalammar.github.io/visual-interactive-guide-basics-neural-networks/) post is a good resource.[^notjustlinear]
 
Compared to other machine learning methods, neural networks have a massive number of free parameters. Typical architectures for image classification typically have 10-100 _million_ free parameters.[^freeparams] More importantly, the use of large fully connected linear layers means tuning many parameters at the same time, worsening the problem. Because of this, neural networks have a tendency to overfit the training data. Overfitting is essentially equivalent to memorizing the training set instead learning its patterns. Just like non-artificial learning, memorization is of limited utility if you need to be able to handle things you've never seen before.  Squashing high dimensional space into two dimensional space, neural networks can very effectively learn the red function, minimizing error for a small set of (training) inputs. We want to train a neural network to learn the green function, good on the training inputs, but on effective for general inputs as well.

![Sketch of overfitting](/images/overfitting.svg)

Ideally, there would be a way to train a neural network in way that prevented it from overfitting.

## Enter Dropout
Developed by Nitish Srivastava, Geoffrey Hinton, et al. in 2013, dropout is one of the first generalizable techniques to combat overfitting. Dropout is simply ignoring, at random, the results of some neurons in the network each time you train a minibatch (subset) of your data. When we evaluate our performance on the validation data, we don't ignore any nodes. There is one wrinkle: Suppose we randomly ignore the output of 50% of the nodes. On average, the total output of the layer will be 50% less, confounding the neural network when run without dropout. Since neural networks are simply a linear combination, however, we can multiply the outputs at each layer by 2x to compensate. This process is known as re-scaling. For more details on the nitty gritty and the original performance results, see the [original paper](https://www.cs.toronto.edu/~hinton/absps/JMLRdropout.pdf).

{{<figure src="/images/standard-vs-dropout.png" attr="Image from Srivastava et al 2014" attrlink="" >}}

## Dropout in 2018
Dropout was a big step forward when it was [originally proposed](http://jmlr.org/papers/volume15/srivastava14a/srivastava14a.pdf) resulting in new state-of-the-art results on most major machine vision problems. In combination with batch normalization, dropout is still the goto technique when training fully connected layers such as in the example above. In computer vision, however, the convolutional kernels used by most models have a much smaller number of parameters per-layer, rendering dropout less effective. Most models like ResNet are using [batch normalization](https://towardsdatascience.com/batch-normalization-in-neural-networks-1ac91516821c) instead.[^resnet] However, the recent work of Sergey Zagoruyko, Nikos Komodakis (2017) experiments with a wider version of Resnet and seems to show that dropout can still be effective in convolution architectures.[^stilldropout]

## Closing Thoughts
When I first learned about dropout, I loved its counter intuitive nature -- in order to learn properly, you need to forget part of what you've learned. I think we see this in real life as well, where society has "overfit" on specific solutions to problems, blinding it to a potentially more general solution. For a much more rigorous exploration of these concepts, I recommend Antifragile by Nassim Nicholas Taleb. In some sense, dropout is a way to make neural networks more "antifragile."
***
{{% subscribe %}}


[^stilldropout]: https://arxiv.org/pdf/1605.07146.pdf Zagoruyko and Komodakis propose a much wider Resnet block. Their new, wider, Resnet block introduces significantly more parameters per layer -- as such, they return to the original purpose of dropout, to prevent overfitting in layers with 1000s of parameters. 

[^resnet]: The standard Resnet uses batch normalization prior to the ReLU instead of dropout: https://towardsdatascience.com/an-overview-of-resnet-and-its-variants-5281e2f56035

[^notjustlinear]: It's important to call out that neural networks in 2018 aren't just fully connected linear layers like the picture. Vision based neural networks are based on convolutional kernels. Networks for learning longer sequences of data like language use "recurrent neural networks" or more recently "attention networks".

[^itdid]: Dropout made a big difference across the board when it was introduced because other modern techniques had yet to be developed. Other techniques like batch normalization have replaced it in certain cases today.

[^freeparams]: https://arxiv.org/pdf/1702.08782.pdf, page 5