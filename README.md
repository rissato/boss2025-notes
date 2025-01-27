# My notes for the BOSS 2025

## Basics

### base58

base58 encoding means each character can be 58 different values. These are the possible characters:

```123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz```

When decoding, the checksum can be used to verify the data. IMPORTANT: Only use the first 4 characters of the sha25 digest result.

### Bitcoin Protocol

#### Transaction Types (Input and Outputs)

The type of the input transaction/nout combination (also called an outpoint) will determine the script code field (which is part of the data hashed to create the signature) and the witness field. The output type of the transaction will determine the transaction outputs scriptPubKey format, which can be of different types in the same transaction.

* Pay to Witness Public Key Hash (p2wpkh): The input transaction is commiting to a public key hash, so the transaction spending this coins should simply prove it has control over the private key from which the public key and hash are derived.
  * The script code should be: ```OP_DUP OP_HASH160 <pubkey hash> OP_EQUALVERIFY OP_CHECKSIG```.
  * The witness field should be: ```<signature> <pubkey>```.
  * When creating and output of this type, the scriptPubKey should be: ```OP_0 <pubkey hash>```.
* Pay to Witness Script Has (p2wsh): A transaction can only spend this coins if it can run a script that hashes to the same script hash as the original output. The script can be of a lot of different types. For example, a M-of-N multisig script means the spender needs to prove onwership of at least M out of a total of N keys.

Depending on the types above, previous transactions data and the new transaction data are combined together in different ways and validated by the bitcoin protocol via script execution (section below).

#### Script Processing

When a transaction is processed, bitcoin nodes use a combination of data from the original transaction and the one being currently validated, in a very particular way. Some definitions:

* *Witness Program* and *Witness Scripts* are different things. The witness program comes from the input transaction output being spent, specifically the scriptPubKey value. The witness script is the last item (when present) of the witness section of the new transaction (spending the UTXO).

* *Witness Stack* is the stack of data that is used to execute the script. It includes the witness script (last element) and the data used in the script, like keys and signatures. 

The example below is taken from the bitcoin book (chapter 4), but described with much more slowly going step by step. Start by combining the script with the input data:

```
<sig> <pubkey>  OP_DUP OP_HASH160 <commitment> OP_EQUALVERIFY OP_CHECKSIG
--------------  ---------------------------------------------------------
     INPUT                               SCRIPT        
```
Now items are pushed to the stack in order. Because of the nature of a stack, the last item pushed is the top elemenent:

``` 
<sig> <pubkey> OP_DUP OP_HASH160 <commitment> OP_EQUALVERIFY OP_CHECKSIG
               ˆ NEXT ITEM
STACK:
<pubkey>
<sig>
```
All OP_* opeartions consume one of more items from the top os the stack. When the OP_DUP is executed, the top element is duplicated:
``` 
<sig> <pubkey> OP_DUP OP_HASH160 <commitment> OP_EQUALVERIFY OP_CHECKSIG
                      ˆ NEXT ITEM
STACK:
<pubkey>
<pubkey>
<sig>
```
The OP_HASH160 is executed, and the top element is hashed. 
``` 
<sig> <pubkey> OP_DUP OP_HASH160 <commitment> OP_EQUALVERIFY OP_CHECKSIG
                                 ^ NEXT ITEM
STACK:
<pubkey_hash>
<pubkey>
<sig>
```
The commitment is pushed to the stack:
``` 
<sig> <pubkey> OP_DUP OP_HASH160 <commitment> OP_EQUALVERIFY OP_CHECKSIG
                                              ^ NEXT ITEM
STACK:
<commitment>
<pubkey_hash>
<pubkey>
<sig>
```
The OP_EQUALVERIFY is executed, and the top two elements are compared. If they are not equal, the script fails. In this case, the commitment is the hash of the public key, so the match.
``` 
<sig> <pubkey> OP_DUP OP_HASH160 <commitment> OP_EQUALVERIFY OP_CHECKSIG
                                                             ^ NEXT ITEM
STACK:
<pubkey>
<sig>
```
The OP_CHECKSIG is execute, last two items are consumed, and the signature is verified. This concludes the execution. Depending on the transaction type, data elements and operations will be different, but the process is the same.


