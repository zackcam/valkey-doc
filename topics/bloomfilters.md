---
title: "Bloom Filters"
description: >
    Introduction to Bloom Filters
---

In Valkey, the bloom filter data type / commands are implemented in the [valkey-bloom module](https://github.com/valkey-io/valkey-bloom) which is an official valkey module compatible with versions 8.0 and above. Users will need to load this module onto their valkey server in order to use this feature.

Bloom filters are a space efficient probabilistic data structure that allows adding elements and checking whether elements exist. False positives are possible where a filter incorrectly indicates that an element exists, even though it was not added. However, Bloom Filters guarantee that false negatives (incorrectly indicating that an element does not exist, even though it was added) do not occur.

## Basic Bloom commands

* `BF.ADD` adds an item to a bloom filter
* `BF.CARD` returns the cardinality of a bloom filter
* `BF.EXISTS` checks if an item has been added to a bloom filter
* `BF.INFO` returns information about a bloom filter

See the [complete list of bloom filter commands](../commands/#bloom).

## Common use cases for bloom filters

### Fraud detection

Bloom filters can be used to answer the question, "Has this card been flagged as stolen?". To do this, use a bloom filter that contains cards reported as stolen. When a card is used, check whether it is present in the bloom filter. If the card is not found, it means it is not marked as stolen. If the card is present in the filter, a check can be made against the main database, or the purchase can be denied.

### Ad placement / Deduplication

Bloom filters can help advertisers answer the following questions:
* Has the user already seen this ad?
* Has the user already purchased this product?

For each user, use a Bloom filter to store all the products they have purchased. The recommendation engine can then suggest a new product and check if it is present in the user's Bloom filter.

* If the product is not in the filter, the ad is shown to the user, and the product is added to the filter.
* If the product is already in the filter, it means the ad has already been shown to the user and the recommendation engine finds a different ad to show.

### Check if URL's are malicious

Bloom filters can answer the question "is a URL malicious?". Any URL inputted would be checked against a malicious URL bloom filter. 

* If no, then we allow access to the site
* If yes, then we can deny access or perform a full check of the URL

### Check if a username is taken

Bloom filters can answer the question: Has this username/email/domain name/slug already been used?

In this username example, we can use use a Bloom filter to track every username that has signed up. When a new user attempts to sign up with their desired username, the app checks if the username exists in the Bloom filter.

* If no, the user is created and the username is added to the Bloom filter.
* If yes, the app can decide to either check the main database or reject the username.

## Scaling and non scaling bloom filters

The bloom filter data type can act either as a "scaling bloom filter" or "non scaling bloom filter" depending on user configuration.

The difference between scaling and non scaling bloom filters is that scaling bloom filters do not have a fixed capacity, instead they can grow. Non-scaling bloom filters will have a fixed capacity, meaning only a fixed number of items can be inserted to it. Scaling bloom filters consist of a vector of "Sub filters" with length >= 1, while non scaling will only contain 1 sub filter.

When a scaling bloom filter reaches its capacity, adding a new unique item will trigger a scale out and a new sub filter is created and added to the vector of sub filters. This new sub filter will have a larger capacity (previous bloom filter's capacity * expansion rate of the bloom object).

After a non scaling bloom filter reaches its capacity, if a user tries to add a new unique item, an error will be returned

The expansion rate is the rate that a scaling bloom filter's capacity is increased by upon scale out. For example, we have a bloom filter with capacity 100 at creation with an expansion rate of 2. After adding 101 unique items, it will scale out and create a new sub filter with capacity 200. Then, after adding 200 more unique items (301 items total), a new sub filter of capacity 400 is added upon scale out and so on. 

### When should you use scaling vs non-scaling filters

If the capacity (number of items we want to add) is known and fixed, using a non-scaling bloom filter is preferred. Likewise the reverse case, if the capacity is unknown / dynamically calculated, using a scaling bloom filters is ideal.

There are a few benefits for using non scaling filters. A non scaling filter will have better performance than a filter that has scaled out several times (e.g. > 100). Also, non scaling filters in general use less memory for a scaling filter that has scaled out several times to hold the same capacity.

However, to ensure you do not hit any capacity related errors, and want use-as-you-go capacity, scaling is better.

## Bloom properties

* Capacity - The number of unique items that would need to be added before a scale out occurs or (non scaling) before it rejects addition of unique items. 

* False Positive Rate (Error rate) - The rate that controls the probability of bloom check/set operations being false positives. Example: An item addition returning 0 (or an item check returning 1) indicating that the item was already added even though it was not.

* Expansion - This is a property of scalable bloom filters which controls the growth in overall capacity when a bloom filter scales out by determining the capacity of the new sub filter which gets created. This new capacity is equal to the previous filters capacity * expansion rate 

### Advanced Properties

The following two properties can be specified in the `BF.INSERT` command:

* Seed -  This is the key with which hash functions are created for the bloom filter. In case of scalable bloom filters, the same seed is used across all sub filters. This property is only useful if you have a specific 32 byte seed that you want your bloom filter to use. By default every bloom filter will use a random seed.

* Tightening Ratio - This is a property of scalable bloom filters which controls the overall correctness of the bloom filter as it scales out by keeping the actual false positive rate closer to the user requested false positive rate when the bloom filter was created. This is done by using the tightening ratio to set a stricter false positive on the new sub filter which gets created during each scale out. We do not recommend fine tuning this unless there is a specific use case for lower memory usage with higher false positive or vice versa.

### Default bloom properties

These are the default bloom properties along with the commands and configs which allow customizing.

<table width="100%" border="1" style="border-collapse: collapse; border: 1px solid black" cellpadding="8">
<tr>
<th width="20%">Property</th>
<th width="20%">Default Value</th>
<th width="30%">Command Name</th>
<th width="30%">Configuration name</th>
</tr>
<tr>
<td>Capacity</td>
<td>100</td>
<td>BF.INSERT, BF.RESERVE</td>
<td>BF.BLOOM-CAPACITY</td>
</tr>
<tr>
<td>False Positive Rate</td>
<td>0.01</td>
<td>BF.INSERT, BF.RESERVE</td>
<td>BF.BLOOM-FP-RATE</td>
</tr>
<tr>
<td>Scaling / Non Scaling</td>
<td>Scaling</td>
<td>BF.INSERT, BF.RESERVE</td>
<td>BF.BLOOM-EXPANSION</td>
</tr>
<tr>
<td>Expansion Rate</td>
<td>2</td>
<td>BF.INSERT, BF.RESERVE</td>
<td>BF.BLOOM-EXPANSION</td>
</tr>
<tr>
<td>Tightening Ratio</td>
<td>0.5</td>
<td>BF.INSERT</td>
<td>BF.BLOOM-TIGHTENING-RATIO</td>
</tr>
<tr>
<td>Seed</td>
<td>Random Seed</td>
<td>BF.INSERT</td>
<td>BF.BLOOM-USE-RANDOM-SEED</td>
</tr>
</table>


Since bloom filters have a default expansion of 2, this means any default creation as a result of `BF.ADD`, `BF.MADD`, `BF.INSERT` will be a scalable bloom filter. Users can create a non scaling bloom filter using `BF.RESERVE <filter-name> <error-rate> <capacity> NONSCALING` or by specifying `NONSCALING` in `BF.INSERT`. Additionally, the other default properties of a bloom filter creation can be seen in the table above and BF.INFO command response below. These default properties can be configured through configs on the bloom module.

Example of default bloom filter information:

```
127.0.0.1:6379> BF.ADD default_filter item
1
127.0.0.1:6379> BF.INFO default_filter
 1) Capacity
 2) (integer) 100
 3) Size
 4) (integer) 384
 5) Number of filters
 6) (integer) 1
 7) Number of items inserted
 8) (integer) 2
 9) Error rate
10) "0.01"
11) Expansion rate
12) (integer) 2
13) Tightening ratio
14) "0.5"
15) Max scaled capacity
16) (integer) 26214300
```

## Performance

The bloom commands which involve adding items or checking the existence of items have a time complexity of O(n * k) where n is the number of hash functions used by the bloom filter and k is the number of elements being inserted. This means that both BF.ADD and BF.EXISTS are both O(n) as they only operate on one item.

Since performance relies on the number of hash functions, choosing the correct capacity and expansion rate can be important. In case of scalable bloom filters, with every scale out, we increase the number of checks (using hash functions of each sub filter) performed during any add / exists operation. For this reason, it is recommended that users choose a capacity after evaluating the use case / workload to avoid several scale outs and reduce the number of checks.

The other bloom filter commands are O(1) time complexity: BF.CARD, BF.INFO, BF.RESERVE, and BF.INSERT (when no items are provided).

## Monitoring 

To check the server's overall bloom filter metrics, you can use the `INFO BF` or the `INFO MODULES` command. 

Example of `INFO BF` calls in different scenarios:

```
127.0.0.1:6379> INFO BF
# bf_bloom_core_metrics
bf_bloom_total_memory_bytes:0
bf_bloom_num_objects:0
bf_bloom_num_filters_across_objects:0
bf_bloom_num_items_across_objects:0
bf_bloom_capacity_across_objects:0

# bf_bloom_defrag_metrics
bf_bloom_defrag_hits:0
bf_bloom_defrag_misses:0
127.0.0.1:6379> bf.add key value
(integer) 1
127.0.0.1:6379> info bf
# bf_bloom_core_metrics
bf_bloom_total_memory_bytes:384
bf_bloom_num_objects:1
bf_bloom_num_filters_across_objects:1
bf_bloom_num_items_across_objects:1
bf_bloom_capacity_across_objects:100

# bf_bloom_defrag_metrics
bf_bloom_defrag_hits:0
bf_bloom_defrag_misses:0
```

### Bloom filter core metrics

* `bf_bloom_total_memory_bytes`: Current total number of bytes used by all bloom filters.

* `bf_bloom_num_objects`: Current total number of bloom filters.

* `bf_bloom_num_filters_across_objects`: Current total number of sub filters across all bloom filters.

* `bf_bloom_num_items_across_objects`: Current total number of items across all bloom filters.

* `bf_bloom_capacity_across_objects`: Current total capacity across all bloom filters.

### Bloom filter defrag metrics

* `bf_bloom_defrag_hits`: Total number of defrag hits that have occurred on bloom filters.

* `bf_bloom_defrag_misses`: Total number of defrag misses that have occurred on bloom filters.

## Limits

There are two limits a bloom filter faces.

1. Memory Usage Limit:

    The memory usage limit per bloom filter by default is defined by the `BF.BLOOM-MEMORY-USAGE-LIMIT` module configuration which has a default value of 128 MB. If a command results in a creation / scale out causing the overall memory usage to exceed this limit, the command is rejected.

2. Number of sub filters (in case of scalable bloom filters):

    When a bloom filter scales out, a new sub filter is added. The limit on the number of sub filters depends on the false positive rate and tightening ratio. Each sub filter has a stricter false positive, and this is controlled by the tightening ratio. If a command attempting a scale out results in the sub filter reaching a false positive of 0, the command is rejected. 


We have implemented `VALIDATESCALETO` as an optional arg of `BF.INSERT` to help determine whether the bloom filter can scale out to the reach the specified capacity without hitting either limits mentioned above. It will reject the command otherwise.

As seen below, when trying to create a bloom filter with a capacity that cannot be achieved through scale outs (given the memory limits), the command is rejected. However, if the capacity can be achieved through scale out (even with the limits) then the creation of the bloom filter will succeed.

Example:

```
127.0.0.1:6379> BF.INSERT validate_scale_fail VALIDATESCALETO 26214301
(error) ERR provided VALIDATESCALETO causes bloom object to exceed memory limit
127.0.0.1:6379> BF.INSERT validate_scale_valid VALIDATESCALETO 26214300
[]
```

We can use the `BF.INFO` command's `MAXSCALEDCAPACITY` field to find out the maximum capacity that the scalable bloom filter can expand to hold.

```
127.0.0.1:6379> BF.INFO validate_scale_valid MAXSCALEDCAPACITY
(integer) 26214300
```
