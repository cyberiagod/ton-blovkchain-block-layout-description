### Block layout

This article presents the block scheme used in the TON blockchain in combination with data structures described separately in tblch.pdf to create full description of the shardchain block. In addition to TL-B schemes which define the representation of a shardchain block by a tree of cells, the exact serialization formats for the resulting bags (collections) of cells that are needed to represent a shardchain block as a file.
Masterchain blocks are similar to shardchain blocks, but have some special features. Necessary modifications are discussed below.

### Common parts of the block layout for all workchains.

So that different work chains do not conflict with each other, common components were allocated for all blocks:
• OutMsgQueue, the outbound message queue of a shardchain, which is scanned by neighboring shardchains for messages addressed to them.
• The outer structure of InMsgDescr as a hashmap with 256-bit keys equal to the hashes of the imported messages. The descriptors of incoming messages themselves may differ in structure.                                                                                                                                                        • Some fields in the block header identifying the shardchain and the block, along with the paths from the block header to the other information indicated in this list.
• The value flow information.

### Shardchain block layout

The Shardchain block layout consists of a header, a status, a list of transactions and a signature:
• *ShardAccounts*, the split part of the shardchain state containing the state of all accounts assigned to this shard.
• *OutMsgQueue*, the output message queue of the shardchain.
• *SharedLibraries*, the description of all shared libraries of the shardchain (for now, non-empty only in the masterchain).
• The logical time and the unixtime of the last modification of the state, total balance of the shard, a hash reference to the most recent masterchain block, indirectly describing the state of the masterchain and, through it, the state of all other shardchains of the TON Blockchain.

### Components of a shardchain block.
A shardchain block must contain:
• A list of validator signatures, which is external with respect to all other contents of the block.
• BlockHeader, containing general information about the block.
• Hash references to the immediately preceding block or blocks of the same shardchain, and to the most recent masterchain block.
• *InMsgDescr* and *OutMsgDescr*, the inbound and outbound message descriptors.
• *ShardAccountBlocks*, the collection of all transactions processed in the block along with all updates of the states of the accounts assigned to the shard. This is the split part of the shardchain block.
• The value flow, describing the total value imported from the preceding blocks of the same shardchain and from inbound messages, the total value exported by outbound message, the total fees collected by validators, and the total value remaining in the shard.
• A Merkle update of the shardchain state. Such a Merkle update contains the hashes of the initial and final shardchain states with respect to the block, along with all new cells of the final state that have been created while processing the block.

## TL-B scheme for the shardchain state.
The shardchain state is serialized according to the following TL-B scheme:
```
ext_blk_ref$_ start_lt:uint64 end_lt:uint64
	seq_no:uint32 hash:uint256 = ExtBlkRef;

master_info$_ master:ExtBlkRef = BlkMasterInfo;

shard_ident$00 shard_pfx_bits:(## 6)
	workchain_id:int32 shard_prefix:uint64 = ShardIdent;

shard_state shard_id:ShardIdent
	out_msg_queue:OutMsgQueue
	total_balance:CurrencyCollection
	total_validator_fees:CurrencyCollection
	accounts:ShardAccounts
	libraries:(HashmapE 256 LibDescr)
	master_ref:(Maybe BlkMasterInfo)
	custom:(Maybe ^McStateExtra)
	= ShardState;
```

The field `custom` is usually present only in the masterchain and contains all the masterchain-specific data. However, other workchains may use the same cell reference to refer to their specific state data.

###  Shared libraries description

Shared libraries currently can be present only in masterchain blocks. They are described by an instance of `HashmapE(256, LibDescr)`, where the 256-bit key is the representation hash of the library, and `LibDescr` describes one library:

```
shared_lib_descr$00 lib:^Cell publishers:(Hashmap 256 True)
	= LibDescr;
```

Here `publishers` is a hashmap with keys equal to the addresses of all accounts that have published the corresponding shared library. The shared library is preserved as long as at least one account keeps it in its published libraries collection.

### TL-B scheme for an unsigned shardchain block

