---
layout: post
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
Following the leetcode [problem prompt](https://leetcode.com/problems/lfu-cache) and API restrictions as follows
- LFU cache constructor takes an arugment of int capacity
- LFU maintains a user counter for each key to track frequency.
- `int get(key)` to obtain the value of given key
- `void put(key, value)` set thew new (key,value) entry inside cache. If full, eviction policy follows choosing the least frequently used key based on the user counter. If multiple candidates, following LRU policy to choose the candidate among these keys. 

The above requirements bring the constructor and data structures of how 
```java
    private int minFreq; // user counter
    private int cap;  // capacity
```
Next, we need a hashmap to store key-value pairs. We design the cache to include the frequency value as a compounded value pairs

```java
    // Track key, <frequency, cache value> look up by the key
    private HashMap<Integer, Pair<Integer,Integer>> keyToFreqValuePairMap;
```
As the problem statements hint, need LRU policy to elect the evicted key more than one LFU keys. One may leverage `LinkedHashSet` in Java as the underlying implementation is doubly linked list. Another option is to use `ArrayDeque`.
```java
    // frequencyt to set of keys with such frequency
    private HashMap<Integer, LinkedHashSet<Integer>> frequencyList;
```

For `get(key)`,
- attempt to obtain value and its frequency given key from the `keyToFreqValuePairMap`. 
- increment frequnecy related to key
-- remove key from list of keys by frequecy from `frequencyList`
-- clean up frequencyList if no such frequency exists inside cache
-- if the removed key is minFreq, update the minFreq

For `put(key, value)`

For cache eviction, use the iterator `next()` API call from the value of LinkedHashSet to obtain the next candidate to be evicted.
- Get the frequency-value pair from the `keyToFreqValuePairMap` if input key already exists, update  new value, update frequency and return
- otherwise. First check if capacity is reached. Eviction is reached, otherwise, this is a new key , set its freq to 1 and  reset minFreq = 1, insert the key to cache

# Snapshot Based KV

This is a phone screen question from a startup, very similar to [Leetcode 1146](https://leetcode.com/problems/snapshot-array/description/).
Key difference is inthe `get()` API, expected API
```java
    public int get(int index, int snap_id) ;
```
From this API the cache needs 

## Time based cache
A very similar problem

# Rank Cache

# Transactional KV store

# Expiry Priority Cache

# Reflection
- Understand the problem and asking any clarifying questions are crucial in success. Unlike many other algorithms-heavy coding problems, above design problems need more communications about thought process to address the cache features and describe what data structures and how chosen data structures address these requirements.
- Try and clarify any confusions during the interview. For interactive problems like
- Practice "communicating over pseudocode/typing". I believe success of solving above problems rely more on effectiveness of thought process. I was not comfortable typing and talking together in a virtual interview at first, and I realized how much time I have spent not producing effective code in a short time. Thought process is important but in the end interviewers need to see some code as deliverables. My personal way is to jot down key algorithms/data structures and reasoning behind it, and highlevel todos on implementation
For instance, I will jot down I need hashmap and 
- Be very comfortable on common data structures in your interviewing programming language and common APIs and their time complexity. If possible, getting familiar with implementation of each data structure. For example, difference and time complexity of Java `HashMap` and `TreeMap`, C++ `std::unordered_map` and `std::map`.
- Since 2023/2024, I have observed increaseing difficulties in coding challenges. The challenge is not necesarily on difficult algorithms or tricky math theorems or bit manupulations, but more on a combinations of code completion, code speed, time to reach a well-organized solution and effective communications to interviewer. Many problems are framed as follow ups to easier first problem or multiple series. I experienced many design-like problems in both screen and onsite, so practicing all above is worthwhile. Different companies may have different interview expecations or emphasis, and it is perfectly fine to ask and clarify any requirements in advance(how many problems each round, focusing on thought process or completion, etc).
