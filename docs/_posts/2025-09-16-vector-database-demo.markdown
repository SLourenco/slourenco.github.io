---
layout: post
title:  "Intro to Vector Databases"
date:   2025-09-16 10:00:00 +0100
categories: [database vector embeddings llm]
---

A vector database is database that stores its primary data in vector format, commonly allowing searches of the closest results.
This type of database is normally used in similarly, semantic or multi-model search, recommendation engines, LLMs and others.

To grasp the basic principles of a vector database, I've decided to create a very simple and naive database in rust.

The main parts of the vector database will be:

 - Data structures that support vectors
 - Converting some data into vector (embedding)
 - Comparing vectors through a similarity score function
 - Experimenting with metadata filtering and indexing

# Data structures

To start our vector database, let's define the basic struct for it.
We will need some storage to store all the created tables, and each table must contain the list of embedding vectors.

```rust
#[derive(Default)]
pub(crate) struct Storage {
    tables: BTreeMap<String, table::Table>,
}

pub(crate) struct Table {
    pub name: String,
    pub records: Vec<Record>,
}

pub(crate) struct Record {
    pub id: usize,
    pub name: String,
    pub runtime_minutes: usize,
    pub vector: Vec<usize>,
}
```

For each table, we will want to be able to insert records and search for the most similar results

```rust
impl Storage {
    pub fn create_table(&mut self, name: &str) -> Result<&mut table::Table, Error> {
        let table = table::Table {
            name: String::from(name),
            records: Vec::new(),
        };

        self.tables.insert(String::from(name), table);
        self.tables.get_mut(name).ok_or(Error::new(
            ErrorKind::Other,
            "could not find table inserted",
        ))
    }

    pub fn get_table(&mut self, name: &str) -> Result<&mut table::Table, Error> {
        self.tables
            .get_mut(name)
            .ok_or(Error::new(ErrorKind::NotFound, "table does not exist"))
    }
}

impl Table {
    pub fn add_data(&mut self, data: Record) -> Result<usize, Error> {
        self.records.push(data);
        Ok(1)
    }

    pub fn search(
        &self,
        vec: Vec<usize>,
        limit: usize,
        similarity: &str,
    ) -> Result<Vec<(f64, &Record)>, Error> {
        panic!("not implemented")
    }
}
```

To test this database, we can have some static data inserted from a json file and then search for a specific title.

Json
```json
[
  {
    "name": "Dune",
    "category": "sci-fi",
    "runtime": 137
  },
  {
    "name": "DragonHeart",
    "category": "fantasy",
    "runtime": 103
  },
  {
    "name": "Solaris",
    "category": "sci-fi",
    "runtime": 166
  },
  {
    "name": "Lord of the Rings - The Two Towers",
    "category": "fantasy",
    "runtime": 179
  },
  {
    "name": "Lord of the Rings - The Return of the King",
    "category": "fantasy",
    "runtime": 201
  }
]
```

Our main script
```rust
use crate::embeddings::calculate_embeddings;
use serde::{Deserialize, Serialize};
use std::fs;

mod embeddings;
mod storage;

#[derive(Serialize, Deserialize, Debug)]
struct Movie {
    name: String,
    category: String,
    runtime: usize,
}

fn main() {
    println!("Initializing vector database...\n");
    let mut db = storage::Storage::default();
    let movies = db
        .create_table("movies")
        .expect("error creating movies table");

    let movies_json = parse_movies_file();
    for (idx, movie) in movies_json.into_iter().enumerate() {
        movies
            .add_data(storage::table::Record {
                id: idx,
                name: movie.name.clone(),
                runtime_minutes: movie.runtime,
                vector: calculate_embeddings(
                    movie.name.as_str(),
                    movie.category.as_str(),
                    movie.runtime,
                ),
            })
            .expect(format!("error adding {} to movies table", movie.name).as_str());
    }

    println!("Searching for similar movies...\n");
    let similar_movies = movies
        .search(
            calculate_embeddings("Lord of the Rings - Fellowship of the Ring", "fantasy", 178),
            2,
            "cosine",
        )
        .expect("error searching for similar movies");
    for similar_movie in similar_movies {
        println!("Recommendation:");
        println!(
            "Title: {}\nRuntime: {} (minutes) \n",
            similar_movie.1.name, similar_movie.1.runtime_minutes
        );
    }
}

fn parse_movies_file() -> Vec<Movie> {
    let file = fs::File::open("movies.json").expect("file should open read only");
    let movies: Vec<Movie> = serde_json::from_reader(file).expect("file should be proper JSON");
    movies
}
```