The precise format of an unsigned shardchain block is given by the following TL-B scheme:

```
block_info version:uint32
	not_master:(## 1)
	after_merge:(## 1) before_split:(## 1) flags:(## 13)
	seq_no:# vert_seq_no:#
	shard:ShardIdent gen_utime:uint32
	start_lt:uint64 end_lt:uint64
	master_ref:not_master?^BlkMasterInfo
	prev_ref:seq_no?^(BlkPrevInfo after_merge)
	prev_vert_ref:vert_seq_no?^(BlkPrevInfo 0)
	= BlockInfo;
prev_blk_info#_ {merged:#} prev:ExtBlkRef
	prev_alt:merged?ExtBlkRef = BlkPrevInfo merged;

unsigned_block info:^BlockInfo value_flow:^ValueFlow
	state_update:^(MERKLE_UPDATE ShardState)
	extra:^BlockExtra = Block;

block_extra in_msg_descr:^InMsgDescr
	out_msg_descr:^OutMsgDescr
	account_blocks:ShardAccountBlocks
	rand_seed:uint256
	custom:(Maybe ^McBlockExtra) = BlockExtra;
```

The field `custom` is usually present only in the masterchain and contains all the masterchain-specific data. However, other workchains may use the same cell reference to refer to their specific block data.

### Description of total value flow through a block

The total value flow through a block is serialized according to the following TL-B scheme: 

```
value_flow _:^[ from_prev_blk:CurrencyCollection
	to_next_blk:CurrencyCollection
	imported:CurrencyCollection
	exported:CurrencyCollection ]
	fees_collected:CurrencyCollection
	_:^[
	fees_imported:CurrencyCollection
	created:CurrencyCollection
	minted:CurrencyCollection
	] = ValueFlow;
```

Recall that `_:ˆ[. . . ]` is a TL-B construction indicating that a group of fields has been moved into a separate cell. The last three fields may be non-zero only in masterchain blocks.  

### Signed shardchain block

A signed shardchain block is just an unsigned block augmented by a collection of validator signatures:

```
ed25519_signature#5 R:uint256 s:uint256 = CryptoSignature;

signed_block block:^Block blk_serialize_hash:uint256
	signatures:(HashmapE 64 CryptoSignature)
	= SignedBlock;
```

The **serialization hash** `blk_serialize_hash` of the unsigned block block is essentially a hash of a specific serialization of the block into an octet string . The signatures collected in signatures are Ed25519-signatures made with a validator’s private keys of the sha256 of the concatenation of the 256-bit representation hash of the block block and of its 256-bit serialization hash `blk_serialize_hash`. The 64-bit keys in dictionary `signatures` represent the first 64 bits of the public keys of the corresponding validators.  

### Serialization of a signed block.

The general procedure for serialization and block signing can be described as follows:

1. An unsigned block B is generated, transformed into a complete bag of cells, and serialized into an octet string  S_B.

2. Validators sign the 256-bit combined hash

$$
H_B := sha256(Hash_∞ (B).Hash_M(S_B))
$$

​		of the representation hash of B and of the Merkle hash of its serialization S_B.

	3. A signed shardchain block B˜ is generated from B and these validator signatures as described above.

4. This signed block B˜ is transformed into an incomplete bag of cells, which contains only the validator signatures, but the unsigned block itself is absent from this bag of cells, being its only absent cell.

5. This incomplete bag of cells is serialized, and its serialization is prepended to the previously constructed serialization of the unsigned block.

The result is the serialization of the signed block into an octet string. It may be propagated by network or stored into a disk file.

### Masterchain block layout
Masterchain is a special chain in the TON Blockchain that is responsible for maintaining the state of the entire blockchain network. It stores information about all accounts and smart contracts in the system and serves as a source of truth for all shardchains. In this sense, the Masterchain can be considered the backbone of the TON Blockchain.  

### Additional components present in the masterchain state

In addition to the components listed earlier, the state of the main circuit must contain:

• **ShardHashes** — Describes the current shard configuration, and contains the hashes of the latest blocks of the corresponding shardchains.

