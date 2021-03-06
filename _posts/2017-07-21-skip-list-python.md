---
layout: post
title:  "A Simple Python Skip List Implementation"
date:   2017-07-21 20:49:03 -0800
---

In the source codes of [leveldb](https://github.com/google/leveldb), [skiplist](https://en.wikipedia.org/wiki/Skip_list) plays a key role for implementing the Memtable and Immutable Memtable. In this blog, I will briefly introduce some properties of skip list, and provide a simple Python implementation.

## What Is Skip List?
Skip list, developed by [William Pugh](https://en.wikipedia.org/wiki/William_Pugh), is a probabilistic data structure that allows fast search within an ordered sequence of elements. Simply put, it contains multiple ordered linked lists, each jumping at different gaps. For example, you can build a skip list that contains five elements (1 - 5) and three ordered lists.

```
Level 3 Linked list: 1,       4,    NIL
Level 2 Linked list: 1,       4, 5, NIL
Level 1 Linked List: 1, 2, 3, 4, 5, NIL
```

When searching, we start from the highest level list (level 3 in above example). If you want to search for element *4*, with just one hop, you can reach *4* directly by skipping 2 and 3. On the other hand, if the element to search is *5*, then we have to switch to level 2 when we are on element 4, and one extra hop on level 2 yields *5*. Wiki actually provides a more dynamic example:

![]({{ site.url }}/assets/skip_list.gif)

## Why Skip List?
Skip list is an alternative for balanced binary search tree. It is easy to implement and parallelize. In fact, it has been applied in many well-known open source projects such as leveldb, redis and Lucen.

## Time and Space Complexity
I will provides the conclusions without proof. Given **n** elements:

1. Time: the average complexity for find, insert and delete are all **log(n)**.
2. Space: the average space complexity is **O(n)**, while worst case complexity is **O(nlogn)**.

## A Simple Python Implementation
The Python code is heavily influenced by a [C++ implementation](https://codereview.stackexchange.com/questions/116345/skip-list-implementation). It supports three basic operations:

1. Insert
2. Delete
3. Find

The three operations share many things in common. Here we will have a deep dive for **insert** operation.

```python
# All methods inside the SkipList class, then the self in method params.

# level in range of [0, self.max_children - 1].
def random_level(self):
  v = 0
  while random.uniform(0, 1) < self.probability and v < self.max_children - 1:
    v += 1
  return v

# level starts from 0 in this impl.
# node_level == Num of non-Nil objects - 1. If there is no non-nil object, return 0.
def node_level(self, node):
  assert node.__class__.__name__ == 'Node', node.__class__.__name__
  level = -1
  for next_node in node.next_nodes:
    if next_node:
      level += 1
    else:
      break
  return max(level, 0)

def insert(self, key):
  if self.find(key):
    return

  # Deep copy to keep head.next_nodes intact.
  previous_nodes = self.head.next_nodes[:]

  current_max_level = self.node_level(self.head)
  curr = self.head
  for level in range(current_max_level, -1, -1):
    while curr.next_nodes[level] and curr.next_nodes[level].data < key:
      curr = curr.next_nodes[level]
    previous_nodes[level] = curr

  new_node_level = self.random_level()
  assert(new_node_level in range(0, self.max_children)), new_node_level

  new_node = self.create_random_children_node(key, new_node_level)
  if new_node_level > current_max_level:
    for i in range(current_max_level + 1, new_node_level + 1):
      previous_nodes[i] = self.head

  for level in range(new_node_level + 1):
    new_node.next_nodes[level] = previous_nodes[level].next_nodes[level]
    previous_nodes[level].next_nodes[level] = new_node
  assert(any(previous_nodes[:new_node_level + 1])), previous_nodes[:new_node_level + 1]
```

The code is straightforward. When inserting an element to skiplist, we first need to identify the linked list insert locations at different levels. The locations are stored in *previous_nodes* variable. Then, we generate a node with random level by flipping a coin, and insert it accordingly. For complete code snippet, please refer to [github](https://github.com/wang-ye/code/blob/master/python/skip_list.py).
