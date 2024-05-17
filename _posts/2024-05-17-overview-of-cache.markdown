---
layout: single
title:  "Overview of KV store Interview Problems"
---

Over the several months of interviews I frequently encountered problems to design KV store, cache, of various flavors. This post is intended to summarize thought processes to following problems, and to identify any patterns, if possible, to similar problems. For simplicity, I will assume all key values in these problems are integers.

For cache, at least it should support the put(k, v) and get(k) API. Eviction logic is also critical key. 

# LRU

LRU cache, from the leetcode quesiton prompt, takes a capacity argument for maximum size and, two API get and put() in average O(1) time complexity.  Key challenge is to maintain the least-recently used  

- eviction policy when cache is full.
- access or update to a key already exists in cache.

Modeling the order of access can use a linked list of key-value pairs where the head is the least recently used, and the tail is the most recently used. Linked list offers O(1) on insert and remove. However LinkedList does not provide O(1) time complexity to update cache by the correct index whenever a key is recently accessed by either get or put, so we need a another data structure to bookkeep current keys inside the cache. LinkedList  offers O(1) to remove or insert a node, and we can take advantage of the doubly linked list with both prev and next pointer.
```java
    // Linked List node class 
    class Node{
        Node prev;
        Node next;
        int key;
        int value;
        
        public Node(int k, int v){
            this.key = k;
            this.value = v;
        }
    }

		
    private int capacity;//capacity of the 
    private HashMap<Integer, Node> map = new HashMap<>(); // integer key to the linked node that stores the key 
    private Node head = new Node(-1, -1); // head node, beginning of the 
    private Node tail = new Node(-1, -1); // end of linked list
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.head.next = this.tail;
        this.tail.prev = this.head;
    }
```

The head is the least recently used key,  and the tail should be most recently accessed element. 

To maintain readability we can add two new functions to handle removing node from the lined list and update a node to the end

```java
    // append curr to the tail from the most recent access
    private void update(Node curr){
        curr.next = tail;
        tail.prev.next = curr;
        curr.prev = tail.prev;
        tail.prev = curr;
    }

    //Remove key from old index from the most recent access(put/get)
    private void remove(Node curr){
        curr.prev.next = curr.next;
        curr.next.prev=  curr.prev;
    }
```

With the help of data structure and two utility functions, we can implement the required APIs as follows
```java
 public int get(int key) {
        if(!map.containsKey(key)){
            return -1;
        }  
        Node curr = map.get(key);
        
        //update the previous AND next node pointer because once
        //current node is accessed, it will be removed and moved to the tail
        //to maintain LRU
        remove(curr);
        update(curr); // move to tail
        
        return curr.value;
    }
    
    public void put(int key, int value) {
        // if key already exists in the cache
        // node with the key is accessed so need to remove
        if(map.containsKey(key)){
            Node oldNode = map.get(key);
            remove(oldNode);
        }
        Node newNode = new Node(key, value);
        map.put(key, newNode);
        update(newNode);     
        //exceeds capacity, need evict LRU key, that is the node at the beggining
        if(map.size() > capacity){
            Node nodeToRemove = head.next;
            remove(nodeToRemove);
            map.remove(nodeToRemove.key);
        }
    }
```

# LFU

Least frequently used mechanism, as the name suggests, requires keeping track of frequencies of each key.
Following the leetcode [problem prompt][] and API restrictions as follows
- LFU cache constructor takes an arugment of int capacity
- LFU maintains a user counter for each key.
- `int get(key)` to obtain the value of given key
- `void put(key, value)` set thew new pair of 

# Snapshot  Based KV

This is a popular phone screen question from a startup,very similar to Leetcode 1146.

# Rank Cache

# Transactional KV store

# Expiry Priority Cache

# Review
- Understand the problem is crucial with communication with the interviewer. 
- 