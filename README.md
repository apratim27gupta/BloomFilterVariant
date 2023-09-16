# BloomFilterVariant
A new Bloom Filter alternative that supports deletion and scaling

FAV Filter
Introduction
Hood is an anonymous social media app that allows users to share their thoughts and opinions anonymously thus allowing them to express themselves more freely. 
As with all social media apps, we face a number of tech challenges, in order to give a seamless experience to our users, while ensuring scale and good unit economics. 

The Problem
The problem we describe here is storing the binary state for a user activity. Examples of which are : Whether a user has liked a post or not, whether this post has been previously seen or not, a user has blocked another user or not, a user has hidden another post or not, and so on and so forth. 
The approach that was previously being followed was storing all this information in a cache or a database, and using this information in real time while rendering a user’s feed:
So removing blocked users’ posts, removing hidden posts, trying to show previously unseen posts etc. 



To minimise latency, we were storing all this information in a redis cache. One downside of which is the cost incurred on storing this huge amount of data. If we want to store blocked users, we would need to store both the blocking as well as the blocked user. 

Solution 1
The first solution we considered was using Bloom Filters. They are extremely space efficient, probabilistic data structures which were perfect for the use cases described above. The operations they allow are : insert an item, and lookup if an item exists. 

So let’s say a customer hides a post, which means they would not want to see it again. When they hide a post , we would insert an entry in the Bloom Filter. (so we would insert the value Ux_Px -> where Ux is the user and Px is the post they wish to hide, we can maintain the same Bloom Filter for all users this way)

Before returning the feed of a user, we would check against the bloom filter, all the posts that can be returned, and if any of them has been hidden, we would get that info from the Bloom Filter and we can therefore remove that post from the feed. 

For understanding Bloom Filters, these are some of the suggested articles : https://brilliant.org/wiki/bloom-filter/, https://www.baeldung.com/cs/bloom-filter

Bloom Filters seemed to be perfect for our use case : they are sublinear, the probability of error can be controlled by varying the size, the number of hash functions, it’s relatively easy to implement the vanilla version etc. 
Redis also provides Bloom Filters in one of its implementations. 

