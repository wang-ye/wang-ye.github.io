---
layout: post
title:  "Kaggle Public Leaderboard Hack"
date:   2017-01-26 19:40:01 -0800
---
Recently there was some big news from [Kaggle](https://www.kaggle.com/)'s [third data science bowl](https://www.kaggle.com/c/data-science-bowl-2017). This time, the competition asked competitors to predict whether a given image has cancer or not. This is a very challenging problem and attracts thousands of data enthusiasts in the world. However, in just a couple of weeks, [Oleg Trott](https://www.kaggle.com/olegtrott) predicted the labels of all testing images correctly and reached #1 in the public leaderboard, far better than all other competitors. What happened?


## How Submissions Are Evaluated
The validation dataset has 198 images, and every competitor is asked to predict a probability between 0 and 1 to indicate the cancer presence. The submission score is computed using log loss:


Everyone would train a model locally, and apply the model to predict the test datasets. Test dataset has 198 images. The competitors are asked to calculate the probility of image containing cancer for the 198 images. The final scores are via [log loss](https://www.kaggle.com/c/data-science-bowl-2017/details/evaluation):

![]({{ site.url }}/assets/log_loss.png)


## Potential Data Leaking Issue
The ground truth probability for each label is either 0 or 1.
If the prediction is 0.5, then whatever the ground truth probability is, the score for this image would be ln(2). To crack the label of one record, if we assign the first image's cancer probability as 0.9, while all other probability as 0.5, then the total log score would be 

```
(197 * ln(2) - y0 * ln(0.1) - (1 - y0) * ln(0.9)) / 198
```

Here *y0* is either 0 or 1. Given the score returned by Kaggle, we can get the *y0*!

However, the above brute force approach would take 198 times to crack all the labels. Given that each competitor can only submit a couple of times every day, the cracking would take months. Is there a faster way to do so?

## Oleg's Hack
Oleg later posted [a blog] (https://www.kaggle.com/olegtrott/data-science-bowl-2017/the-perfect-score-script) discussing his approach of the perfect score.
Basically, he proposed a way to crack 15 labels each time by using [sigmoid function](https://en.wikipedia.org/wiki/Sigmoid_function): 

```
sigmoid(-n * epsilon * 2 ** i)
```

Here n = 198, 0 <= i < 15, and epsilon = 1.05e-5. The Numpy code to generate the probability is:

```python
import numpy as np
def build_template(n, chunk_size):
    epsilon = 1.05e-5
    return 1 / (1 + np.exp(n * epsilon * 2 ** np.arange(chunk_size)))

n = 198
chunk_size = 15
template = build_template(n, chunk_size) 
```

Running the above code gets us the 15 probabilities, and it is easy to verify that these 32768 possible label combinations lead to different scores.

What is the underlying magic here? Assume for an image, the ground truth probability is 1. with Taylor Expansion at *2* on the sigmoid function, the calculated score is:

![]({{ site.url }}/assets/log2.png)

On the other hand, if the ground truth probability is 0, the calculated score is

![]({{ site.url }}/assets/log4.png)

By summing all the scores together, it is easy to know which ground truth probability is 0 and which is 1!

## Final Thoughts
If Kaggle returns more than 5 digit precision, say 9 digits, then we can crack even more images each time. Following Oleg's script, I wrote a [simple one](https://github.com/wang-ye/code/blob/master/python/kaggle_leaderboard_cracker.py) to play with the different conditions by varying epsilon, the precision and the base of the exponential expression.
With more precisions, we will be also able to crack more images each time. If you are willing to tolerate small uncertainties, then you can crack even more images with a high probability.