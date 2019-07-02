+++
title = "Clinical Dataset Analysis and Patient Outcome Prediction via Machine Learning"
date = 2018-12-08T16:51:44-07:00
draft = false
summary = "Thesis defense by **Alex Buettner**"
# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = []
categories = []

# Featured image
# Place your image in the `static/img/` folder and reference its filename below, e.g. `image = "example.jpg"`.
# Use `caption` to display an image caption.
#   Markdown linking is allowed, e.g. `caption = "[Image credit](http://example.org)"`.
# Set `preview` to `false` to disable the thumbnail in listings.
[header]
image = ""
caption = ""
preview = true

+++

**Alex Buettner** from our research group successfully defended his thesis titled "Clinical Dataset Analysis and Patient Outcome Prediction via Machine
Learning" on December 07, 2018. Below is the abstract of his thesis:

## Abstract

We analyze and evaluate relevant machine learning methods for use in extracting and understanding clinical data sets in the context of optimization of clinical processes. Three data sets were considered to demonstrate the types and style of data found in the healthcare field: (a) the Pima Indians diabetes dataset (PIDD), a non-time-dependent diabetes onset study, (b) an alcoholism EEG dataset (AED), studying responses of alcoholic and control subjects when exposed to image stimulus, and (c) the diabetes readmission dataset (DRD), that focuses on factors that relate to diabetic patient readmission times. Each dataset is modeled using a variety of machine learning methods, including Bayesian, neural network, and decision tree methods, to better understand the advantages and disadvantages as applicable to rapid dependency extraction and understanding of the information contained therein. The goal of this work is to analyze the potential of machine learning for use in management of clinical processes and operations.

Neural network models are used to assess all three data sets; two using dense
neural networks, and one using convolutional neural networks. The dense neural
network model used on the PIDD resulted in a maximum prediction accuracy of
81.77%. In contrast, the use of neural network (NN) models on the much larger
DRD demonstrated some drawbacks that were not expected upon initial analysis of
the data. We found that the NN model performs poorly on this dataset, with classification accuracy no higher than 61.17%, due to the complexity of the dataset and potential need for more data. The use of convolutional neural networks for analysis of time series data was demonstrated on the alcoholism EEG dataset, resulting in subject classification accuracies between 91.41% and 98.82% depending on the training and testing sets used to analyze the model.

