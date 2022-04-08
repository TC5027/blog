# Optimizing deserialization with an alternative to Vec

We consider data of dynamic size, for example a vector of bytes.
We set the following alias:

```rust
type Bytes = Vec<u8>;
```

We want to write these data on disk and then load them back to memory

To write them on disk (serialize) we can use this approach:

* a header containing the full size of the sequence and the number of vectors represented
* sizes of the vectors, written successively
* data itself, all concatenated

The serialization can be done that way:

```rust
fn serialize(set_of_bytes: Vec<Bytes>) -> Vec<u8> {
    let nb_sets = set_of_bytes.len();
    let total_length = std::mem::size_of::<usize>()
        + std::mem::size_of::<usize>() * nb_sets
        + set_of_bytes.iter().map(|bytes| bytes.len()).sum::<usize>();
    // size to write nb_sets
    // + size to write length fields
    // + size to write all content

    let mut result: Vec<u8> = Vec::with_capacity(std::mem::size_of::<usize>() + total_length);
    result.append(&mut Vec::from(total_length.to_ne_bytes()));
    result.append(&mut Vec::from(nb_sets.to_ne_bytes()));

    for bytes in &set_of_bytes {
        result.append(&mut Vec::from(bytes.len().to_ne_bytes()));
    }
    for mut bytes in set_of_bytes {
        result.append(&mut bytes);
    }

    result
}
```

And the deserialization like so:

```rust
fn deserialize1<R>(mut storage: R) -> Vec<Bytes>
where
    R: Read,
{
    // we get the total_length to know until where we should read
    let mut a_usize = [0; std::mem::size_of::<usize>()];
    storage.read_exact(&mut a_usize).unwrap();
    let total_length = usize::from_ne_bytes(a_usize);
    // we get all the data
    let mut full_data = vec![0; total_length];
    storage.read_exact(&mut full_data).unwrap();
    // we get the number of sets
    let nb_sets = usize::from_ne_bytes(full_data[0..8].try_into().unwrap());
    // we split full_data between the sizes and the contents
    let usize_size = std::mem::size_of::<usize>();
    let mut contents = full_data.split_off(usize_size + usize_size * nb_sets);
    // we reform the sizes from bytes
    let mut sizes: Vec<usize> = Vec::with_capacity(nb_sets);
    for i in 0..nb_sets {
        sizes.push(usize::from_ne_bytes(
            full_data[usize_size * (i + 1)..usize_size * (i + 2)]
                .try_into()
                .unwrap(),
        ));
    }
    let mut result = Vec::with_capacity(nb_sets);
    for size in sizes {
        let remaining_contents = contents.split_off(size);
        result.push(contents);
        contents = remaining_contents;
    }

    result
}
```

We can verify our functions with the following ```main```:

```rust
fn main() {
    let bytes1: Bytes = vec![1,2,3];
    let bytes2: Bytes = vec![4,5];
    let serialized = serialize(vec![bytes1, bytes2]);
    let mut storage: File = OpenOptions::new().append(true).read(true).create(true).open("storage.strg").unwrap();
    storage.write_all(&serialized).unwrap();
    storage.seek(SeekFrom::Start(0)).unwrap();
    assert_eq!(vec![vec![1,2,3],vec![4,5]], deserialize1(storage));
}
```

Now, if we consider a scarier ```main``` like this one:

```rust
fn main() {
    let set_of_bytes = (0..10_000).into_iter().map(|_| vec![0; 50]).collect();
    let serialized = serialize(set_of_bytes);
    let mut storage: File = OpenOptions::new()
        .append(true)
        .read(true)
        .create(true)
        .open("storage.strg")
        .unwrap();
    storage.write_all(&serialized).unwrap();
    storage.seek(SeekFrom::Start(0)).unwrap();
    assert_eq!(10_000, deserialize1(storage).len());
}
```

To investigate it and have a better understanding of what is happening, I can use a flamegraph which visualizes stack traces. Before running it, I do:
```cmd
echo -1 | sudo tee /proc/sys/kernel/perf_event_paranoid
echo 0 | sudo tee /proc/sys/kernel/kptr_restrict
```

and then
```cmd 
cargo flamegraph
```
which produces a pretty .svg :D .

