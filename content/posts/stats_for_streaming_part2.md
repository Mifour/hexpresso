+++
title = "Stats for Streams 2 - Scalability in parallele"
date = "2022-06-05T22:39:47+02:00"
author = "Mifour"
authorTwitter = "" #do not include @
cover = ""
tags = ["rust", "maths"]
keywords = ["stats", "parallele", "Rust"]
description = "Efficient statistics computed in parallele using Rust"
showFullContent = false
readingTime = true
hideComments = false
+++

# Stats for Streams - part 2
*Efficient statistics computed in parallele using Rust*  
  
*TLDR; The complete script is available on my [github]()*  
  
## Previsouly on this site
I wrote the [first part](http://localhost:1313/posts/stats_for_streaming/) of this article explaining how I implemented statistics with a constant time routine. The two parts have different subjects but I highly suggest to read the first part before this one.  
  
A morning, drinking coffee, it striked me in the face: the implementation can be distributed between machines and scale horizontaly.
That's excellent news because if the event rate of the stream is too important or the amount of data is too large for one single computer, it is now possible to share the load among a cluster of machines.     
  
## Reimplementing in Rust
In this part, I will reimplement the Python code in Rust.
Don't get me wrong, I absoluty love Python. I do primarily code in Python at my job.  
But the difference between a fanatic and an engineer is to know what is the right tool for the right job.  
Now, the problem is not to transform math formulas anymore but to scale as much as possible. For that, I will use parallele computing (multi-threading, as rust threads do truly run in parallele, unlike Python). So, bye Python and hello Rust.  
  
Rust is a good tool for the job because it is a low-level compiled langage with great performances. Its compilation rules help a lot while writting concurent programs.  
    
Here's the same algorithm re implmeented in rust:

```rust
// rustc 1.61
struct Mean{
    nb: u64,
    total_sum: f64
}

impl Mean{
    fn update(&mut self, data: f64) -> f64{
        self.nb += 1;
        self.total_sum += data;
        (self.total_sum / (self.nb as f64)) as f64
    }
}


struct Variance {
        nb: i64,
        mean: f64,
        total_sum: f64,
        total_square: f64
}

impl Variance{
    fn update(&mut self, data: f64) -> f64{
        self.nb += 1;
        self.total_sum += data;
        self.total_square += data.powf(2.0);
        self.mean = (self.total_sum/(self.nb as f64)) as f64; 
        ((self.total_square as f64 - (2.0 * self.total_sum) as f64 * self.mean + self.nb as f64 * self.mean.powf(2.0)) / (self.nb) as f64) as f64
    }
}

fn main() {

    let mut mean = Mean{ nb:0, total_sum:0.0};
    let mut variance = Variance{ nb:0, mean:0.0, total_sum:0.0, total_square:0.0};

    for i in 0..100{
        println!("mean {:?}", mean.update(i as f64));
        println!("variance {:?}", variance.update(i as f64));
    }
    let std = variance.update(0.0);
    println!("standard variation {:?}", std.powf(0.5));
}

```
> As a side note, when writting this article, generators are not a stable feature of the Rust programming language. Some improvments may since been made, go check this [issue](https://github.com/rust-lang/rust/issues/43122).  
It's a bit disapointing but fortunately, a generator is essentially a state machine.
As long as the necessary data can be held staticaly, this can be implemented through a struc.  
Hopefully, the incremental formulas from part 1 permit just that.  
  
> Another downside of using Rust's native type (or a lof of languages' but Python or Js), is that overflows can happen.
Compiled in `--release`, rust code will not panic in case of overflow.  
You can read more at [Myths and Legends about Integer Overflow in Rust](https://huonw.github.io/blog/2016/04/myths-and-legends-about-integer-overflow-in-rust/)
  
## Map Reduce for distributed computing
In the case of a very large amount of data to process, it will be become impracticle to move the data into a single machine.
There's is only one solution: to distribute the load among a cluster of machines that only carry their own data localy.
Each machine will use the formula to iterate over all its own data, just like described in the previous code block.  
This is called mapping the update function to the data segment of a machine.
  
Despite being hundreds of machines, everyone of them can merge its data with another at the end.
Considering my usecase, this is true because merging two structs can be done by just adding similar attributes together, a simple `+`.  
This is called reducing the set of isolated states.  
  
Simulating this architecture on a single machine can done by using threads as separated machines:
```rust
//re-using the Mean and Variance structs

use std::fs::File;
use std::io::{BufRead, BufReader};
use std::str;
use std::thread;


fn compute_variance_from_file(i: &str) -> Variance{
    // read all float numbers from a txt file while updating a Variance
    // then return the variance
    let mut variance = Variance{nb:0, mean:0.0, total_sum:0.0, total_square:0.0};
    let file = File::open(i.to_owned() + ".txt").unwrap();
    let reader = BufReader::new(file);
    let mut lines = reader.lines();
    while let Some(line) = lines.next() {
        match line.unwrap().parse::<f64>(){
            Ok(data) => {
                variance.update(data);
            },
            Err(e) => {
                eprintln!("{:?}", e);
                break;
            }
        }
    }
    variance
}

fn main() {
	let mut whole_variance = Variance{nb:0, mean:0.0, total_sum:0.0, total_square:0.0};
	let mut handles = vec![];
    let mut results = Vec::<Variance>::new();
    for i in 0..4{
        handles.push(
            // map
            thread::spawn(
                move || -> Variance{
                    compute_variance_from_file(&i.to_string())
                }
            )
        );
    }
    for handle in handles{
    	results.push(handle.join().unwrap())
    }
    whole_variance.merge(
    	&results.into_iter().reduce(|mut a, b| { a.merge(&b); a}).unwrap() // reduce
    );
    println!("standard variation from map reduce is {:?}", whole_variance.compute().powf(0.5));
}
```

And the best part is that the `whole_variance` is the same updatable struct.
That means the worker can update the main node again and again with the differences between two synchronisations, the system scalability will remains as good.  

Linear scalability for every node, for the overwhole system and forever without any scope limitations.  
Scalability all the way!

  
Imagine you're a vast industrial company, you can get live statistics over all your sensors in the world, over the whole existence of the system.
That is awesome!  

## Benchmarks

I created 4 files of 10 000 000 floating point values with this command:
```bash
seq 0 .01 100000 | shuf | rg "," -r "." > 0.txt
```
The files are 85MB each and located in the ram thanks to /dev/shm with the command:  
```bash
mount -o remount,size=1G /dev/shm
cp 0.txt /dev/shm/0.txt 
```
So there is no bottle neck due to the disk.  
  
For comparaison, I made a single threaded version that is just iterating over the files one by one.

```rust
// single threaded!
fn main() {
	let mut whole_variance = Variance{nb:0, mean:0.0, total_sum:0.0, total_square:0.0};
	let mut results = Vec::<Variance>::new();
    for i in 0..4{
        results.push(
            // map
            thread::spawn(
                move || -> Variance{
                    compute_variance_from_file(&i.to_string())
                }
            ).join().unwrap()
        );
    }
    whole_variance.merge(
    	&results.into_iter().reduce(|mut a, b| { a.merge(&b); a}).unwrap() // reduce
    );
    println!("standard variation from map reduce is {:?}", whole_variance.compute().powf(0.5));
}
```
  
Of course, I use the `cargo build --release` option each time to tell Rustc to compile for performance.  
  
Here's the performance benchmark on my 2014 duo-core banged-up macbook air:
| program | time |
| ----------- | ----------- |
| multithreaded x4 | 2.2s |
| single thread | 4.3s |
  
My friend got a much more satisafying 1.3s with 16 threads on his 8 cores dual thread cpu equiped laptop.
  
Yes, there is a small overhead to launch threads compare to just a single thread loop.
_________________
## Thanks

That was it.  
Thank you for reading this article.  
I hope you like this article and this adventure for scalability.  
After though, I understood it was basically rediscovering map reduce and stats 101 at the same time.  
  
Have a nice day,  
Mifour :) 
