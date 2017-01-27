---
layout: post
title:  "Kaggle Public Leaderboard Hack"
date:   2017-01-26 19:40:01 -0800
---
Recently there was some big news from [Kaggle](https://www.kaggle.com/)'s [third data science bowl](https://www.kaggle.com/c/data-science-bowl-2017). This time, the competition asked competitors to predict whether a given image has cancer or not. This is a very challenging problem and attracts thousands of data enthusiasts in the world. However, in just a couple of weeks, [Oleg Trott](https://www.kaggle.com/olegtrott) predicted the labels of all testing images correctly and reached #1 in the public leaderboard, far better than all other competitors. What happened?


## How Submissions Are Evaluated
The validation dataset has 198 images, and every competitor is asked to predict a probability between 0 and 1 to indicate the cancer presence. The submission score is computed using log loss:


Everyone would train a model locally, and apply the model to predict the test datasets. Test dataset has 198 images. The competitors are asked to calculate the probility of image containing cancer for the 198 images. The final scores are via [log loss](https://www.kaggle.com/c/data-science-bowl-2017/details/evaluation):

![]({{ site.url }}/assets/log_loss.gif)

where
1. n is the number of patients in the test set
2. ŷiy^i is the predicted probability of the image belonging to a patient with cancer
3. yi is 1 if the diagnosis is cancer, 0 otherwise
4. log is the natural (base e) logarithm

## Data Leaking Issue
The ground truth probability for each label is either 0 or 1.
If the prediction is 0.5, then whatever the real probability is, the score for this record would be the same - ln(2). As a simple example, if we assign the first image's cancel probability as 0.9, while all other probability as 0.5, then the total log score would be 1 / 198 * (197 * ln(2) - y0 * ln(0.1) - (1 - y0) * ln(0.9)), where y0 is either 0 or 1. Given the score returned by Kaggle, we can get the label of y0!

However, the above brute force approach would take 198 times to crack all the labels. Given that each competitor can only submit a couple of times every day, the cracking would take months. Is there a faster way to do so?

## Oleg's Hack
Oleg later posted [a blog] (https://www.kaggle.com/olegtrott/data-science-bowl-2017/the-perfect-score-script) discussing his approach of the perfect score.
He proposed a way to crack 15 labels each time by constructing 15 probabilities using sigmoid function: sigmoid(- n * epsilon * 2 ** i).
where n=198, 0 <= i < 15, and epsilon = 1.05e-5.

```python
import numpy as np
def build_template(n, chunk_size):
    epsilon = 1.05e-5
    return 1 / (1 + np.exp(n * epsilon * 2 ** np.arange(chunk_size)))

n = 198
chunk_size = 15
template = build_template(n, chunk_size) 
```

What is the underlying magic here? 

Running the above code gets us the 15 probabilities, and it is easy to verify that these 32768 possible label combinations lead to different scores.

## What Else?
Actually, if we are willing to accept a bit of uncertainties, we can use the following probabilistic approach:
Pick K uniformly distributed values among [0, 1], and then run through the algorithm.
 But is there another way to crack the labels? Consider the following case - if predict all labels as

This would give you a lot of collisions. For example, when trying 

## Final Thuoghts