• **ShardFees** — Describes the total fees collected by the validators of each shardchain.

• **ShardSplitMerge** — Describes future shard split/merge events. It is serialized as a part of ShardHashes.

• **ConfigParams** — Describes the values of all configurable parameters of the TON Blockchain.

### Additional components present in masterchain blocks

Each master chain block must contain:

• `ShardHashes` — Describes the current shard configuration, and contains the hashes of the latest blocks of the corresponding shardchains. (Notice that this component is also present in the masterchain state.)

### Description of ShardHashes

ShardHashes is represented by a dictionary with 32-bit `workchain_ids` as keys, and “shard binary trees”, represented by TL-B type `BinTree` `ShardDescr`, as values. Each leaf of this shard binary tree contains a value of type `ShardDescr`, which describes a single shard by indicating the sequence number `seq_no`, the logical time `lt`, and the hash hash of the latest (signed) block of the corresponding shardchain.

```
bt_leaf$0 {X:Type} leaf:X = BinTree X;
bt_fork$1 {X:Type} left:^(BinTree X) right:^(BinTree X)
			= BinTree X;
fsm_none$0 = FutureSplitMerge;
fsm_split$10 mc_seqno:uint32 = FutureSplitMerge;
fsm_merge$11 mc_seqno:uint32 = FutureSplitMerge;

shard_descr$_ seq_no:uint32 lt:uint64 hash:uint256
	split_merge_at:FutureSplitMerge = ShardDescr;

_ (HashmapE 32 ^(BinTree ShardDescr)) = ShardHashes;
```

Fields `mc_seqno` of `fsm_split` and `fsm_merge` are used to signal future shard merge or split events. Shardchain blocks referring to masterchain blocks with sequence numbers up to, but not including, the one indicated in `mc_seqno` are generated in the usual way. Once the indicated sequence number is reached, a shard merge or split event must occur. Notice that the masterchain itself is omitted from ShardHashes (i.e., 32- bit index −1 is absent from this dictionary).

### Description of ShardFees

`ShardFees` is a masterchain structure used to reflect the total fees collected so far by the validators of a shardchain. The total fees reflected in this structure are accumulated in the masterchain by crediting them to a special account, whose address is a configurable parameter. Typically this account is the smart contract that computes and distributes the rewards to all validators.

```
bta_leaf$0 {X:Type} {Y:Type} leaf:X extra:Y = BinTreeAug X Y;
bta_fork$1 {X:Type} {Y:Type} left:^(BinTreeAug X Y)
			right:^(BinTreeAug X Y) extra:Y = BinTreeAug X Y;
_ (HashmapAugE 32 ^(BinTreeAug True CurrencyCollection)
	CurrencyCollection) = ShardFees;
```

The structure of ShardFees is similar to that of ShardHashes, but the dictionary and binary trees involved are augmented by currency values, equal to the `total_validator_fees` values of the final states of the corresponding shardchain blocks. The value aggregated at the root of ShardFees is added together with the `total_validator_fees` of the masterchain state, yielding the total TON Blockchain validator fees. The increase of the value aggregated at the root of ShardFees from the initial to the final state of a masterchain block is reflected in the `fees_imported` in the value flow of that masterchain block.  

## Description of ConfigParams.

Recall that the configurable parameters or the configuration dictionary is a dictionary `config` with 32-bit keys kept inside the first cell reference of the persistent data of the configuration smart contract γ. The address γ of the configuration smart contract and a copy of the configuration dictionary are duplicated in fields `config_addr` and `config` of a ConfigParams structure, explicitly included into masterchain state to facilitate access to the current values of the configurable parameters:

```
_ config_addr:uint256 config:^(Hashmap 32 ^Cell)
	= ConfigParams;
```

### Masterchain state data

The data specific to the masterchain state is collected into **McStateExtra**:

```
masterchain_state_extra#cc1f
	shard_hashes:ShardHashes
	shard_fees:ShardFees
	config:ConfigParams
= McStateExtra;
```

### Masterchain block data