![ouch](https://github.com/TC5027/blog/blob/master/pngs/bad.svg)

Scary... For the analysis, we see that our ```main``` spends a lot of times in ```deserialize1```, more precisely in ```split_off``` from ```Vec```. Reading its [source code](https://doc.rust-lang.org/src/alloc/vec/mod.rs.html#1928-1930) we see that it leads to an allocation and a new vector ( see [1](https://doc.rust-lang.org/beta/src/alloc/raw_vec.rs.html#132) and [2](https://doc.rust-lang.org/beta/src/alloc/raw_vec.rs.html#170) ). If we think of it, it is a bit dumb in our case to perform a new allocation as we have already loaded everything from disk ! Because of this behaviour, we suffer from a uselessly huge memory consumption which surely is responsible for the page faults we highlight in the flamegraph.
There is a lot happening as we see in the flamegraph, but there is surely TOO MUCH happening given our task ? ```Vec``` seems to be a bad idea for this deserialization.

We propose ```SliceableBytes```.

```SliceableBytes``` is made of a field pointing to the start of the data, a size and a memory area wrapped inside a ```Rc```.

```rust
#[derive(Debug)]
struct SliceableBytes {
    start: *const u8,
    size: usize,
    shared_playground: Rc<Vec<u8>>,
}
```
This ```SliceableBytes``` has at creation a vector of u8 (a memory area) assigned to it. In our case this vector is the one loaded from disk. Then, the ```SliceableBytes``` that we will obtain through calls to ```split``` will share with the first one this vector.
Doing so we have a memory area that we can slice as we want, inherited from the ```Global``` allocator, which will be correctly deallocated when the ```Rc``` will fall to 0 meaning when there is no more use, no more ```SliceableBytes``` linked to it.

To build a ```SliceableBytes```, given our usage, we take as input a vector of u8 that we want to slice. We get its size and we take ownership of it, wrapping it inside a ```Rc```. Like that the memory area won't be deallocated as long as we have ```SliceableBytes``` instances using it.
```rust
impl SliceableBytes {
    pub fn new(to_be_sliced: Vec<u8>) -> Self {
        let size = to_be_sliced.len();    
        let start = to_be_sliced.as_ptr();
        Self {
            start,
            size,
            shared_playground: Rc::from(to_be_sliced),
        }
    }
}
```
The second method, the key one, is ```split``` which returns a new instance of ```SliceableBytes```.

We perform a test on the feasibility of the cut given the size. We update it and we return a new ```SliceableBytes``` corresponding to the right side of the cut.
```rust
pub fn split(&mut self, index: usize) -> SliceableBytes {
    assert!(index <= self.size);
    let size = self.size - index;
    self.size = index;
    SliceableBytes {
        start: unsafe { self.start.add(index) },
        size,
        shared_playground: self.shared_playground.clone(),
    }
}
```

We can now update our code for deserialize, using ```SliceableBytes```

```rust
fn deserialize2<R>(mut storage: R) -> Vec<SliceableBytes>
where
    R: Read,
{
    // we get the total_length to know until where we should read
    let mut a_usize = [0; std::mem::size_of::<usize>()];
    storage.read_exact(&mut a_usize).unwrap();
    let total_length = usize::from_ne_bytes(a_usize);
    // we get all the data
    let mut full_data = vec![0; total_length];
    storage.read_exact(&mut full_data).unwrap();
    // we get the number of sets
    let nb_sets = usize::from_ne_bytes(full_data[0..8].try_into().unwrap());
    // we split full_data between the sizes and the contents
    let usize_size = std::mem::size_of::<usize>();
    let contents = full_data.split_off(usize_size + usize_size * nb_sets);
    // we are going to slice contents
    let mut sliceable_contents = SliceableBytes::new(contents);
    // we reform the sizes from bytes
    let mut sizes: Vec<usize> = Vec::with_capacity(nb_sets);
    for i in 0..nb_sets {
        sizes.push(usize::from_ne_bytes(
            full_data[usize_size * (i + 1)..usize_size * (i + 2)]
                .try_into()
                .unwrap(),
        ));
    }
    let mut result = Vec::with_capacity(nb_sets);
    // ONLY CHANGE
    for size in sizes {
        let remaining_contents = sliceable_contents.split(size);
        result.push(sliceable_contents);
        sliceable_contents = remaining_contents;
    }

    result
}
```

Replaying the same ```main``` than before, we see as a proof of our success that the flamegraph is now much simpler, we went from 700 samples to 7.

![better](https://github.com/TC5027/blog/blob/master/pngs/good.svg)

note: [bytes](https://crates.io/crates/bytes)
