# RESTful APIs

## Workflow

The main workflow of Gorse is as follows:

<center><img width=480 src="img/workflow.png"/></center>

1. Feedbacks generated by users are collected to the data store.
2. Archived feedbacks are pulled to train the recommender model. There are two types of models (ranking model and CTR model) in Gorse, they are treated as one here.
3. Offline recommendations are generated in the background from all items and cached.
4. Online recommendations are returned to users in real-time based on cached offline recommendations.

## Data Storage

There are two types of storage used in Gorse: data store and cache-store.

### Data Store

The `data_store` is used to store items, users, feedbacks, and measurements. Currently, MySQL and MongoDB are supported as data storage. Other databases will be available once its interface is implemented.

Unfortunately, there are two challenges in data storage:

1. What if feedback with an unknown user or item is inserted? There are two options `auto_insert_user` and `auto_insert_item` to control feedback insertion. If new users or items insertion is forbidden, feedback with new users or items will be ignored.

2. How to address stale feedback and items? Some items and their feedbacks are short-lived such as news. `positive_feedback_ttl` and `item_ttl` are used to ignore stale feedback and items when pulling datasets from a data store.

### Cache-Store

The `cache_store` is used to store offline recommendation and temp variables. Only Redis is supported. The latest items, popular items, similar items, and recommended items are cached in Redis. The length of each cached list is `cache_size`.

## Recommendation

Recommended items come from multiple sources through multiple stages. Non-personalized recommendations (popular/latest/similar) are generated by the master node. Offline personalized recommendations are generated by worker nodes while online personalized recommendations are generated by server nodes.

### Popular Items

Items with the maximum number of users will be collected. To avoid popular items resist on the top list, `popular_window` restricts that timestamps of collected items must be after `popular_window` days ago. There will be no timestamp restriction if `popular_window` is `0`.

### Latest Items

Items with the latest timestamps are collected. Items won't be added to the latest items collection if their timestamp is empty.

### Similar Items

For each item, top n (n equals `cache_size`) similar items are collected. In the current implementation, the similarity between items is the number of common users of two items[^6].

### Offline Recommendation

Worker nodes collect top n items from all items and save them to cache. Besides, the latest items are added to address the cold-start problem in the recommender system. When labels of items exist, the CTR prediction model is enabled, vice versa. The procedure of offline recommendation is different depending on whether the CTR mode is enabled.

**If the CTR model is enabled:**

1. Collect top `cache_size` items from unseen items of current users using the ranking model.
2. Append `explore_latest_num` latest items to the collection.
3. Rerank collected items using the CTR prediction model.

**If the CTR model is disabled:**

1. Collect top `cache_size` items from unseen items of current users using the ranking model.
2. Insert `explore_latest_num` latest items to random positions in the collection.

Offline recommendation cache will be consumed by users and fashion will change. The offline recommendation will be refreshed under one of these two conditions:

- The timestamp of offline recommendation has been `refresh_recommend_period` days ago.
- New feedbacks have been inserted since the timestamp of the offline recommendation.

There are 4 ranking models (BPR[^5]/ALS[^3]/CCD[^4]) and 1 CTR model (factorization machines[^2]) in Gorse. They will be applied automatically by the model searcher. In ranking models, items and users are represented as embedding vectors. Since the dot product between two vectors is fast, ranking models are used to find top N items among all items. In CTR models, features from users or items are used in prediction. It's expensive to use CTR models to predict scores of all items.


### Model Search

There are many hyperparameters for each recommendation model in Gorse. However, it is hard to configure these hyperparameters manually even for machine learning experts. To help users get rid of hyperparameters tuning, Gorse integrates random search[^1] for hyperparameters optimization. The procedure of model search is as follows:
