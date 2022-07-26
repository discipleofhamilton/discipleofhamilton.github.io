---
title: Auto-Tiering
date: 2022-07-26 21:31:36
tags: 'Auto Tiering Storage'
---

The project is an algorithm reproduction of the paper "[Automating Distributed Tiered Storage Management in Cluster Computing](https://arxiv.org/pdf/1907.02394.pdf)". In brief, the main idea concept of the prediction is HOW TO GENERATE THE USEFUL DATA OR FEATURE TO THE MODEL. The paper emphasis on how to collect the data, preprocessing, and determine which attribution is useful, then extract them to the features. The model it use is XGBoost, and it says directly. There only 2 hyper parameters matter: one is "the depth" of the tree, the other is "the rounds" of the training. 



## Data Preprocessing

Before training and testing the model, we need to preprocess the data to get the features and labels. When the model would be released, the process still need to be done because of the online training policy.

As paper says there are 3 basic and significant attributes:

1. the creation time of a file
2. the last k access time of a file
3. the file size of a file

Because the access time increases by the time, the author propose a method: replace using time stamp directly with time interval which named time delta in the paper. 

Besides the time delta idea, there is another idea called reference time which is time point to separate the "past" from the "future". For training, collect the features with the basic attributes from the "past", and then mark the label "1" if the file will be access in the "future" but mark "0". Here come two expanded parameters:

1. the interval between the reference time which named "reference poll size" by me
2. the window size of the future, which means the next k minutes we want to predict



### Steps

