---
layout: post
title: "Pylearn2 MLPs With Custom Data"
date: 2014-08-17 23:39:28 -0400
comments: true
categories: image-recognition machine-learning
description: I show identifying transformed US state maps using MLPs.
keywords: image recognition, machine learning, Hu moments, image moments, sklearn, MLP, pylearn2
published: true
---

I decided to revisit the [state map recognition problem](/2014/05/15/map-recognition/), only this time, rather than using an SVM on the Hu moments, I used an [MLP](http://en.wikipedia.org/wiki/Multilayer_perceptron). This is not the same as using a "deep neural network" on the raw image pixels themselves as I am still using using domain specific knowledge to build my features (the Hu moments).

As of my writing this blog post, scikit-learn does not support MLPs (see [this GSoC for plans to add this feature](https://github.com/scikit-learn/scikit-learn/wiki/GSoC-2014:-Extending-Neural-Networks-Module-for-Scikit-learn)). Instead I turn to [pylearn2](http://deeplearning.net/software/pylearn2/), the machine learning library from the [LISA lab](http://lisa.iro.umontreal.ca/index_en.html). While pylearn2 is [not as easy to use as scikit-learn](http://fastml.com/pylearn2-in-practice/), [there are some great tutorials](http://deeplearning.net/software/pylearn2/tutorial/notebook_tutorials.html) to get you started.

<!-- more -->

I added this line to the bottom of [my last notebook](/2014/05/15/map-recognition/#notebook-download) to dump the Hu moments to a CSV so that I could start working in a fresh notebook:

```python
items.to_csv(CSV_DATA, index=False)
```

In my new notebook, I performed the same feature normalization that had for the SVM: taking the log of the Hu moments and then mean centering and std-dev scaling those log-transformed values. I did actually test the MLP classifier without performing these normalization and like the SVM, classification performance degraded.

With the data normalized, I shuffled the data and then split the data into test, validation and training sets (15%, 15%, 70% respectively). The simplest way I found to shuffle the rows of a Pandas DataFrame was: `df.ix[np.random.permutation(df.index)]`. I initially tried `df.apply(np.random.permutation)`, which lead to each column being shuffled independently (and my fits being horrible).

I saved the three data sets out to three separate CSV files. I could then use the [CSVDataset](http://deeplearning.net/software/pylearn2/library/datasets.html#module-pylearn2.datasets.csv_dataset) in the training YAML to load in the file. I created my YAML file by taking the one used for classifying the MNIST digits in the [MLP tutorial](http://nbviewer.ipython.org/github/lisa-lab/pylearn2/blob/master/pylearn2/scripts/tutorials/multilayer_perceptron/multilayer_perceptron.ipynb), modifying the datasets and changing `n_classes` (output classes) from 10 to 50 and `nvis` (input features) from 784 to 7. I also had to reduce `sparse_init` from 15 to 7 after a mildly helpful assertion error was thrown (I am not really sure what this variable does but it has to be <= the number of input features). For good measure I also reduced the size of the first layer from 500 nodes to 50, given that I have far fewer features than the 784 pixels in the MNIST images.

After that I ran the fit and saw that my `test_y_misclass` went to 0.0 in the final epochs. I was able to plot this convergence using the [`plot_monitor` script](http://deeplearning.net/software/pylearn2/library/scripts.html#module-pylearn2.scripts.plot_monitor). Since [I like to have my entire notebook self-contained](https://github.com/lisa-lab/pylearn2/issues/1034), I ended up modifying the plot_monitor script to run as a function call from within IPython Notebook ([see my fork](https://github.com/cancan101/pylearn2/compare/ipython_embed_script)).

Here is [the entire IPython Notebook](http://nbviewer.ipython.org/gist/cancan101/ea563e394ea968127e0e).

I didn't tinker with regularization, but this topic is well addressed in the pylearn2 tutorials.

In a future post, I plan to use CNN to classify the state map images without having to rely on domain specific knowledge for feature design.