However there were some concerns that we had related to this out of box implementation : 
AWS Elasticache does not provide probabilistic data structures, and while there are redis clients that do the task for us, we were not very comfortable with them. 
Bloom Filters do not support deletion. Our use case is such that we do need deletion capabilities. (You can unblock someone, unhide a post, unlike a post or a comment etc.). Cuckoo filters do provide deletion, but Elasticache does not provide cuckoo filters either. Moreover we do need repeated insertion and deletion of the same data. (You may block, unblock, then block … and so on)
Bloom Filters are not scalable inherently. There has been some research on scalable Bloom Filters and we used the same reference for scaling our bloom filters. (https://gsd.di.uminho.pt/members/cbm/ps/dbloom.pdf) 
Since we are storing strings (that is hashing strings), we would need some way to make the hashing operation efficient. Moreover with scalable Bloom Filters, the number of hash functions increases, hence making it inefficient. 


These concerns made us consider implementing our own bloom filter in elasticache, using Redis Bitmaps. 
We first thought about using cuckoo filters, however the issue with that was if insertion failed for the first two times, it would cause eviction, which would increase our network calls. This is because the Bloom Filter logic in our case is residing on the client side. 

These are some challenges and how we overcame them,  about our implementation of Bloom Filter (aka FAV filter : Farhan, Apratim & Vikas):
Minimise Network Calls. The network calls should ideally be minimal since that adds to latency.  
For deletion, the additional space should be a small fraction of the vanilla Bloom Filter. Else the advantage of Bloom Filter gets diluted to some extent. 
Support deletion to a high level of accuracy. This is the part that is explicitly solved by the FAV filter, which we will explain below. 
Support scaling. For this we decided to go with the approach mentioned in the paper. (https://gsd.di.uminho.pt/members/cbm/ps/dbloom.pdf)
We decided to go with only two hash functions. In order to support, let’s say n hash functions, this is the approach we follow : 
Compute two hash functions Ha & Hb
For n hash functions we go over a loop and compute the ith hash function as : Ha + i*Hb. Where i is the iterator variable and can vary between 0 and n-1. 
This has the same performance as using n different hash functions. (https://www.eecs.harvard.edu/~michaelm/postscripts/tr-02-05.pdf)


How we handle deletions
Before we proceed, it is important to understand why we can’t delete an entry in bloom filter. Consider a bloom filter with 3 hash functions. 

Consider we want to remove the input value in purple. If we unset the bits, then it can cause other values to be removed as well which were colliding with the hash of the value to be deleted. (We consider a value to be in a Bloom filter only if all bits are set)

But it is also important to note that this only happens if there’s a collision while inserting. If there was no collision, then deletion is pretty easy. 
So if we could find a way to track collisions, when we delete, we can be careful in ensuring that we only unset bits where there has been no collision. 
For the above solution, what we can do is, create a bitmap and a Bloom filter of the same size. 

While inserting in the Bloom Filter, we check if the bit is already set (this is done for all the bits to be inserted), if it is, then we set the Collision Tracking Bitmap (for the collision index) to 1. 



E.g. if we are trying to insert “hello”, and let’s assume H1(“Hello”) is 3, and the index 3 is already set in the Bloom Filter.  In which case, we would mark index 3 at the collision Tracking Bitmap to 1. 

The 1 at the Bitmap, tells us that we cannot unset the corresponding bit at the Bloom Filter. This is important because if we do that, then other elements are also affected resulting in false negatives.



For deletions, we simply need to unset all uncollided bits. (i.e. bits at indexes in the CT Bitmap, which are are NOT 1 )

Tradeoff
We had to decide whether we should unset 1 bit for deletion or all possible bits. 
We decided to go for all possible bits because of two reasons : 
When we unset only 1 bit, it is possible that some other insertion would cause that bit to be set again, hence potentially ‘undeleting’ the deleted element. 
Also the abandoned bits (belonging to deleted elements) would fill up space and also cause more collisions. 


Optimisations
One problem with the above solution is the need for 2X memory as compared to a normal Bloom Filter. This can be optimised by forming regions in the Bloom Filter. Each region would refer to n contiguous bits. Whenever there is a collision in any of the bits in a region, we would mark the entire region in the bitmap as 1. This would prevent any further unsetting of bits from that region. While it may seem that only one collision would cause the entire region to be unavailable for deletion, in practice the performance was satisfactory. 

Assume one region consists of 10 bits. So in total we would need N bits (for the filter) + N/10 bits (for the Collision Tracker at Region level Bitmap) for total of 1.1N, which is better than 2N. 

Performance 
In the evaluation of our FAV filter, we assessed its performance with a focus on the false positive rate. The false positive rate, often referred to as the probability of incorrectly identifying an element as present when it’s not. It’s a critical metric in the context of bloom filters as the filter may identify an element as present, even if it's not there and vice versa.

In assessing the false positive rate, we investigated the capability of our filter to identify deleted elements. We wanted to check the accuracy of deleted elements which should be marked as absent in our filter, so we evaluated it by a well-structured set of steps which we will be explaining next time. Our filter gave a performance of false positives rate to 0.4% to 0.6% for deleted elements, underscoring the filter’s robustness in distinguishing deleted elements. 



Network Optimization 
As we’ve told we employed Redis’s Bitfield command to set individual bits corresponding to hash positions. Setting each bit individually would entail a substantial number of network calls, resulting in latency and reduced efficiency. However, we used Redis pipelines to streamline this process. Redis pipeline mechanism allowed us to batch multiple Bitfield commands into a single network request. This approach reduced the number of round trip communications between the client and Redis server resulting in improvement.

Written By:
Farhan Jafri : https://www.linkedin.com/in/farhan-jafri-4a8b3617b/
Apratim Gupta : https://www.linkedin.com/in/apratim-gupta-3a540a118/
Vikas Vimal : https://www.linkedin.com/in/vikasvimal01/



