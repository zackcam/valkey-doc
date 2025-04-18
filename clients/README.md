# Clients Documentation

This repo contains the detailed JSON files specifying client libraries. It is used for generating contect for the [clients documentation page](https://valkey.io/clients/)

## JSON Fields
Each JSON file includes general fields as well as boolean feature fields, specifying whether the client supports them or not.

### General Fields 

1. **`description`** - a short despcription of the library, mostly taken from their repos. 
2. **`repo`** - the url to the library's repo.
3. **`installation`** - an installation command from the most used package manager in the respective language.  
4. **`language`** - the programming language in which the library is written.
5. **`package_size`** - the library's unpacked package size, including dependencies. 

### Feature Fields
1. **`read_from_replica`** - The ability to read data from a replica node, which can be useful for load balancing and reducing the load on the primary node. This feature is particularly important in read-heavy applications.

2. **`smart_backoff_to_prevent_connection_storm`** - A strategy used to prevent connection storms by progressively updating the wait time between retries when attempting to reconnect to a Valkey server. This helps to reduce the load on the server during topology updates, periods of high demand or network instability.

3. **`pubsub_state_restoration`** - The ability to restore the state of Pub/Sub (publish/subscribe) channels after a client reconnects. This feature ensures that clients can continue receiving messages after disconnections or topology updates such as adding or removing shards, for both legacy Pub/Sub and sharded Pub/Sub. The client will automatically resubscribe the connections to the new node. The advantage is that the application code is simplified, and doesn’t have to take care of resubscribing to new nodes during reconnects.

4. **`cluster_scan`** - This feature ensures that the user experience and guarantees for scanning a cluster are identical to those for scanning a single node. The SCAN function operates as a cursor-based iterator. With each command, the server provides an updated cursor, which must be used as the cursor argument in subsequent calls. A complete iteration with SCAN retrieves all elements present in the collection from start to finish. If an element exists in the collection at the beginning and remains until the end of the iteration, SCAN will return it. Conversely, any element removed before the iteration begins and not re-added during the process will not be returned by SCAN. A client supporting this feature ensures the scan iterator remains valid even during failovers or cluster scaling (in or out) during the SCAN operation. 

5. **`latency_based_read_from_replica`** - This feature enables reading data from the nearest replica, i.e., the replica that offers the best latency. It supports complex deployments where replicas are distributed across various distances, including different geographical regions, to ensure data is read from the closest replica, thereby minimizing latency.

6. **`AZ_based_read_from_replica`** - This feature enables reading data from replicas within the same Availability Zone (AZ). When running Valkey in a cloud environment across multiple AZs, it is preferable to keep traffic localized within an AZ to reduce costs and latency. By reading from replicas in the same AZ as the client, you can optimize performance and minimize cross-AZ data transfer charges. For more detailed information about this feature and its implementation, please refer to [this link.](https://github.com/valkey-io/valkey/pull/700)

7. **`client_side_caching`** - Valkey client-side caching is a feature that allows clients to cache the results of Valkey queries on the client-side, reducing the need for frequent communication with the Valkey server. This can significantly improve application performance by lowering latency, reducing the network usage and cost and reducing the load on the Valkey server. 

8. **`client_capa_redirect`** - The `CLIENT CAPA redirect` feature was introduced in Valkey 8 to facilitate seamless upgrades without causing errors in standalone mode. When enabled, this feature allows the replica to redirect data access commands (both read and write operations) to the primary instance. This ensures uninterrupted service during the upgrade process. For more detailed information about this feature, please refer to [this link.](https://github.com/valkey-io/valkey/pull/325)

9. **`persistent_connection_pool`** - This feature enables the Valkey client to maintain a pool of persistent connections to the Valkey server, improving performance and reducing overhead. Instead of establishing a new connection for each request, the client can reuse existing connections from the pool, minimizing the time and resources required for connection setup.

10. **`specific_properties`** - An array field to specify any additional language or client specific properties that might not be relevant to all of the other clients (i.e. sync/async support for python clients).   