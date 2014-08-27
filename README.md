#Simple Nodestore Proposal

##Insights:

* Keys are fixed length and don't compress well due to high entropy
* Values are variable length and compress well
* Keys and values are write once, read many
* There is no natural ordering to keys, but keys and values written together in chronological order are more likely to read in the same order.

##Design

Three file types exist in a folder. The folder comprises a full nodestore. An in-memory hash table exists which tracks the most recent keys and values. When it gets full a new one is created. The old one is written to disk, but continues to serve requests until it it is fully committed to disk.

The set of three files are named based on the first ledger for which a key and value is entered. Eg:
```
0000000000.keys
0000000000.values
0000000000.bloom
0000000534.keys
0000000534.values
0000000534.bloom
...
4294967000.keys
4294967000.values
4294967000.bloom
```

As the ledger state expands or transaction submission rate increases the sets of three files will become closer together, but will remain ordered.

The three file types could potentially be merged into a single file, but this would make the format less simple :-)

###Keys

Simple immutable sorted uncompressed store of keys with a Values file frame locator occupying 4 bytes. A frame is up to 65536 bytes in length and there are up to 65536 frames in a Values file. The frame is compressed and the Offset in Frame refers to the start of the value when uncompressed.

|Key (32 bytes)|Frame (2 bytes)|Offset in Frame (2 bytes)|
|--------------|---------------|-------------------------|
|37F86DD3722863...4984DDA9C64|0| 0|
|4149AB11F9AE0F...5BA5599DB90|0|557|
|504CC68BFB0374...106A42FC9E9|0|1114|
|...|...|...|
|5E707D710B2035...35DC762CA76|0|63000|
|916828F070A8B1...98D9C912513|1|0|
|A8B4EFBF94E0E3...3FEA6CE926E|1|127|
|...|...|...|

###Values
Simple immutable compressed store of values in key order. Compressed using [Snappy frame compression](https://snappy.googlecode.com/svn/trunk/framing_format.txt). Format is as follows:
```
Number of Frames
Offset of Frame #0
Offset of Frame #1
Offset of Frame #2
...
Frame #0 type 0x00 (compressed)
CRC-32C value
Compressed contents (probably about 50% of 65536 bytes)
Frame #1 type 0x00 (compressed)
...
```
###Bloom Filters
Simple immutable file containing compressed serialized form of a bloom filter which contains k hashes of all the keys in the associated Keys file. It has a fixed length to be decided on based on the expected number of sets of threee files that will exist for the lifetime of the server.

## Operation

All bloom filters are loaded into memory. File descriptors for opened keys and values files are kept open in a LRU cache.

### Write Value for Key
* Add key to in-memory hash table.
* If key is close to overflowing hash table limit continue else return.
* Create new current hash table for next write operation.
* Dump hash table to disk.
* Delete hash table from memory.

### Dump Hash Table
* Sort keys and write to disk.
* Add each key to bloom filter and write to disk.
* Compress and write values to disk in key order in Values format.

###Get Value for Key
* Check both current in-memory hash table and being-committed hash table for key. If exists return value.
* Find bloom filters for which key provides a positive. This should usually be a single filter.
* Binary search key file for matching key and next key.
* Open values file (if not already open) and determine frame offset using matching key frame index.
* Decompress frame at frame offset and the use matching key frame offset and next key frame offset to extract stream of uncompressed value.

###Range Over All Keys In Order
* Open all key files and load all the first keys into a priority queue ordered lexically.
* Take the top key and add the next key from the source key file.
* Get value for key.
* Continue until priority queue is empty.

## Maintenance

###Pruning
* Use the nodestore to load a historic ledgerstate, say 1000 ledgers behind the current ledger. Write it fully to disk with the ledger number as the filename. This set of three files will probably be bigger than other sets. 
* Delete all sets of three files with a ledger number below the target ledger.



