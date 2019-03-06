#  More Efficient Unitrie Key Mapping  

| RSKIP          | nn                                 |
| :------------- | :--------------------------------- |
| **Title**      | More Efficient Unitrie Key Mapping |
| **Created**    | 2019                               |
| **Author**     | SDL & AL                           |
|                | Sca, Usa                           |
| **Layer**      |                                    |
| **Complexity** | 2                                  |
| **Status**     | Draft                              |

# Abstract

The Unitrie RSKIP16 specifies that the account/contract trie is combined with the per contract storage tries. However that RSKIP uses the same key mapping as Ethereum: an account address is hashed with Keccak to obtain a 256-bit key, and the same for storage cell addresses.

This mapping is inefficient in terms of space and also looses the pre-image information, which nodes must store elsewhere. We propose a new key mapping that not only preserves the original keys, but also consumes less space than in the previous system. We also change storage gas costs to incentivize using shorter keys and values.

# Motivation

Blockchain state is a scarce resource that must be protected. The shortest the state, the more decentralize a cryptocurrency can be. The "secure" Patricia trie, is a data structure that is invariant to the order elements are inserted. But to keep the data structure probabilistically balanced, it hashes all keys. However, the blockchain doesn't need to keep keys balanced with cryptographical assurance, but only with economical assurance. Therefore a 80-bit hash digest prefix is more than enough to make the risk of imbalance attacks negligible. However a 80-bit key is not enough to protect the account data from pre-image attacks that try to obtain ownership of an account. Therefore we can build keys using a 80-bit hash prefix to keep the trie balanced and concatenate the key with the original key, to make it unique. We obtain account trie keys consuming only 30 bytes (instead of 32 bytes, as before, and therefore reducing the space requirements)

When the same strategy is applied to storage cell keys, the size of the key results in an expansion from 32 bytes to 42 bytes, because storage cell addresses can use up to 256-bits. However we use the fact that many storage addresses are very low when considered as big-integers. For example, Solidity uses sequential addresses, starting at 0x00, for the contract fields. Therefore we can compress keys by removing all leading zeros (the same that RSK does with cell values). This yields keys that vary between 11 bytes (80-bit hash prefix plus one byte) to 42 bytes (80-bit hash prefix plus 32 bytes). We also change the cost of storage cells so that the cost to create new cells using shorter keys is less than using longer keys. 

This scheme incentivizes the use of shorter keys and shorter values. It's important to note that Solidity hashes the mapping keys with the field index using the SHA3 opcode to obtain cell addresses, even when the mapping index size is shorter than 256 bits. For instance, in the following code:

```
pragma solidity ^0.5.0;
contract hash {
        public uint useSlot0;
        mapping (address => bool) public isOwner;    
        function add() public {
            isOwner[msg.sender] = true;
        }
}
```

The address for SSTORE in add() is obtained by padding msg.sender to 256-bits, appending a 256-bit number equal to 1, and then hashing with SHA3 to obtain a 256-bit result.

Clearly Solidity could have concatenated the field index (0x01) in the 21st byte (1st byte being the LSB) with the address, which is a 20-byte field in the least significant part. This would yield a 21-byte key, instead of a 32-byte key,  and therefore each key/value would consume 11 bytes less.  Therefore by reducing the cost of using shorter keys we're incentivizing the compiler to implement more space-efficient constructions.



## Specification

 The new key mapping is as follows:

| Key                                                          | Value                   | Description                                |
| ------------------------------------------------------------ | ----------------------- | ------------------------------------------ |
| 0x00<br />SHA3(account_addr)[0:9]<br />account_addr          | rlp(nonce,amount,flags) | account state                              |
| 0x00<br />SHA3(account_addr)[0:9]<br />account_addr<br />0x80 | byte array              | account code                               |
| 0x00<br />SHA3(account_addr)[0:9]<br />account_addr<br />0x00 | 0x00                    | placeholder for isolating the storage tree |
| 0x00<br />SHA3(account_addr)[0:9]<br />account_addr<br />0x00<br />SHA3(trimmed_storage_address)[0..9]   trimmed_storage_address | byte array              | value at a storage cell                    |

trimmed_storage_address is the address without any leading zeros, except for the address 0x00 which is stored as a single byte 0x00. 

[a:b] represents the most significant bytes starting with byte at offset "a" until byte at offset "b" (included).

The new gas cost for creating a new storage cell is changed from 20k to:

​	newCellCost = 5000 + size_cost
​	size_cost = 230*(keyBytes+dataBytes)

This is the SET cost. As 0x00 is trimmed to 0x00, the minimum value for keyBytes is 1, and the minimum value for dataBytes is 1.

The maximum SET cost is: 19720

The minimum SET cost is: 5460

When a cell value is changed, the cost of the keyBytes stay constant, but the cost of dataBytes can change. Therefore both the REFUND value and the RESET cost must be modified.

Any change attempt that does not modify the value has a cost of 200 (similar to SLOAD).  For instance zero to zero, or A to A.

The REFUND value is computed as SET cost minus 5000. (Maximum 14720, minimum 460).

The CLEAR cost is still computed as 5000.

The RESET cost now only accounts for value changes from a non-zero value to a different non-zero value. This has a base cost of 5000 plus the difference between new cell costs and old cell cost (maximum 12130, minimum 5000).



# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



```

```