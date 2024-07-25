---
layout: default
title:  "Overview of KV store Interview Problems (II)"
tag: "coding"
categories: coding interview
---
## Transactional KV store
This problem is from a [leetcode discussion post](https://leetcode.com/discuss/interview-question/279913/bloomberg-onsite-key-value-store-with-transactions), and as inspired as the transactional shell [blog post](https://www.freecodecamp.org/news/design-a-key-value-store-in-go/). The transaction is similar as  database transaction. Followups include support single and multiple rollbacks, similar to Git commits of transactions. Using built-in map/dictionary container works for this part. 

The problem starts with implementation of standard in memory store to support get, put, and delete, then following with the support of transaction and three new APIs: `begin`, `commit` and `rollback`.  

For transaction begin will begin() call to create a new context, and end the context with commit(). In between, any operations to access or modify the data store will not persist.
```
db.set(key0, val0)
db.get(key0) => return val0
db.begin()
db.get(key0)
db.set(key1, val1)
db.get(key1) => return val1
db.commit()
db.get(key1) => return val1

Sample sequence 2:
db.begin()
db.set(k0, v0)
db.get(k0) =>retrun v0
db.rollback()
db.get(k0) =>error as k0 value hasn't commit
```
We introduce new data structure, a separate dictionary to store data of current transaction. Assuming key and values are both string, we build on top of part 1. 

```java
public class TransactionStore{
    Map<String, String> datastore;

    public void set(String key, String val){

    }

    public String get(String key){

    }

    public void delete(String key){

    }

}
'''

Now we add new hashmap to keep track of KV pairs during a transaction. 

```java
 public class TransactStore {
        
        Map<String, String> store;
        Map<String, String> transactionStore;
        
        public TransactStore() {
            store = new HashMap<>();
        }
        
        //Add a new transaction and push to stack
        public void begin(){
            transactionStore = new HashMap<>();
        }
        
        //Retrieve from KV store 
        //return null if store is empty of 
        //doesn't contain key
        public String get(String key){
            if(transactionStore!= null){
                if(transactionStore.containsKey(key)){
                    return transactionStore.get(key);
                }
            }
            if(store.isEmpty() || !store.containsKey(key)){
                return null;
            }
            return store.get(key);
        }
        
        //Set the given key-value pair
        public void set(String key, String val){
            if(transactionStore == null){
                store.put(key, val);
                return;
            }
            transactionStore.put(key, val);
        }
        
        // commmit all modifications in current transaction and 
        // end the transaction
        public void commit(){
            if(transactionStore == null || transactionStore.isEmpty()){
                //exception handling, cannot commit empty transactions
                return;
            }
            
            store.putAll(commitStore);
            transactionStore.clear();
            transactionStore = null;
        }
        
        // Discard current transaction and close it
        public void rollback(){
            if(transactionStore == null || transactionStore.isEmpty()){
                //exception handling, cannot rollback
                return;
            }
            // Discard current transaction map
            transactionStore.clear();
            transactionStore= null;
            
        }
        
        //Delete given key
        public void delete(String key){
            if(transactionStore== null || transactionStore.isEmpty()){
                store.remove(key);
                return;
            }
            //Get key value pair from current stack and remove
            transactionStore.remove(key);
        }
    }
```

Since Java hash map operates on object, I distinguish when `transactionStore`  is Null and `transactionStore` is created but has no key-value pairs. The former means current store is outside a transaction, while the latter means data store is within a transaction and it is possible no modifications are made. 

Part 3 improves to support nested transaction, meaning a child transaction spawned within a transaction inherits all variables and states from parent transaction. 

- Once commit is called on parent transaction all following children transactions will be committed
- rollback: rollback on parent transaction will discard all following ones,  while rollback on current transaction will not reflect on parent transaction

The hierarchical structure of nested transactions, as well as the order of starting and returning back one transact resembles function calls, and thus the push and pop of a stack. I approach this problem with a stack of `transactionStore`.  To begin transaction is to create one stack of transaction and copy all key value pair of top of transaction stack. To rollback means to pop off current stack.

```java
class TransactStore {
    
    Map<String, String> store;
    //New
    //Every stack frame represents key value pairs in one transaction
    Stack<Map<String, String>> transactionalStack;
    
    public TransactStore() {
        store = new HashMap<>();
        transactionalStack = new Stack<>();
    }
    
    //Add a new transaction and push to stack
    public void begin(){
        transactionalStack.push(new HashMap<String, String>());
    }
    
    //Retrieve from KV store, return null if store is empty of 
    //doesn't contain key
    public String get(String key){
        if(store.isEmpty() || !store.containsKey(key)){
            return null;
        }
        return store.get(key);
    }
    
    public void set(String key, String val){
        if(transactionalStack.isEmpty()){
            store.put(key, val);
            return;
        }
        Map<String, String> currTransaction = transactionalStack.peek();
        currTransaction.put(key, val);
    }
    
    public void commit(){
        if(transactionalStack.isEmpty()){
            //exception handling, cannot commit empty transactions
            return;
        }
        Map<String, String> curr = transactionalStack.peek();
        transactionalStack.pop();
        
        store.putAll(curr);
    }
    
    public void rollback(){
        if(transactionalStack.isEmpty()){
            //exception handling, cannot rollback
            return;
        }
        //POP and discardcurrent transaction stack and return to previous stack
        transactionalStack.pop();
        
    }
    public void delete(String key){
        if(transactionalStack.isEmpty()){
            store.remove(key);
        }
        //Get key value pair from current stack and remove
        transactionalStack.peek().remove(key);
    }
}
    
```
Time Complexity:

O(1) for set, get and delete

To summarize, part 2 and 3 build on top of a standard hashmap. The key is to introduce new hashmap and how to represent and modify transaction.
## Expiry Priority Cache
This is a screen problem I came across at Tesla, similar to [this post.](https://leetcode.com/discuss/interview-question/1114869/Tesla-Phone-screen/880661) 

The requirements for the problem is as follows:

```
Implement a key-value cache with capacity which include:

 Expire Time - entry becomes expire if the expirationTime < currentTime
 (assume get can current system clock. Cache data structure don't track the time)

 Priority -lower priority cache entry will be evicted before higher entry

 LRU - keep track of the least recently entry(both get and set). Evict LRU entry when no expired entry
 and more than one entries with the same lowest priority).

 Cache eviction strategy:
 1. evict an expired cache entry first. Any expired entry is suffice to remove. 
 2. If no expired entry, select the lowest priority and evict.
 3. If no expired entry and multiple entries with the same priority, evict the least recently used among them.

 TODO: Design a cache data structure with a given capacity and implement API to support above requirements. 
 - Public Cache(int capacity)
 - int get(String key)
 - void set(String key, int value, int priority, int expirationTime)

 This data structure is expected to hold large data sets and should be as efficient as possible.

 Example
 Cache c=newCache(3) //cache capacity=3c.set("A",1,5,100 );//key=A,value=1, priority =5, expirationTime=100(in second)
 c.set("B" 2,15,3 );
 c.set("C" 35,10);
 Time +=5; // simulate sleep for 5 sec
 c.set("D",4,1,15 ); // should evict B because 3<5 and B expired// now cache has {"A" “C" "D"}
 c.get("A"); // return 1c.set("E"5,5,150 );// should evict A,LRU
```
 My focus is to skew faster get(key) and eviction. 

### 1st approach:
Lets break down the restrictions and what data structures.  The first order is through expiration time, since we only need one expired entry if there more than one candidates. We can add a heuristics to remove the oldest expiry entry. 

Let say we can use a priorityqueue min heap on expiration time, the peak of heap is the oldest time. If the heap peak hasn’t expired then no entries expire. If the peak expires then we remove it. 

The second metric is priority. There is also LRU component coming in the picture to break tie if lowest priority keys are more than one. We can construct a data structure of ordered map(TreeMap in Java), where key is the priority and values are entries with the same priority. Further more we can leverage the LRU abstraction and its implementation  [in leetcode](https://leetcode.com/problems/lru-cache/description/) to track LRU. 

For us to conveniently describe entry, we can construct an object/struct to represent cache entry.
## Reflection
- Understand the problem and asking any clarifying questions are crucial in success. Unlike many other algorithms-heavy coding problems, above design problems need more communications about thought process to address the cache features and describe what data structures and how chosen data structures address these requirements.
- Try and clarify any confusions during the interview. For interactive and open-ended problems, one may waste time working on a solution trivial or completely incorrect from the requirements from the interviewer. 
- Practice "communicating over pseudocode/typing". I believe success of solving above problems rely more on effectiveness of thought process. I was not comfortable typing and talking together in a virtual interview at first, and I realized how much time I have spent not producing effective code in a short time. Thought process is important but in the end interviewers need to see some code as deliverables. My personal way is to jot down key algorithms/data structures and reasoning behind it, and highlevel todos on implementation. For instance, I will jot down I need hashmap and doubly linked list, and why it will 
- Be very comfortable on common data structures in your interviewing programming language and common APIs and their time complexity. If possible, getting familiar with implementation of each data structure. For example, difference and time complexity of Java `HashMap` and `TreeMap`, C++ `std::unordered_map` and `std::map`.
- Since 2023/2024, I have observed increaseing difficulties in coding challenges. The challenge is not necesarily on difficult algorithms or tricky math theorems or bit manupulations, but more on a combinations of code completion, code speed, time to reach a well-organized solution and effective communications to interviewer. Many problems are framed as follow ups to easier first problem or multiple series. I experienced many design-like problems in both screen and onsite, so practicing all above is worthwhile. Different companies may have different interview expecations or emphasis, and it is perfectly fine to ask and clarify any requirements in advance(how many problems each round, focusing on thought process or completion, etc).