### Rust Trivia

#### Stack vs Heap
* Stack: Faster, more structured memory. Collections variables must be of a fixed size.
* Heap: Slower, more flexible memory. Collections variables can be of any size. Pointers to heap data are stored on the stack.

#### Ownership/Borrowing:
* Ownership rules:
  - Each value in Rust has an owner.
  - There can only be one owner at a time.
  - When the owner goes out of scope, the value will be dropped.
* Passing a variable to a function gives up ownership, unless passed by reference or copied. 
* Using ``&var`` gives a reference to the value, without giving up ownership. This is called borrowing.
* Not allowed to modify a borrowed value, unless it's explicitly marked as ``&mut var``. Only one mutable reference can exist at a time, and only if there are no other immutable references to the value. But the above will still complie if the immutable references were already used.
* Reference rules:
  - At any given time, you can have either one mutable reference or any number of immutable references.
  - References must always be valid.


#### Different types of collections
* Vectors
  - Can use [..] for indexing.  
  - Fixed length at compile time.
* Slices
  - Slices are a kind of reference(like in Ownership/Borrowing).
  - Can use [..] for indexing. 
  - Represented as `[u8]`
  - Variable size. 
  - Use `&vec` to get a slice from a vector.
  - Works on Strings.

#### Derive 
Derive is used to import traits to implement common behavior.
* Debug - Allows to use `{:?}` in println!.
* Serialize/Deserialize - Allows to use `serde_json::to_string(&my_struct)`.
* PartialEq/Eq - Allows to use `==` in custom structs.
* Copy - perform a deep copy on custom object. 

#### String vs `&str`:
* `&str` are immutable text literals. These are all imutable references to the literal stored in the binary.
* `String` is a growable, heap-allocated string unkown at compile time, like user inputs. The pointer to the heap is stored on the stack.  Use `format!` to create String objects.

Use :: to call a function on a type. Use . to call a function on an instance.

#### Types conversion table
| From/To | Slice | Vector | Hex String | Array 
| --- | --- | --- | --- | --- 
| Slice | - | extend_from_slice | hex::encode | .try_into()
| Vector | [..] (Slicing) | - | - | .try_into()
| Hex String | - | hex::decode | - | -
| Array | &array[..] | - | - | -

### Useful Code Snippets

#### Writing a file

```rust
  // Save incoming and outgoing transactions to files
let mut outgoing_txs_file = File::create("data/outgoing_txs.txt").unwrap();
for tx in &outgoing_txs {
    outgoing_txs_file.write_all(tx).unwrap();
    outgoing_txs_file.write_all(b"\n").unwrap();
}
```

#### Reading a file
```rust 
let mut outgoing_txs_file = File::open("outgoing_txs.txt").unwrap();
match File::open("data/outgoing_txs.txt") {
    Ok(mut file) => {
        let mut content = String::new();
        file.read_to_string(&mut content).unwrap();
        let lines: Vec<&str> = content.lines().collect();
        for line in lines {
            let tx: Vec<u8> = hex::decode(line).unwrap();
            outgoing_txs.push(tx);
        }
    },
    Err(_) => {
        // Handle the error
        println!("Error opening file: outgoing_txs.txt");
    }
}
```

#### Create a json object with serde
```rust
let mut descriptors = vec![];
let mut test_desc = serde_json::json!({
    "desc": format!("wpkh({}/{}/*)", extended_private_key, derivation_path),
    "range": 2000 
});
```

## Sources

Bitcoin Book

https://github.com/bitcoinbook/bitcoinbook

Learn me a bitcoin

https://learnmeabitcoin.com/

BIPs

https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki

The math behind BIP32:

https://medium.com/@robbiehanson15/the-math-behind-bip-32-child-key-derivation-7d85f61a6681


Useful blogs:

https://medium.com/coinmonks/creating-and-signing-a-segwit-transaction-from-scratch-ec98577b526a