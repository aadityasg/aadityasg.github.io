---
layout: post
title: Java Synchronized Collections and Thread Safety
---

One week back at work, one of our downstream systems consuming feeds reported missing records. Process logs revealed one of the reporting threads having <i>Concurrent Modification Exception</i>. With first overview, everything appeared in the right place. Every thread shared object was syncronized as required.

The exception was being thrown because one of the thread iterating over a synchronized object while doing removeAll operation and on the other hand another thread adding objects to it.

```
Set<String> synSharedSet = Collections.synchronizedSet(set);

private final Set<String> filterLocalSet(Set<String> localSet) {
	localSet.removeAll(synSharedSet);
}

private final void addToSynSharedSet(String node){
	synSharedSet.add(node);
}
```


Considering it was a SynchronizedSet, it appeared to be an unexpected behaviour. But then remember, there's always a [JavaDoc](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#synchronizedSet-java.util.Set-) to explain every behavior. Below is how Set's [removeAll()](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/util/AbstractSet.java#AbstractSet.removeAll%28java.util.Collection%29) function works.

```
public boolean removeAll(Collection<?> c) {
       boolean modified = false;

       if (size() > c.size()) {
       	  
       	  // Needs an iterator on collection c, to iterate through
	  // each entry and remove all equal objects from
	  // the original set
		
       	  for (Iterator<?> i = c.iterator(); i.hasNext(); )
              modified |= remove(i.next());
       } else {
       	      for (Iterator<?> i = iterator(); i.hasNext(); ) {
                  if (c.contains(i.next())) {
                     i.remove();
                     modified = true;
                  }
              }
       }
       return modified;
}
```

Method [SynchronizedSet()](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/util/Collections.java#Collections.synchronizedSet%28java.util.Set%29) from the Collections API returns an object of class <i>[SynchronizedSet](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/util/Collections.java#Collections.SynchronizedSet)</i> which extends <i>[SynchronizedCollection](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/util/Collections.java#Collections.SynchronizedCollection)</i>. SynchronizedCollection class defines iterator method defined as below:

```
public Iterator<E> More ...iterator() {
       // Must be manually synched by user! It is not synchronized by default.
       return c.iterator();
}
```


So to explain the cause of ConcurrentModificationException observed:

 - One thread is adding data to the SynchronizedSet in a synchronized manner as:

       ```
       public boolean add(E e) {
       	      // Here mutex is the object itself
       	      synchronized(mutex) {return c.add(e);}
       }
	```
 - Another thread is removing data from a local HashSet using removeAll().

 - As removeAll() obtains the iterator over the synSharedSet (which is not synced as shown earlier), while the other thread is adding data to the set. Thus resulting into ConcurrentModificationException.


Solution to the problem: Synchronize evey access to the iterator. As all other methods of the SynchronizedSet are obtaining lock on the object itself, before obtaining iterator one needs to obtain lock on the object to ensure thread safety. After doing so the code would look like:


```
Set<String> synSharedSet = Collections.synchronizedSet(set);

private final Set<String> filterLocalSet(Set<String> localSet) {
	//Obtain lock on the synSharedSet itself before obtaining its iterator
	synchronized(synSharedSet) {
	        localSet.removeAll(synSharedSet);
	}
}

private final void addToSynSharedSet(String node){
	synSharedSet.add(node);
}
```