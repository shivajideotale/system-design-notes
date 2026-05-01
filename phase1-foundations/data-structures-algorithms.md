# Data Structures & Algorithms for System Design

This document covers advanced data structures and algorithms crucial for scalable system design, focusing on distributed systems applications.

## Consistent Hashing

A distributed hashing scheme that minimizes reorganization when nodes are added or removed.

### How It Works
- Hash space is a ring (0 to 2^32-1)
- Nodes and keys mapped to ring positions
- Each key assigned to next node clockwise
- Virtual nodes reduce uneven distribution

### Advantages
- Minimal data movement during scaling
- Load balancing across nodes
- Fault tolerance

### System Design Use Cases
- Distributed caches (Redis Cluster, Memcached)
- Database sharding (Cassandra, DynamoDB)
- Load balancers

### Code Example (Simplified)
```java
public class ConsistentHash {
    private SortedMap<Integer, String> ring = new TreeMap<>();
    
    public void addNode(String node) {
        int hash = node.hashCode();
        ring.put(hash, node);
    }
    
    public String getNode(String key) {
        int hash = key.hashCode();
        if (ring.isEmpty()) return null;
        
        SortedMap<Integer, String> tailMap = ring.tailMap(hash);
        return tailMap.isEmpty() ? ring.get(ring.firstKey()) : tailMap.get(tailMap.firstKey());
    }
}
```

## Bloom Filters

A space-efficient probabilistic data structure for testing set membership.

### How It Works
- Array of bits, initially all 0
- Multiple hash functions
- For insertion: Set bits at hash positions
- For lookup: Check if all bits are set

### Properties
- **False Positives**: May incorrectly report presence
- **No False Negatives**: If says absent, definitely absent
- **Space Efficient**: Much smaller than hash sets
- **No Deletion**: Cannot remove elements

### System Design Use Cases
- Caching layer optimization (avoid cache misses)
- Database query optimization
- Web crawling (URL deduplication)
- Spell checkers

### Trade-offs
- Tunable false positive rate
- Fixed size, no growth
- Memory vs accuracy balance

## LSM Trees (Log-Structured Merge Trees)

A data structure optimized for write-heavy workloads, used in modern databases.

### Components
- **MemTable**: In-memory sorted structure
- **SSTable**: Immutable sorted files on disk
- **WAL (Write-Ahead Log)**: Durability guarantee

### Write Path
1. Write to WAL
2. Insert into MemTable
3. When MemTable full, flush to SSTable
4. Merge SSTables during compaction

### Read Path
1. Check MemTable
2. Check SSTables from newest to oldest
3. Bloom filters optimize lookups

### Advantages
- High write throughput
- Efficient range queries
- Crash recovery via WAL

### System Design Use Cases
- NoSQL databases (LevelDB, RocksDB, Cassandra)
- Time-series databases (InfluxDB)
- Search engines (Lucene)

### Comparison with B-Trees
- LSM: Write-optimized, eventual consistency
- B-Tree: Read-optimized, immediate consistency

## Key Takeaways

- Consistent hashing enables scalable, fault-tolerant distribution
- Bloom filters provide fast, space-efficient membership testing with controlled false positives
- LSM trees optimize for high write throughput at the cost of read amplification

## Implementation Considerations

### Consistent Hashing
- Virtual nodes for better distribution
- Replication for fault tolerance
- Hotspot mitigation strategies

### Bloom Filters
- Optimal number of hash functions: k = (m/n) * ln(2)
- False positive rate formula: (1 - e^(-kn/m))^k
- Sizing: m = - (n * ln(p)) / (ln(2))^2

### LSM Trees
- Compaction strategies (size-tiered, leveled)
- Write amplification vs read amplification trade-offs
- Memory management for MemTable

## Resources

- [Consistent Hashing Paper](https://www.akamai.com/us/en/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf)
- [Bloom Filter Calculator](https://hur.st/bloomfilter/)
- [LSM Tree Paper](https://www.cs.umb.edu/~poneil/lsmtree.pdf)
- [LevelDB Documentation](https://github.com/google/leveldb)