---
layout: default
title:  "Overview of KV store Interview Problems I"
tag: "coding"
categories: coding interview
---
Over the several months of interviews I frequently encountered problems to design KV store, cache, of various flavors. This post is intended to summarize thought processes to following problems, and to identify any patterns, if possible, to similar problems. For simplicity, I will assume all key values in these problems are integers.

For cache, at least it should support the put(k, v) and get(k) API. Eviction logic is also critical key. 

## LRU

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

## LFU
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
    // frequency to set of keys with such frequency
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

## Snapshot Based KV
This is a phone screen question from a startup, very similar to [Leetcode 1146](https://leetcode.com/problems/snapshot-array/description/).
Key difference is inthe `get()` API, expected API to take both index/key, and a snapshot_id. The output is the value of given index at the time snapshot `snap_id` was taken. 
- A clarifying question worth asking is what if there is no match of `snap_id` of a given index? Assume the qeustions asks to find the maximum closest `snap_id`. 

```java
    public int get(int index, int snap_id) ;
```
Snapshot KV class need to track the current snapshot id, as an always increment key and increment whenver a ` void snap()` API is called.
Data structure choice can be nested map, where the outer layer is hashmap, key is index/key, and value is another map. The key of inner map is snapshot id, and value of inner map is value at snapshot id. 
```java
    private Map<Integer, TreeMap<Integer, Integer>> snapshotCache;
```
Note the implementation of inner map interface can be as simple as hashmap, or a treemap wiht ordering of keys. Note one need to give clear explanation of pros and cons of each choice. A hashmap is fast for set and get value, however to address the snapshot requirements of finding the closest maximum snapshot id, worst case can be O(N) since hashmap provides no ordering of keys. If keys are sorted, it is equivalent to do a binary search over order keys to find the closest snapshot_id. TreeMap will be the better choice since it gives O(log N) time complexity over the binary search. 

```java

    //brute force using linear scan using hashmap
    public int get(int index, int snap_id) {
        HashMap<Integer,Integer> map = arr.get(index);
        int maxKey = Integer.MIN_VALUE;
        int resultKey = maxKey;
        for(Map.Entry<Integer, Integer> e : map.entrySet()){
           int key = e.getKey();
           if(key <= snap_id){
               resultKey = Math.max(resultKey, key);
           } 
        }
        return map.get(resultKey);
    }

    //using treemap and floorEntry() to binary search on snap_id
    public int get(int index, int snap_id) {
        return arr.get(index).floorEntry(snap_id).getValue();
    }
```
Time complexity:


## Time based KV 
This is the [leetcode 981](https://leetcode.com/problems/time-based-key-value-store/). The prmpot is to design a data structure that 
- stores multiple values for the same key 
- able to retrieve the key's value at a certain timestamp.

Specifically, `String get(String key, int timestamp)` returns a value such that set was called previously, with `timestamp_prev <= timestamp`. If there are multiple such values, it returns the value associated with the largest timestamp_prev. If there are no values, it returns "".

The approach is very similar to the snapshot id. Timestamp by naturally is auto incrementing, so we can design the hashmap with key is key string, and value a TreeMap of <timestamp, value> as entries. 

```java
class TimeMap {

    private HashMap<String, TreeMap<Integer, String>> map; // {key, to a hashmap of {timestamp, value pairs}
    
    public TimeMap(){
        map = new HashMap<>();    
    }
    
    public void set(String key, String value, int timestamp){
        if(!map.containsKey(key)){
            map.put(key, new TreeMap<>());
        }
        TreeMap<Integer, String> pairs = map.get(key);
        pairs.put(timestamp, value);
        map.put(key, pairs);
    }
    
    /**
     * get the value closest to the given timestamp of given key
     * if key doesn't exist return empty string 
     */
    public String get(String key, int timestamp){
        if(!map.containsKey(key)){
            return "";
        }
        TreeMap<Integer, String> values = map.get(key);
        
        Map.Entry<Integer, String> entry = values.floorEntry(timestamp);
        
        if(entry != null){
            String val = entry.getValue();
            return val;
        }
        else {
            return "";
        }
    }
}
```

Time complexity: Assume the average inner TreeMap size is N.
- `set()`: standalone `set()` call takes O(logN) since outer hashmap takes O(1) to access and inner treeMap put/get dominates the 
- `get()`: hashmap get(key) takes O(1) time, and finding the `floorEntry` takes O(logN). 