![data-preprocessing](https://i.imgur.com/BZlp9m7.png)

<center>Figure1: Data Preprocessing Workflow </center>



Figure 1 is the full workflow of data processing. There are 3 steps:

1. Clean the raw data/log
2. Parse structured data to get processed data and reference list
3. Prepare dataset which contains features and labels



#### Step 1: Data Clean

Raw data is quite messy and unstructured. The GOAL is to structurize it and output `processed_log` which contains 4 useful attributes:

1. time: Convert the date and current to minutes, because the basic time unit that would be predicted is minutes.
2. file path: The absolute path to the file.
3. file size: Bytes.
4. mode: ACCESS, MODIFY, CREATE, ...

```python
data = {
        "time": t,
        "file_path": file_path,
        "size": size,
        "mode": mode
        }
```



**The paper use 6-hour data for training, validation and testing. In order to reproduce the result, I cut the 6-hour data from my log at the step.**



#### Step 2: Data Parsing

The GOAL is to get the file based dataset `processed_data` and time based dataset `reference_time_list` from `processed_log`. 

`processed_data`: 

```python
processed_data[file] = {
    "create_time": c_time,
    "access_time_list": acs_time,
    "file_size": fsize
}
```

1. Use double map structure to store the 3 main attributes of each file.
2. Find the lack creation time, use the other tool to get it and fill into the dataset.



`reference_time_list`:

![reference-list](https://i.imgur.com/rb87ExE.png)

<center>Figure2: Reference Time List</center>

Reference time list is to know when to get the features and predict. It and `reference poll size` is related to how to split whole dataset into training, validation and testing set. The description of how to split is in Step 3.



#### Step 3: Generate Features & Labels

In the Step, I need to deal with 2 issue: 

1. Features and labels generate
2. Prepare training, validation, testing set



![gen-feat-label](https://i.imgur.com/g7xCffq.png)

<center>Figure3: Features and Labels Generate</center>



**Features and labels generate**: Figure 3 is an example and it is Figure 4 in the paper, the correction with red ink is a typo which I asked the first author already. 

> Hence, we use the timestamps to generate time deltas, that is, the time difference between (i) two consecutive time accesses, (ii) the oldest time access and creation time, (iii) a reference time and the most recent access, and (iv) a reference time and creation time. A reference time is a particular point in time chosen to separate the perceived “past” from the perceived “future”. The past represents when and how frequently a file has been accessed and, thus, it is used to generate the feature vectors ~xi. On the other hand, the future shows whether the file will be reaccessed in a given forward-looking class window, which is used to generate the class label y: if a file is accessed during the window, then y= 1; otherwise y= 0. Note that by sliding the reference time in the time axis, we can generate multiple training points (i.e., feature vectors and corresponding class values) based on the access history of a single file.
>
> The final step in our data preparation involves normalizing the features by rescaling all the values to lie between 0 and 1. Normalization is performed by diving the time deltas by a maximum time interval (e.g., 1 month), which is useful for avoiding outliers from situations where a file was not accessed for a long time. Figure 4 shows a complete example of the training data preparation process. As the file is accessed 3 times before the chosen reference time, the feature vector will contain 5 normalized time deltas as explain above and one file size feature. The remaining k − 3 access-based features are encoded as missing values. The class value is set to 1 since the file is accessed within the class window.

**Here are 3 further issues**:

1. How to normalize the file size and time delta?

   I have done double check. First, I calculated the denominator from the example and got 4GB file size and 2 days time interval as a denominator for normalization. Second, I asked the first author of the paper about the what size of the denominator did he use. The responses are exactly the same of mine.

2. What size of the future/class window is appropriate?

   I use the 30 minutes which is recommended by the paper.

3. What size of access time list is appropriate? what is `k`?

   The size I use is 12. The reason is in Section 7.6 of the paper.


![training-validation-testing](https://i.imgur.com/KE0k1bp.png)

<center>Figure4: Preparation of Training, Validation and Testing Dataset</center>



**Prepare training, validation, testing set**:

I split the first 4 hours as training set. To remain the features and labels consist, every points in time of the split need to be in reference time list. e.g. $r_k$ is the split time point and the $k^{th}$ reference time.



## Training

![training-workflow](https://i.imgur.com/zuFRqPg.png)

<center>Figure3: Training Workflow</center>





## Testing

![testing-workflow](https://i.imgur.com/kjODcz5.png)

<center>Figure4: Testing Workflow</center>



`future_win_size`: 30 mins, `reference_poll_size`: 15 mins

| Accuracy | Precision | Recall | F1-Score |
| :------: | :-------: | :----: | :------: |
|  87.22%  |  93.33%   | 81.16% |  86.82%  |



`future_win_size`: 30 mins, `reference_poll_size`: 30 mins

| date       | t_accuracy | t_precision | t_recall | v_accuracy | v_precision | v_recall | record_file_num | start_time | Training time (secs) | use_model_selection |
| ---------- | ---------- | ----------- | -------- | ---------- | ----------- | -------- | --------------- | ---------- | -------------------- | ------------------- |
| 2021/11/12 | 88.47%     | 74.22%      | 91.21%   | 83.27%     | 82.56%      | 68.74%   | 1759            | 11:00:00   | 0.113                | False               |
| 2021/11/13 | 82.87%     | 78.52%      | 88.50%   | 75.42%     | 67.01%      | 86.90%   | 618             | 10:00:00   | 0.212                | False               |
| 2021/11/14 | 92.18%     | 93.87%      | 88.93%   | 67.43%     | 62.45%      | 81.25%   | 632             | 10:00:00   | 0.093                | False               |
| 2021/11/15 | 84.54%     | 65.84%      | 96.52%   | 71.23%     | 50.26%      | 84.91%   | 1094            | 10:00:00   | 0.128                | False               |
| 2021/11/16 | 87.98%     | 82.82%      | 86.07%   | 80.28%     | 92.58%      | 65.99%   | 1660            | 10:00:00   | 0.115                | False               |
| 2021/11/17 | 78.76%     | 58.91%      | 84.43%   | 84.84%     | 64.61%      | 88.89%   | 1620            | 10:00:00   | 0.131                | False               |
| 2021/11/18 | 86.00%     | 88.49%      | 79.17%   | 86.58%     | 90.08%      | 79.84%   | 1324            | 10:00:00   | 0.106                | False               |
| 2021/11/19 | 89.87%     | 83.36%      | 77.05%   | 88.81%     | 78.80%      | 80.44%   | 1768            | 10:00:00   | 0.197                | False               |
| 2021/11/20 | 89.07%     | 81.76%      | 99.45%   | 94.41%     | 95.88%      | 94.26%   | 624             | 10:00:00   | 0.096                | False               |
| 2021/11/21 | 83.76%     | 75.94%      | 98.99%   | 83.89%     | 84.80%      | 89.23%   | 626             | 10:00:00   | 0.099                | False               |
| 2021/11/22 | 84.60%     | 81.97%      | 83.38%   | 87.39%     | 82.84%      | 90.82%   | 983             | 10:00:00   | 0.119                | False               |
| 2021/11/23 | 85.71%     | 82.58%      | 82.20%   | 91.02%     | 92.49%      | 89.00%   | 1019            | 04:00:00   | 0.129                | False               |