## Calculating embeddings

One of the key parts of a vector database is the encoding of the vectors into embeddings.
There are multiple ways to do this, from simple embedding functions to using complex LLM models from OpenAI and other companies.

For this exercise, let's do a very simple and naive function:

```rust
pub fn calculate_embeddings(movie: &str, category: &str, runtime: usize) -> Vec<usize> {
    vec![movie.len(), get_category_code(category), runtime]
}

fn get_category_code(category: &str) -> usize {
    match category {
        "sci-fi" => 1,
        "comedy" => 2,
        "action" => 3,
        "romance" => 4,
        "horror" => 5,
        "drama" => 6,
        "fantasy" => 7,
        "thriller" => 8,
        "animation" => 9,
        "adventure" => 10,
        _ => 999,
    }
}
```

If you want to explore using LLMs with these, you can explore the [Open AI docs on embeddings](https://platform.openai.com/docs/guides/embeddings).

## Persisting the records

A basic characteristic of any database is the ability to persist data across server restarts. 
To emulate that, let's store our data into disk files.

To start, let's update our add_data method

```rust
    pub fn add_data(&mut self, data: Record) -> Result<usize, Error> {
    let mut file = OpenOptions::new()
        .write(true)
        .append(true)
        .open(self.name.clone())?;

    let d = serde_json::to_string(&data)?;
    file.write_all(d.as_bytes())?;
    file.write_all(b"\n")?;
    self.size += 1;
    Ok(1)
}
```

And now, on our search method we need to load the data from files:
```rust
pub fn search(
    &self,
    vec: Vec<usize>,
    limit: usize,
    similarity: &str,
) -> Result<Vec<(f64, Record)>, Error> {
    let file =
        BufReader::new(File::open(self.name.clone()).expect("Unable to open table file"));

    let mut results = Vec::new();
    for line in file.lines() {
        let record: Record = serde_json::from_str(line?.as_str())?;
        if similarity == "cosine" {
            let s = similarity::cosine_similarity(vec.clone(), record.vector.clone());
            results.push((s, record));
        }
    }

    results.sort_by(|a, b| b.0.partial_cmp(&a.0).unwrap());
    results.truncate(limit);
    Ok(results)
}
```

Lastly, we need to create an initialization method to ensure the files exist:
```rust
pub(crate) fn init_storage() -> Result<Storage, Error> {
    fs::create_dir_all("database")?;
    let mut storage = Storage::default();
    for entry in fs::read_dir("database")? {
        let path = entry?.path();
        let Some(file) = path.file_name() else {
            return Err(Error::new(ErrorKind::InvalidData, "Path is not a file"));
        };
        let Some(filename) = file.to_str() else {
            return Err(Error::new(
                ErrorKind::InvalidData,
                "File does not have a name",
            ));
        };

        let table = table::init_table(String::from(format!("database/{}", filename)))?;
        storage.tables.insert(String::from(filename), table);
    }
    Ok(storage)
}

pub(crate) fn init_table(full_filename: String) -> Result<Table, Error> {
    let path = Path::new(full_filename.as_str());
    if !path.exists() {
        File::create(full_filename.clone())?;
        return Ok(Table {
            name: full_filename,
            ..Default::default()
        });
    }

    let file =
        BufReader::new(File::open(full_filename.clone()).expect("Unable to open table file"));
    let mut cnt = 0;

    for _ in file.lines() {
        cnt = cnt + 1;
    }

    Ok(Table {
        name: full_filename,
        size: cnt,
        ..Default::default()
    })
}
```

## Similarity Score

The search function works by finding the closest vectors to our query. 
Distance can be calculated with several functions, with a common one being the [cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity).

Cosine similarity gives the cosine of the angle between 2 n-dimensional vectors.
This is calculated as the dot product of the vectors divided by the product of the vector length.

Let's create a method to return the cosine similarity between 2 vectors:

```rust
pub fn cosine_similarity(a: Vec<usize>, b: Vec<usize>) -> f64 {
    let mut sum_of_product = 0;
    let mut a_norm = 0;
    let mut b_norm = 0;
    for i in 0..a.len() {
        sum_of_product += a[i] * b[i];
        a_norm += a[i] * a[i];
        b_norm += b[i] * b[i];
    }

    (sum_of_product as f64) / (f64::sqrt(a_norm as f64) * f64::sqrt(b_norm as f64))
}
```

Now we can write our search function. It goes through each stored vector and calculates the distance between it and our query vector.
We can then return the top n results.

```rust
    pub fn search(
        &self,
        vec: Vec<usize>,
        limit: usize,
        similarity: &str,
    ) -> Result<Vec<(f64, Record)>, Error> {
        let file =
            BufReader::new(File::open(self.name.clone()).expect("Unable to open table file"));

        let mut results = Vec::new();
        for line in file.lines() {
            let record: Record = serde_json::from_str(line?.as_str())?;
            if similarity == "cosine" {
                let s = similarity::cosine_similarity(vec.clone(), record.vector.clone());
                results.push((s, record));
            }
        }

        results.sort_by(|a, b| b.0.partial_cmp(&a.0).unwrap());
        results.truncate(limit);
        Ok(results)
    }
```


## Adding filters

In most cases, the search query will have some filters that are not related with the embeddings vector.
This filtering would work as it does for other databases, by looking up some column value.

For example, let's say we want to limit the runtime of the results to 180 minutes.
Our search function would look like:

```rust
   pub fn search(
        &self,
        vec: Vec<usize>,
        runtime_limit: usize,
        limit: usize,
        similarity: &str,
    ) -> Result<Vec<(f64, Record)>, Error> {
        let file =
            BufReader::new(File::open(self.name.clone()).expect("Unable to open table file"));

        let mut results = Vec::new();
        for line in file.lines() {
            let record: Record = serde_json::from_str(line?.as_str())?;
            if record.runtime_minutes > runtime_limit {
                continue
            }

            if similarity == "cosine" {
                let s = similarity::cosine_similarity(vec.clone(), record.vector.clone());
                results.push((s, record));
            }
        }

        results.sort_by(|a, b| b.0.partial_cmp(&a.0).unwrap());
        results.truncate(limit);
        Ok(results)
    }
```

This type of pre-filtering reduces the number of vectors we have to look through and improves the accuracy of our search queries. 

## Adding Indexes

For any moderately sized dataset the topic of indexation will be relevant.
Indexation allows us to avoid searching through the entire dataset to get the results we want.
For our exercise, we will be introducing a basic **Inverted File Flat index** (IVF FLAT).

In IVF FLAT we group the data into clusters, using clustering techniques such as K means. 
When searching for results, the centroid of each cluster is used to determine the candidate cluster to drill down.

![Image with initial search step: checking closes centroid](/assets/vector-database-demo/vector-database-index-1.png)

On that cluster, all vectors are processed for our search query.

![Image with second search step: brute force on cluster](/assets/vector-database-demo/vector-database-index-2.png)

Let's update our code to support this type of index.

### Creating an index file

We start by creating a file to store the index. To do so, let's first define what our index looks like:
An index will be stored as a collection of index entries. Each entry will include:
- the filename of the file that stores the vectors of each cluster,
- the centroid that represents the cluster.

The file format we will use will be `{tablename}_partX.vdbx`, where X a random number representing the cluster.
With this change, our original `vdb` file will now store the index entries, while each part file will store the actual data.

```rust
use crate::storage::table::Table;
use rand::Rng;
use serde::{Deserialize, Serialize};
use std::fs::{File, OpenOptions};
use std::io::{BufRead, BufReader, Error, Write};

pub(crate) trait Index {
    fn read_index_file(&self) -> Result<Vec<IndexEntry>, Error>;
    fn add_new_center(&self, data: Vec<usize>) -> Result<IndexEntry, Error>;
    fn number_of_clusters(&self) -> usize {
        2
    }
}

impl Index for Table {
    fn read_index_file(&self) -> Result<Vec<IndexEntry>, Error> {
        let file =
            BufReader::new(File::open(self.name.clone()).expect("Unable to open table index file"));

        let mut index = Vec::new();
        for line in file.lines() {
            let entry: IndexEntry = serde_json::from_str(line?.as_str())?;
            index.push(entry);
        }
        Ok(index)
    }

    fn add_new_center(&self, data: Vec<usize>) -> Result<IndexEntry, Error> {
        let mut file = OpenOptions::new()
            .write(true)
            .append(true)
            .create(true)
            .open(self.name.clone())?;

        let idx = IndexEntry {
            filename: format!(
                "{}_part{}.vdbx",
                self.name,
                rand::rng().sample(rand::distr::Alphanumeric)
            )
                .to_string(),
            center: data,
        };
        let d = serde_json::to_string(&idx)?;
        file.write_all(d.as_bytes())?;
        file.write_all(b"\n")?;

        Ok(idx)
    }
}

#[derive(Default, Clone, Serialize, Deserialize)]
pub(crate) struct IndexEntry {
    pub filename: String,
    pub center: Vec<usize>,
}
```

Now let's change our data insertion method, to populate the index on insert:

```rust
pub fn add_data(&mut self, data: Record) -> Result<usize, Error> {
        let index = self.read_index_file()?;
        let filename;
        if index.len() < self.number_of_clusters() {
            let index_entry = self.add_new_center(data.vector.clone())?;
            self.indexes.push(index_entry.clone());
            filename = index_entry.filename;
        } else {
            let index_entry = self.closest_index(data.vector.clone())?;
            filename = index_entry.filename;
        }

        let mut file = OpenOptions::new()
            .write(true)
            .append(true)
            .create(true)
            .open(filename.clone())?;

        let d = serde_json::to_string(&data)?;
        file.write_all(d.as_bytes())?;
        file.write_all(b"\n")?;
        self.size += 1;
        Ok(1)
    }
```

As you might've noticed, we should recalculate the clusters periodically, to ensure the data is properly distributed.
But let's leave that part out for now.

With the new index, our search method can be updated to make use of it:

```rust
pub fn search(
        &self,
        vec: Vec<usize>,
        limit: usize,
        similarity: &str,
    ) -> Result<Vec<(f64, Record)>, Error> {
        let index = self.closest_index(vec.clone())?;
        let file =
            BufReader::new(File::open(index.filename.clone()).expect("Unable to open table file"));

        let mut results = Vec::new();
        for line in file.lines() {
            let record: Record = serde_json::from_str(line?.as_str())?;
            if similarity == "cosine" {
                let s = similarity::cosine_similarity(vec.clone(), record.vector.clone());
                results.push((s, record));
            }
        }

        results.sort_by(|a, b| b.0.partial_cmp(&a.0).unwrap());
        results.truncate(limit);
        Ok(results)
    }
```

And now we are doing indexed search. By looking up the index file first, we avoid having to process a lot of data.

```stdout
Initializing vector database...

Searching for similar movies...

Recommendation:
Title: Lord of the Rings - The Return of the King
Runtime: 201 (minutes) 

Recommendation:
Title: Lord of the Rings - The Two Towers
Runtime: 179 (minutes) 
```

If you are interested in reading more about indexing techniques for vector databases, I've found [this article](https://medium.com/@myscale/understanding-vector-indexing-a-comprehensive-guide-d1abe36ccd3c) to be an interesting read.




---
Example products:
  - Example database vectors (qdrant, ...)
  - Compare basic APIs

Code example:
  - Building our basic in memory db
  - Using cosine distance function
  - Embeddings function
    - Extension to more complex uses, like LLM models

- Add links to github, etc...

## Delete all below this

You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

Jekyll requires blog post files to be named according to the following format:

`YEAR-MONTH-DAY-title.MARKUP`

Where `YEAR` is a four-digit number, `MONTH` and `DAY` are both two-digit numbers, and `MARKUP` is the file extension representing the format used in the file. After that, include the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
