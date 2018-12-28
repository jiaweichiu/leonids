---
layout: post
title:  "Boost coroutines as Python generators"
categories: [ml]
comments: false
math: true
excerpt: "Boost coroutines can be used to build Python generators."
---

Coroutines are essentially cooperative threading in userspace. The main motivation is that often IO takes some time to complete, and while waiting for completion, we could switch to another coroutine and keep the CPU core busy. Moreover, we want to do it in a lightweight and efficient way. We do not want to use threads because it would lead to expensive **context switches**.

There is a standard algorithm question that asks you to build an iterator over a binary search tree. This is trivial if you can use Boost coroutines. Basically, you are cheating by restoring the stack used in the recursive tree traversal. Note that this example is using Boost coroutines2 which is available at around Boost 1.64. If you are using standard Ubuntu 16.04 packages like me, you would need to install Boost manually.

Here is the DFS code via coroutines2.

```cpp
#include <iostream>

#include <boost/bind.hpp>
#include <boost/coroutine2/all.hpp>

using boost::coroutines2::coroutine;

struct Node {
  Node(int v) : val(v) {}
  int val = 0;
  Node *left = nullptr;
  Node *right = nullptr;
};

// Pre-order traversal, for example.
void dfs(coroutine<int>::push_type &sink, Node *x) {
  if (x->left) {
    dfs(sink, x->left);
  }
  sink(x->val);
  if (x->right) {
    dfs(sink, x->right);
  }
}

int main() {
  Node *x = new Node(10);
  x->left = new Node(5);
  x->left->left = new Node(3);
  x->left->right = new Node(7);
  x->right = new Node(20);
  x->right->left = new Node(15);
  x->right->left->right = new Node(17);
  coroutine<int>::pull_type source(boost::bind(dfs, _1, x));
  for (int x : source) {
    std::cout << x << "\n";
  }
}
```

To compile and run, this might be useful:
```sh
g++ cor_dfs.cc \
-I/usr/local/include \
-L/usr/local/lib \
-std=c++11 -lboost_coroutine -lboost_context

LD_LIBRARY_PATH=/usr/local/lib ./a.out
```

This is reminiscent of Python generators. Here is the Python version of the above:

```python
class Node:
    def __init__(self, value):
        self.left = None
        self.right = None
        self.val = value


def dfs_gen(x):
    if x.left:
        for z in dfs_gen(x.left):
            yield z
    yield x.val
    if x.right:
        for z in dfs_gen(x.right):
            yield z


x = Node(10)
x.left = Node(5)
x.left.left = Node(3)
x.left.right = Node(7)
x.right = Node(20)
x.right.left = Node(15)
x.right.left.right = Node(17)

gen = dfs_gen(x)
for z in gen:
    print(z)
```

Above we are just using coroutines to build "recursive generators". We have not demonstrated any asynchronous IO. This can come in another post.
