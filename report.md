# Data Mining Assignment 2 - Recommender Systems

GitHub repository: `[add repository link here]`

## 1. Introduction
The goal of this assignment is to build and evaluate recommender models for movie recommendation, and then generate 10 recommendations for each user in `ratings_test.csv`. I focused on two things the assignment explicitly asks for:

- rating prediction quality, measured with `RMSE`
- top-10 recommendation quality, measured mainly with `Precision@10` and `Recall@10`

I compared three models:

- a bias-based baseline
- a user-based collaborative filtering model
- an SVD matrix factorization model

The final pipeline also had to handle cold-start users, because some users in the test file do not appear in the training data.

## 2. Data Understanding And Preprocessing
The training data contains `97,801` ratings from `600` users on `9,680` movies. The user-item matrix is about `98.32%` sparse, so most user-movie pairs are missing. This is an important property because sparse data makes neighborhood methods less reliable and usually favors models that can generalize better.

Some additional statistics also help explain the later results:

- the average number of ratings per user is `163.0`, but the median is only `69.5`
- the average number of ratings per movie is `10.1`, but the median is only `3.0`
- about `59.66%` of all ratings come from the top `10%` most-rated movies
- `62` movies in `movies.csv` have no ratings in the training data

These numbers show a strong long-tail structure. A relatively small number of movies receive many ratings, while most movies receive very few. This matters because popular movies are easier to recover, so a popularity-heavy model can look stronger than expected on ranking metrics.

During preprocessing, missing values and duplicate user-movie ratings were checked. No missing values and no duplicate ratings were found. Movie genres were also split into lists so the movie metadata would be easier to inspect, and per-movie statistics such as average rating and rating count were added for later analysis and for the cold-start fallback.

Finally, I checked whether the users in `ratings_test.csv` already appeared in the training set. `90` test users were warm users and `10` were cold-start users, so `10%` of the final recommendations required a fallback strategy.

## 3. Models And Evaluation Setup
For validation, I used a temporal split. For each user, the ratings were sorted by timestamp and the last `5` interactions were held out. The remaining ratings were used for training. This was a better choice than a fully random split because timestamps make it possible to respect interaction order.

For ranking evaluation, only held-out ratings of `4.0` or higher were treated as relevant items. In this split, `1,696` held-out ratings were relevant, and `533` out of `600` users had at least one relevant held-out item. This means `88.83%` of users contributed to the ranking evaluation.

The models were:

### Baseline
The baseline model predicts ratings from the global mean together with user and item bias terms. This gives a simple reference point and is useful because it separates real collaborative effects from global popularity and bias effects. In Surprise, this corresponds to `BaselineOnly`, which predicts from `mu + b_u + b_i` [3].

### User-Based Collaborative Filtering
The user-based model estimates a user's preference from similar users. This is intuitive and easy to explain in an oral defence, but it depends heavily on overlap between users. In sparse data, that overlap can be weak, which often hurts ranking quality.

### SVD Matrix Factorization
The SVD model learns latent user and item factors and combines them with bias terms. This is a standard matrix factorization approach and is often better suited for sparse recommender data because it does not rely as directly on local user-user overlap [1][2].

I reported `RMSE`, `Precision@10`, and `Recall@10` as the main metrics because they are explicitly requested in the assignment. I also kept `HR@10` and `NDCG@10` as supporting metrics, since they help describe ranking quality but are not the main basis of the comparison.

## 4. Results And Discussion
The main validation results are shown below.

| Model | RMSE | P@10 | R@10 | HR@10 | NDCG@10 | Time (s) |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Baseline (mean+biases) | 0.9389 | 0.0756 | 0.2255 | 0.5488 | 0.1761 | 0.3 |
| User-Based CF (KNNBaseline) | 0.9398 | 0.0370 | 0.1137 | 0.3114 | 0.0711 | 1.8 |
| SVD | 0.9366 | 0.0638 | 0.1894 | 0.4883 | 0.1424 | 0.4 |

At first glance, the baseline looks surprisingly strong, especially on `Recall@10`. I do not think this is accidental. Because nearly `60%` of all ratings come from the top `10%` most-rated movies, popular items are relatively easy to recover. A model that captures global movie effects can therefore perform well on ranking metrics even without much personalization.

To make that trade-off more visible, I also compared the recommendation lists themselves across `50` sampled users:

| Model | Coverage (unique movies) | Avg. popularity of recommended movies | Max repeat of one movie |
| --- | ---: | ---: | ---: |
| Baseline | 43 | 129.4 ratings | 46 users |
| User-Based CF | 252 | 14.1 ratings | 15 users |
| SVD | 160 | 88.1 ratings | 17 users |

This table explains the model behavior much better:

- the baseline repeats the same popular movies for many users
- the user-based model is much more personalized, but its accuracy is clearly weaker
- SVD sits between the two and gives the most balanced result

In other words, the baseline is strong as a benchmark, but it is too concentrated around the same high-popularity titles. The user-based model offers more personalization, but its `RMSE`, `Precision@10`, and `Recall@10` are all clearly below SVD. Given the `98%` sparsity and the very low median number of ratings per movie, this is consistent with expectations.

SVD gives the best `RMSE`, clearly better ranking quality than user-based CF, and much broader coverage than the baseline. For that reason, I selected SVD as the final personalized model.

I also ran a small grid search for SVD. The tuned version improved cross-validation `RMSE`, but it did not improve `Precision@10` or `Recall@10` on the temporal validation split. Since that search optimized only `RMSE` on random folds, I kept the simpler default SVD configuration for the final system.

## 5. Final Recommendation Strategy
For the final recommendations, I retrained SVD on the full training set and generated the top `10` unseen movies for each warm user.

The final pipeline handled two cases:

- `Warm users`: score all unseen movies with SVD and return the top `10`
- `Cold-start users`: return a popularity-based fallback list

The fallback was needed for `10` out of `100` test users. For those users, collaborative filtering cannot personalize recommendations because there is no interaction history. Instead of using plain average rating, I used a weighted rating formula so that movies with very few ratings would not appear artificially strong.

This solution is simple, but it is transparent and fits the assignment well. It keeps personalization where the data supports it and uses a reasonable fallback where personalization is not possible.

## 6. Conclusion
This dataset is challenging because it is highly sparse and strongly long-tail. Those properties make popularity-based recommendations look stronger than expected and make user-based neighborhoods less stable.

Among the tested models, SVD gave the best balance for this assignment. It had the strongest `RMSE`, better ranking results than user-based CF, and substantially broader recommendation coverage than the baseline. The final system therefore uses SVD for warm users and a weighted popularity fallback for cold-start users.

I do not claim that this fully solves recommendation quality or cold-start. However, I think it is a logical and well-motivated solution for the scope of the assignment.

## References
[1] Koren, Y., Bell, R., & Volinsky, C. (2009). *Matrix factorization techniques for recommender systems*. Computer, 42, 30-37. Available at: https://www.chrisvolinsky.com/publications/17545-matrix-factorization-techniques-for-recommender-systems

[2] Surprise documentation, *Matrix Factorization-based algorithms*. Available at: https://surprise.readthedocs.io/en/stable/matrix_factorization.html

[3] Surprise documentation, *Basic algorithms*. Available at: https://surprise.readthedocs.io/en/stable/basic_algorithms.html