Similarly, the data specific to the masterchain blocks is collected into **McBlockExtra**:

```
masterchain_block_extra#cc9f
	shard_hashes:ShardHashes
= McBlockExtra;
```

### Serialization of a bag of cells

Serialization of a bag of cells is a critical aspect of the TON Blockchain, as it enables efficient storage and transmission of data on the blockchain. The description provided in the previous section defines the way a shardchain block is represented as a tree of cells. However, this tree of cells needs to be serialized into a file, suitable for disk storage or network transfer. This section discusses the standard ways of serializing a tree, a DAG, or a bag of cells into an octet string.  

### Transforming a tree of cells into a bag of cells

Recall that values of arbitrary (dependent) algebraic data types are represented in the TON Blockchain by trees of cells. Such a tree of cells is transformed into a directed acyclic graph, or DAG, of cells, by identifying identical cells in the tree. After that, we might replace each of the references of each cell by the 32-byte representation hash of the cell referred to and obtain a bag of cells. By convention, the root of the original tree of cells is a marked element of the resulting bag of cells, so that anybody receiving this bag of cells and knowing the marked element can reconstruct the original DAG of cells, hence also the original tree of cells.  

### Complete bags of cells

 Let us say that a bag of cells is complete if it contains all cells referred to by any of its cells. In other words, a complete bag of cells does not have any “unresolved” hash references to cells outside that bag of cells. In most cases, we need to serialize only complete bags of cells.

### Internal references inside a bag of cells

Let us say that a reference of a cell c belonging to a bag of cells B is internal (with respect to B) if the cell ci referred to by this reference belongs to B as well. Otherwise, the reference is called external. A bag of cells is complete if and only if all references of its constituent cells are internal.

### Assigning indices to the cells from a bag of cells

Let c_0, . . . , c_n−1 be the n distinct cells belonging to a bag of cells B. We can list these cells in some order, and then assign indices from 0 to n − 1, so that cell c_i gets index i. Some options for ordering cells are:

• Order cells by their representation hash. Then Hash(ci) < Hash(c_j ) whenever i < j.

• Topological order: if cell ci refers to cell c_j, then i < j. In general, there is more than one topological order for the same bag of cells. There are two standard ways for constructing topological orders:

– Depth-first order: apply a depth-first search to the directed acyclic graph of cells starting from its root (i.e., marked cell), and list cells in the order they are visited.

– Breadth-first order: same as above, but applying a breadth-first search.

Notice that the topological order always assigns index 0 to the root cell of a bag of cells constructed from a tree of cells. In most cases, we opt to use a topological order, or the depth-first order if we want to be more specific. If cells are listed in a topological order, then the verification that there are no cyclic references in a bag of cells is immediate. On the other hand, ordering cells by their representation hash simplifies the verification that there are no duplicates in a serialized bag of cells.

### Outline of serialization process

The serialization process of a bag of cells B consisting of n cells can be outlined as follows:

1. List the cells from B in a topological order: c_0, c_1, . . . , c_n−1. Then c_0 is the root cell of B.

2. Choose an integer s, such that n ≤ 2 ^s . Represent each cell ci by an integral number of octets in the standard way, but using unsigned big-endian s-bit integer j instead of hash Hash(c_j ) to represent internal references to cell c_j.

3. Concatenate the representations of cells ci thus obtained in the increasing order of i.

4. Optionally, an index can be constructed that consists of n + 1 t-bit integer entries L_0, . . . , L_n, where L_i is the total length (in octets) of the representations of cells c_j with j ≤ i, and integer t ≥ 0 is chosen so that L_n ≤ 2 ^t .

5. The serialization of the bag of cells now consists of a magic number indicating the precise format of the serialization, followed by integers s ≥ 0, t ≥ 0, n ≤ 2 ^s , an optional index consisting of d(n+1)t/8e octets, and L_n octets with the cell representations.

6. An optional CRC32 may be appended to the serialization for integrity verification purposes.

If an index is included, any cell ci in the serialized bag of cells may be easily accessed by its index i without deserializing all other cells, or even without loading the entire serialized bag of cells in memory.  

