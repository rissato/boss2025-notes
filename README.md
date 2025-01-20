# My notes for the BOSS 2025

## Basics

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
* `String` is a growable, heap-allocated string unkown at compile time, like user inputs. The pointer to the heap is stored on the stack.

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


### base58

base58 encoding means each character can be 58 different values. These are the possible characters:

```123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz```

When decoding, the checksum can be used to verify the data. IMPORTANT: Only use the first 4 characters of the sha25 digest result.

### Bitcoin Protocol

BIP32 is very important, talks about modern key derivation paths:

https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki

The math behind BIP32:

https://medium.com/@robbiehanson15/the-math-behind-bip-32-child-key-derivation-7d85f61a6681

## Sources

Bitcoin Book

https://github.com/bitcoinbook/bitcoinbook

Learn me a bitcoin

https://learnmeabitcoin.com/

