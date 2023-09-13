---
layout: post
title: Overthinking a Skinny XML Parser
date: 2023-07-20 10:14:00-0400
description: building a parser for MATSim network format
tags: MATSim Rust parsing xml
categories:
giscus_comments: false
related_posts: false
toc:
  sidebar: left
---
Gonna build a nice fast xml parser for the MATSim network format. Using Rust, roughly as follows:

- Problem statement
- xml format
- Broad strategy
- Initial implementation
- Benchmark
- Fiddling

I've built similar stuff before but I'm pretty keen to meddle with the implementation in a systematic way. I'm going to hunt down something fast and skinny.

# Problem statement

I have a big project in mind. For which parsing and holding some big data structures in memory is only a tiny part. But a nice enough place to start. 

I am going to read a MATSim network file (an xml, typically gzip) and store it (but just the bits i need) in memory. Road networks are most usefully represented as (directed) graphs, where junctions are vertices/nodes and the links between them edges. But for starters, I'm just going to focus on link lengths.

I want to get these lengths into memory fast, but in the grandiose scheme of things, it's going to be more useful to have (i) a really fast to access data structure, and (ii) minimal memory impact. Fast creation of this data structure is actually just a nice to have.

---
**NOTE**

*Fast data structure?* - I mean that I want to be able to get the length of a link as quickly as possible using the link name.

---

# MATSim XML Format

The input data format is a MATSim network file:

```xml
<network>

	<nodes>
		<node id="nodeA" x="0.0" y="0.0" ></node>
		<node id="nodeB" x="100.0" y="0.0" ></node>
	  <!-- and so on... -->
	</nodes>

	<links>
		<link id="linkE" from="nodeA" to="nodeB" length="100.0" modes="bus,car" ></link>
		<link id="linkW" from="nodeB" to="nodeA" length="102.0" modes="bus,car" ></link>
	  <!-- and so on... -->

</network>

```