### Serialization of one cell from a bag of cells.

More precisely, each individual cell c = c_i is serialized as follows, provided s is a multiple of eight (usually s = 8, 16, 24, or 32):

1. Two descriptor bytes d_1 and d_2 are computed by setting d_1 = r + 8s + 16h + 32l and d_2 = bb/8c + db/8c, where:

​	• 0 ≤ r ≤ 4 is the number of cell references present in cell c; if c is absent from the bag of 		cells being serialized and is represented by its hashes only, then r = 7.

​	• 0 ≤ b ≤ 1023 is the number of data bits in cell c.

​	• 0 ≤ l ≤ 3 is the level of cell c.

​	• s = 1 for exotic cells and s = 0 for ordinary cells.

​	• h = 1 if the cell’s hashes are explicitly included into the serialization; otherwise, h = 0

​	   (When r = 7, we must always have h = 1.)  

​	For absent cells (i.e., external references), only d__1 is present, always equal to 23 + 32l.

2. Two bytes d_1 and d_2 (if r < 7) or one byte d_1 (if r = 7) begin the serialization of cell c.

3. If h = 1, the serialization is continued by l + 1 32-byte higher hashes of c: Hash_1(c), . . . , Hash_l+1(c) = Hash∞(c).

4. After that, [b/8] data bytes are serialized, by splitting b data bits into 8-bit groups and interpreting each group as a big-endian integer in the range 0 . . . 255. If b 	is not divisible by 8, then the data bits are first augmented by one binary 1 and up to six 	                                                                 binary 0, so as to make the number of data bits divisible by eight.

5. Finally, r cell references to cells c_j1 , . . . , c_jr are encoded by means of r s-bit big-endian integers j_1, . . . , j_r.

### A classification of serialization schemes for bags of cells

A serialization scheme for a bag of cells must specify the following parameters:

• The 4-byte magic number prepended to the serialization.

• The number of bits s used to represent cell indices. Usually s is a

multiple of eight (e.g., 8, 16, 24, or 32).

• The number of bits t used to represent offsets of cell serializations . Usually t is also a multiple of eight.

• A flag indicating whether an index with offsets L_0, . . . , L_n of cell serializations is present. This flag may be combined with t by setting t = 0 when the index is absent.

• A flag indicating whether the CRC32-C of the whole serialization is appended to it for integrity verification purposes.

  

### Fields present in the serialization of a bag of cells

In addition to the values listed above, fixed by choosing a serialization scheme for cell packages, the following parameters must be specified when serializing a specific cell package :

• The total number of cells n present in the serialization.

• The number of “root cells” k ≤ n present in the serialization. The root cells themselves are c_0, . . . , c_k−1. All other cells present in the bag of cells are expected to be reachable by chains of references starting from the root cells.

• The number of “absent cells” l ≤ n − k, which represent cells that are actually absent from this bag of cells, but are referred to from it. The absent cells themselves are represented by c_n−l , . . . , c_n−1, and only these cells may (and also must) have r = 7. Complete bags of cells have l = 0.

• The total length in bytes L_n of the serialization of all cells. If the index is present, L_n might not be stored explicitly since it can be recovered as the last entry of the index.

  

## TL-B scheme for serializing bags of cells.

Several TL-B constructors can be used to serialize bags of cells into octet (i.e., 8-bit byte) sequences. The only one that is currently used to serialize new bags of cell is

  ```
  serialized_boc#b5ee9c72 has_idx:(## 1) has_crc32c:(## 1)
  	has_cache_bits:(## 1) flags:(## 2) { flags = 0 }
  	size:(## 3) { size <= 4 }
  	off_bytes:(## 8) { off_bytes <= 8 }
  	cells:(##(size * 8))
  	roots:(##(size * 8)) { roots >= 1 }
  	absent:(##(size * 8)) { roots + absent <= cells }
  	tot_cells_size:(##(off_bytes * 8))
  	root_list:(roots * ##(size * 8))
  	index:has_idx?(cells * ##(off_bytes * 8))
  	cell_data:(tot_cells_size * [ uint8 ])
  	crc32c:has_crc32c?uint32
  	= BagOfCells;
  
  ```

