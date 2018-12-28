---
layout: post
title:  "Implicit recommender"
categories: [ml]
comments: false
math: true
excerpt: "Implicit dataset / feedback for recommender systems."
---

[This paper](http://yifanhu.net/PUB/cf.pdf) by Yifan Hu et al. introduces me to implicit feedback for recommender systems and I like the paper. Recall the Netflix problem where a subset of user ratings are revealed and we want to predict ratings. Where ratings are present, we call it an **explicit dataset**. In practice, what is more common is whether the user watched a movie or reserved a restaurant or clicked an ad. We only observe **user interactions** and such a dataset is an **implicit dataset**.

Advantage of an implicit dataset: More data is available in this form. Most users do not bother to give ratings.

Disadvantage of an implicit dataset: Data is harder to interpret. Unlike an explicit dataset, whenever a user-item pair is missing, we cannot be sure whether it is because the user hasn't interacted with the item, or because the user does not like the item.

We will consider **ALS** (Alternating Least Squares) collaborative filtering. There are other methods for recommender engines, but ALS is what we will focus on for now. It is also known to perform well for users that have sufficiently interacted with the items. For relatively new users, content-based methods are more appropriate as they do not have to deal with the cold-start problem.

# Objective function

First, recall the ALS method for the explicit dataset. Let $$r_{ui}$$ be user rating between user u and item i. Minimize the RMSE loss:

$$ \min_{x_u,y_i} \sum_{ui} (r_{ui} - x_u^T  y_i)^2 + \lambda(\sum_u \|x_u\|^2 + \sum_i \|y_i\|^2).$$

Note that this first sum over u, i is over observed entries only. Also note that length of $$x_u, y_i$$ is the same, which is the number of "latent factors" in the model. Essentially, we are trying to approximate a matrix $$R$$ as a low rank matrix $$XY^T$$ where $$x_u, y_i$$ are *rows* of $$X,Y$$ respectively.

For the implicit dataset, $$r_{ui}$$ is instead of the number of interactions between user u and item i. We introduce a new indicator variable $$p_{ui}$$ which is 1 if $$r_{ui}>0$$ and 0 otherwise. We want instead a row rank approximation of the boolean matrix $$P$$. The $$r_{ui}$$'s however are used as a **confidence value**:

$$\min_{x_u,y_i} \sum_{ui} c_{ui} (p_{ui} - x_u^T  y_i)^2 + \lambda(\sum_u \|x_u\|^2 + \sum_i \|y_i\|^2)$$

where the confidence value $$c_{ui} = 1+\alpha r_{ui}$$ or $$c_{ui} = 1+\alpha \log(1 + r_{ui}/\epsilon)$$. Observe that most of $$c_{ui}$$'s are equal to 1 as $$r_{ui}$$ is mostly zero. This will be important later.

# ALS modified for implicit

ALS works by fixing X, solving for Y and vice versa, repeatedly. For the explicit dataset, the solution for $$x_u$$ (for every user u) is

$$ x_u = (Y^T Y + \lambda I)^{-1} Y^T R(u) $$

where $$R(u)$$ is a vector of length equal to the number of items. Remember Y is a tall matrix. The number of rows is the number of items. The number of columns is the number of factors. This equation is the standard least squares solution. The solution for $$y_i$$ is unsurprisingly $$ y_i = (X^T X + \lambda I)^{-1} X^T R(i) $$ where $$R(i)$$ is a vector of length equal to the number of users.

For an **implicit** dataset, we have to replace $$R(u),R(i)$$ with the boolean matrix values $$P(u), P(i)$$. In addition, we need to bring in the confidence values. It can be shown that the solution for $$x_u$$ is

$$ x_u = (Y^T C^u Y + \lambda I)^{-1} Y^T C^u P(u) $$

where $$C^u$$ is a square diagonal matrix. Its number of rows or columns is equal to the number of items. And its diagonal entry value is

$$C^u_{ii} = c_{ui}.$$

Let f be the number of factors. Typically it is really small so the matrix inversion is really cheap. The computational bottleneck lies in computing $$Y^T C^u Y$$ and we need a **computation trick**. While $$C^u$$ is a very large matrix, most of its entries are equal to 1. In other words, if $$C^u - I$$ is a really **sparse** matrix. Hence, we evaluate $$Y^T C^u Y$$ as

$$Y^T C^u Y =  Y^T (C^u - I) Y + Y^T Y.$$

The term $$Y^T Y$$ can be computed just once and used for every user u. Say Y is a $$n\times f$$ matrix and $$n$$ is the number of items. Then computing $$Y^T Y$$ takes only $$O(nf^2)$$ which is small. For each user, computing $$Y^T (C^u - I) Y$$ takes $$O(n_u f^2)$$ where $$n_u$$ is the number of item-ratings user $$u$$ has. Summing over users, we have $$O(s f^2)$$ where $$s=\sum_u n_u$$ is the support size of the interaction matrix $$R$$. Including the matrix inversion, the overall running time for solving all $$x_u$$'s is equal to $$O(f^3 + n f^2 + s f^2)$$. This is **linear** with the support size, while fixing $$f$$.

Solving for $$y_i$$'s is similar and we omit the details.

# Remarks
Sparse, diagonal, low rank matrices are nice to work with, and so are linear combinations of them. Here we are using "diagonal + sparse".

As of Dec 2018, Spark only supports ALS for implicit datasets/feedback. See for example [this link](https://spark.apache.org/docs/latest/mllib-collaborative-filtering.html#collaborative-filtering). I guess the reason is that in practice, it is rare to find large explicit datasets that needs the brute force of Spark, whereas implicit datasets tend to be much bigger and common.