Two things to note at this point; (i) nodes (vertices if you're still graph thinking) are defined first, then (ii) links are defined second and contain the desired length attribute.

To get us started we're going to parse into the following structure:

```rust
type LinkLengths = HashMap<String, f32>;
```

Each link will be identified as by it's id value (as a String) and the length as an f32 (the Rust primitive 32-bit floating point type).

We are going to keep this in a `Links` struct so that we can tack on methods and extend in future:

```rust
pub struct Links {
    lengths: LinkLengths
}
```

---
**NOTE**

f32 is dumb. Firstly we probably don't want negative numbers, they do not make sense for a road network. Secondly f32 is unnecessarily large. So we might revisit this later.

---

# Broad Strategy

Firstly I'm just going to get it running. The excellent [quick_xml](https://github.com/tafia/quick-xml) is going to spin through the xml and we're going to populate our `LinkLengths`.

Then we're going to test and establish some benchmarks for:

- parsing the xml (fast is good but this isn't really a top priority) 
- read time (we're going to ask for the length of a bunch of links)

Then we're going to try out a few obvious improvements:

- relentlessly stress about using String and or clone
- squeeze the data types
- completely reimagine how we are going to be able to use the data (ie reindex and use an array)

# Initial Implementation

## Reader

First I set up a helper funcion to set up the [quick_xml reader](https://docs.rs/quick-xml/latest/quick_xml/index.html). I engage in some `Box` shenanigans because the full version also accepts a decoding BufReader for zipped files.

```rs
pub type XMLReader = quick_xml::Reader<Box<dyn BufRead>>;

pub fn from_path(path: impl AsRef<Path>) -> Result<XMLReader, XMLReaderError> {
    let file = File::open(&path).context("Could not open file")?;
    let reader = Box::new(BufReader::new(file));
    let xml_reader = quick_xml::Reader::from_reader(reader)
    xml_reader.check_comments(false).check_end_names(false);
    Ok(xml_reader)
}
```

---
**NOTE**

The quick_xml docs say we can disable checking in various places and go faster. This is only sensible if we trust the ingested xml, which I choose to do for now.

---
**NOTE**

Also note that I've been careful/pedantic/naive and created some custom errors using `thiserror`. I'm not including any explanation of these. If you're interested in exact the final implementation then LINK INCOMING.

---

## Parsing XML

The `XMLReader` we have created is just a fancy quick_xml `Reader` struct. This is used to loop through the xml contents returning them in order as different quick_xml::events::Event enum variations. We can then run a match statement on these events and do something with them depending on thier contents. The general pattern is:

```rs
let mut reader = from_path(&path)
let mut buffer = Vec::new();
loop {
  match reader.read_event_into(& mut buffer)? {
    Event::Start(event) => do_something(event),
    Event::Eof => break,
    _ => {},
  }
}
```

There are various different Events that make up what I generally think of as xml elements. We are going to ignore everything apart from `Event::Start` which will hold the information we need for our link lengths map (eg `<link id="linkE" from="nodeA" to="nodeB" length="100.0" modes="bus,car" >`), and `Event::Eof` which we use to identify the end of the file so we can break the loop. 

---
**NOTE**

quick_xml is reading our data into the `buffer` and generally doing as best as it can to avoid allocating memory. The oportunity for us is to then do the absolute minimum with this buffer data, only using what we need.

---

`Event::Start` holds a `StartEvent` struct which has a `name` field. This is essentially just a view into the buffer, but we can use it to look for elements named "link" and then do something with it:

```rs
loop {
  match reader.read_event_into(& mut buffer)? {
    Event::Start(event) if event.name().as_ref() == b"link" => {
      do_something(event)
    },
    Event::Eof => break,
    _ => {},
  }
}
```

## Parsing XML Attributes

So now `do_something` is only going to be called on link elements. These elements contain the xml attributes, why quick_xml exposes as `Attributes` via `event.attributes()`:

```rs
loop {
  match reader.read_event_into(& mut buffer)? {
    Event::Start(event) if event.name().as_ref() == b"link" => {
      let attributes = event.attributes();
      do_something(attributes);
    },
    Event::Eof => break,
    _ => {},
  }
}
```

The `Attributes` can be iterated though as individual `Attribute` structs. These contain a key and value field. We can now look at these keys, to identify the ones we need (`id` and `length`).

---
**NOTE**

Reminder that `Attributes` and `Attribute` still contain fields that are slices of our buffer. quick_xml is "simply" making sure we look at the right bits.

---

I'm confident that `id` is always going to be the first attribute, so we can create a String called id as follows:

```rs
loop {
  match reader.read_event_into(& mut buffer)? {
    Event::Start(event) if event.name().as_ref() == b"link" => {
      let attributes = event.attributes();
      // some wizardry to create an iterator
      let mut iter_attributes = attributes.with_checks(false).flatten();

      // get id (we "know" it will be first)
      let id_attribute = iter_attributes.next().ok_or(LinkParserError::LinkIdReadFailure())?.value;
      let id = String::from_utf8(id_att.to_owned())?;

      do_something(attributes);
    },
    Event::Eof => break,
    _ => {},
  }
}
```

 `String::from_utf8` is creating a String struct from the buffer. Up until now we avoided any new allocations beyond the buffer.

 I'm less confident on the consistency of the location of the `length` attribute. If I was I would use something like `iter_attributes.take(3).next()`.
 Instead we are going to use `find_map`. This will take a function that should return `Some(f32)` in the event of a `length` attribute and otherwise `None`:

```rs
fn parse_length(a: Attribute) -> Option<f32> {
    match a.key.into_inner() {
        b"length" => {
            let string = str::from_utf8(&a.value).unwrap();
            let float = string.parse::<f32>().unwrap();
            Ok(float)
        },
        _ => None
    } 
}
```

We use this in our loop to get the `length` value:

```rs
loop {
  match reader.read_event_into(& mut buffer)? {
    Event::Start(event) if event.name().as_ref() == b"link" => {
      let attributes = event.attributes();
      // some wizardry to create an iterator
      let mut iter_attributes = attributes.with_checks(false).flatten();

      // get id (we "know" it will be first)
      let id_attribute = iter_attributes.next().ok_or(LinkParserError::LinkIdReadFailure())?.value;
      let id = String::from_utf8(id_att)?;

      // get length (this could be anywhere later)
      let length = attributes.find_map( Self::parse_length )
      .ok_or(LinkParserError::FindLengthError())??;
    },
    Event::Eof => break,
    _ => {},
  }
}
```

---
**NOTE**

On handling errors: `parse_length()` uses `unwrap()` twice, panicking if either it can't convert to `str` or parse that as a float.

The fancier version propagates these errors up by returning a LinkParser error. This results in an awkward looking return type; `Option<Result<f32, LinkParserError>>`  - ie an `Option` of a `Result`.

In this case `None` means that the given `Attribute` was not a link length. No problem - we keep iterating.

`Some<LinkParserError>` means that it found the link length attribute key but failed to parse the value into a float.

`Some<Ok<f32>>` is what we are after but we then have to unpack all this.

There is then the case where we only returned `None` - ie we failed to find the length attribute - in which case we `find_map` returns `None` and we raise a new error `FindLengthError`.

This was certainly all overkill.

---

## Finally

So now we have a way of getting `id` and `length` for each link we can put this logic in a constructor method for our `Links` struct. I simply add an empty hashmap (the `LinkLengths` type), populate this with `id` and `length`. Then return a populated `Links` struct:

```rs
impl Links {

    pub fn from_reader(mut reader: XMLReader) -> Result<Self, LinkParserError> {
        let mut lengths = LinkLengths::new();
        let mut buf = Vec::new();
        loop {
            // omited logic used to get id and length
            // add to hashmap
            lengths.insert(id, length);
        }
        Ok(Links {
            lengths
        })
    }
}
```

Then some tests:

```rs
#[cfg(test)]
mod tests {
    use std::path::PathBuf;
    use super::*;

    #[test]
    fn parse_test_network_xml() {
        let mut path = PathBuf::from(env!("CARGO_MANIFEST_DIR"));
        path.push("fixtures/network_test.xml");
        let links = Links::from_xml(path, false).unwrap();
        assert_eq!(links.lengths.len(), 2);
        assert_eq!(links.lengths.get("linkE"), Some(&100.5));
        assert_eq!(links.lengths.get("linkW"), Some(&102.0));
    }
}
```

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/work.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    <a href='https://xkcd.com/1741'>xkcd</a>
</div>

# Benchmarking

Reminder that I want to build two types of benchmark:

- how fast can we parse the xml
- how fast can we access link lengths from our `Links` struct

I'm using `criterion` for benchmarking. I am not going to go into details (criterion has a good getting started) just focus on broad sweeps.

## Parsing XML

Easy peasy, I bosh a big example xml in memory. I am slightly concerned that using too small a dataset will make measuring improvemnts hard because there will be some fixed io cost that will dominate tiny examples. So I end up using an 8MB MATSim network from central London.

```rs
pub fn parse_link_lengths(c: &mut Criterion) {
    c.bench_function("build link lengths from xml", |b| {
        b.iter(|| {
            let mut path = PathBuf::from(env!("CARGO_MANIFEST_DIR"));
            path.push("fixtures/network_big.xml");
            let links = Links::from_xml(black_box(path)).unwrap();
        })
    });
}
```

We can load this network in ~30ms.

## Accessing

This is basically just calling `get(&key)` on `LinkLengths` which is a hashmap. I use the same network for this test but keep it's creation outside the benchmarker. I stress a little about having a systematic and consistent way of creating link keys:

```rs
pub fn access_link_lengths(c: &mut Criterion) {
    let mut path = PathBuf::from(env!("CARGO_MANIFEST_DIR"));
    path.push("fixtures/network_big.xml");
    let links = Links::from_xml(path, false).unwrap();
    let mut rng = StdRng::seed_from_u64(1234);
    let mut link_keys: Vec<_> = (0..links.lengths.len()).collect::<Vec<_>>();
    link_keys.sort();
    let samples = black_box(link_keys.iter().choose_multiple(&mut rng, 1000));

    c.bench_function("query link lengths", |b| {
        b.iter(|| {
            for lid in &samples {
                links.get(**lid);
            }
        })
    });
}
```

We can access each length in about 3µs.


# Fiddling

## Quick_xml Checks

Disabling the quick_xml checks (`let mut iter_attributes = attributes.with_checks(false).flatten();`) turns out to save us 5% loading time - which seems well worth any risk. So we keep the checks removed.

## Precision

f32 is probably overkill. We can reasonably expect links to be between several meters and several kms long. We can also expect them to always be positive. We also might not care much about decimal places. So I check the impact of switching to u16 (unsigned integer with maximum around 65k).

No change to our benchmarks. This is expected. We actually have to still parse the xml lengths as floats before converting them to integers (maybe we could to something better).

We do know that we have halved the link lengths memory requirement (at least in theory) and can get faster subsequent operations with integers. But it is at the expense of precision, which might cause problems later on. For example I know we will need to calculate duration on a link for a given speed. This will inneviatbly be a float operation and for short links, precision will matter.

So we could try f16 from [half](https://docs.rs/half/latest/half/index.html). But reading the docs I see that we can only guarantee memory improvements, as not all hardware will support f16 operations. But we've come this far and so i set up a new benchmarks that extracts the link lengths and then makes some arbitrary operations. This turns out to be about 5% slower using half::f16, presumably as it adds a bunch of conversions on my hardware.

So to summarise, f32 seems fine for now. If it turns out that precision doesn't matter as much but memory does. Then there are some easy memory wins, but they might come at the expense of inprecise or slower ops.

## Strings

We need to create some data in memory for each key (the fast_xml buffer from which we parse the key is transient). Using `String` is not great. We immediately have to allocate to heap and we get some functionality (`.push()`) that we don't need - our keys can be immutable.

[compact_str](https://docs.rs/compact_str/0.1.0/compact_str/struct.CompactStr.html) has *"an immutable a memory efficient immuatable string that can be used almost anywhere a String or &str can be used".* Sounds good but seems to cause a 5% regression for our hashmap accessing benchmark. Not really sure why but I'm expecting to hit the hashmap a lot so no good.

## BTreeMap

Replacing the HashMap with a BTreeMap very bad idea. Writting is 5% slower. Access 300% slower.

## Complete reindex

I already know that I am going to be looking up link lengths (and other values) a lot. The fastest way to do this would be to dump the length values into an array and use a `usize` key to directly access the array.

To be actually useful I add an `Indexer`, which provides methods to (i) consistently map keys into `usize` (the same key should always return the same `usize`), and (ii) I need to be able to retrieve my keys again using the new `usize`:

```rs
use std::collections::HashMap;

#[derive(Debug, Default)]
pub struct Indexer {
    key_to_index: HashMap<String, usize>,
    index_to_key: Vec<String>,
}

impl Indexer {
    pub fn new() -> Self {
        Indexer::default()
    }
    pub fn add(&mut self, key: String) {
        let index = self.index_to_key.len();
        // todo - remove clone
        self.key_to_index.insert(key.clone(), index);
        self.index_to_key.push(key);
    }

    pub fn get_key(&self, index: usize) -> Option<String> {
        // todo - has a clone - prefer to return a ref
        if index <= self.index_to_key.len() {
            return None
        }
        Some(self.index_to_key[index].clone())
    }
}
```

At xml read time this builds a map of keys to `usize` plus puts a clone of the keys in a vec so they can be retrieved using the index. This is a little more work and about double the memory, but only causes a 5% regression on xml read time. We can probably engineer out the clone and fuss around with the String size in future.

Accessing link lengths using the new `usize` is obviously a lot faster (~95% faster), we no longer have to worry about the hash, instead we immediately fetch with the vec index. Provided we are going to be doing more a lot more accessing versus creating then this is a no-brainer.

---
**NOTE**

WIP
___