Field `cells` is n, `roots` is k, `absent` is l, and `tot_cells_size` is L_n (the total size of the serialization of all cells in bytes). If an index is present, parameters s/8 and t/8 are serialized separately as `size` and `off_bytes`, respectively, and the flag `has_idx` is set. The index itself is contained in index, present only if `has_idx` is set. The field `root_list` contains the (zero-based) indices of the root nodes of the bag of cells.

Two older constructors are still supported in the bag-of-cells deserialization functions:

```
serialized_boc_idx#68ff65f3 size:(## 8) { size <= 4 }
	off_bytes:(## 8) { off_bytes <= 8 }
	cells:(##(size * 8))
	roots:(##(size * 8)) { roots = 1 }
	absent:(##(size * 8)) { roots + absent <= cells }
	tot_cells_size:(##(off_bytes * 8))
	index:(cells * ##(off_bytes * 8))
	cell_data:(tot_cells_size * [ uint8 ])
	= BagOfCells;

```

```
serialized_boc_idx_crc32c#acc3a728 size:(## 8) { size <= 4 }
	off_bytes:(## 8) { off_bytes <= 8 }
	cells:(##(size * 8))
	roots:(##(size * 8)) { roots = 1 }
	absent:(##(size * 8)) { roots + absent <= cells }
	tot_cells_size:(##(off_bytes * 8))
	index:(cells * ##(off_bytes * 8))
	cell_data:(tot_cells_size * [ uint8 ])
	crc32c:uint32 = BagOfCells;
```

### Storing compiled TVM code in files. 

Notice that the above procedure for serializing bags of cells may be used to serialize compiled smart contracts and other TVM code. One must define a TL-B constructor similar to the following:

```
compiled_smart_contract
	compiled_at:uint32 code:^Cell data:^Cell
	description:(Maybe ^TinyString)
	_:^[ source_file:(Maybe ^TinyString)
		compiler_version:(Maybe ^TinyString) ]
	= CompiledSmartContract;
tiny_string#_ len:(#<= 126) str:(len * [ uint8 ]) = TinyString;
```

Then a compiled smart contract may be represented by a value of type `CompiledSmartContract`, transformed into a tree of cells and then into a bag of cells, and then serialized. The resulting octet string may be then written into a file with suffix .tvc (“TVM smart contract”), and this file may be used to distribute the compiled smart contract, download it into a wallet application for deploying into the TON Blockchain, and so on.  

### Merkle hashes for an octet string

On some occasions, we must define a Merkle hash HashM(s) of an arbitrary octet string s of length |s|. We do this as follows:

• If |s| ≤ 256 octets, then the Merkle hash of s is just its sha256:

$$
Hash_M(s) := sha256(s) 
if|s| ≤ 256.
$$


• If |s| > 256, let n = 2^k be the largest power of two less than |s| (i.e., k := [log2 (|s| − 1)], n := 2^k ). If s ^' is the prefix of s of length n, and s ^'' is the suffix of s of length |s| − n, so that s is the concatenation s^ '.s^'' of s ^' and s ^'', we define

$$
Hash_M(s) := sha256(INT_{64}(|s|).Hash_M(s^{'}).Hash_M(s^{''}) )
$$
In other words, we concatenate the 64-bit big-endian representation of |s| and the recursively computed Merkle hashes of `s^'` and `s ^''`, and compute sha256 of the resulting string. One can check that Hash_M(s) = Hash_M(t) for octet strings s and t of length less than `2^64` − `2^56` implies s = t unless a hash collision for sha256 has been found.

## The serialization hash of a block.
The construction above is applied in particular to the serialization of the bag of cells representing an unsigned shardchain or masterchain block. The validators sign not only the representation hash of the unsigned block, but also the “serialization hash” of the unsigned block, defined as Hash_M of the serialization of the unsigned block. In this way, the validators certify that this octet string is indeed a serialization of the corresponding block.

