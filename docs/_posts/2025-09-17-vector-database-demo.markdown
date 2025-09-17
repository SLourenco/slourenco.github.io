---
layout: post
title:  "Intro to Vector Databases"
date:   2025-09-16 10:00:00 +0100
categories: [database vector embeddings llm]
---

A vector database stores its primary data in vector format, allowing searches of the closest results.
This type of database is normally used in similarly, semantic or multi-model search, recommendation engines, LLMs and others.

To grasp the basic principles of a vector database, I've decided to create a very simple and naive database in rust.
There are 3 main parts of a vector database to think about:
 - Data structures and file storage in vector format
 - Creating a function that turns some data into a vector of embeddings
 - Creating a similarity function, that returns the closest results of a query

To explore a bit more, I'm also adding some metadata filters (not based on vector data) and some indexes to the data.

For this example, I'll create a movie database that can recommend similar movies to the searched one.

# Data structures

To start out, I'll create 3 basic structs to represent the tables, records and database.
For this example, I'll store the movies vector, as well as the name and runtime for the metadata.
Each table will represent a file in disk, with the record data.

```rust
#[derive(Default)]
pub(crate) struct Storage {
    tables: BTreeMap<String, Table>,
}

#[derive(Default, Clone)]
pub(crate) struct Table {
    pub name: String,
    pub size: usize,
}

#[derive(Serialize, Deserialize)]
pub(crate) struct Record {
    pub id: usize,
    pub name: String,
    pub runtime_minutes: usize,
    pub vector: Vec<usize>,
}
```

Now I'll need to initialize the database, creating the necessary files when the database and tables are created.
Also, if files are already present in the data folder, we will need to load the file metadata into memory so we achieve durability across restarts.

```rust
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
```
With the database properly initialized, the next step is to be able to add new data to it.
For that, I'll create a new add data method in the table struct.

```rust
impl Table {
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
}
```

And now I can populate the table with some data from a json file, for example.
But before I can do that, I must be able to calculate an embedding vector from some data.

## Embeddings

One of the key parts of a vector database is the encoding of the vectors into embeddings.
These embeddings represent the data across multiple dimensions. 
These dimensions are the key to identify the "similarity" between the data points.

There are multiple ways to do this, from simple embedding functions to using complex LLM models from OpenAI and other companies.

For this exercise, I'm going to create a very simple embedding function based on a movie category and runtime.
This means that movies will be considered similar if they are in the same category and have similar runtimes.

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

The actual value I gave each category is relevant here. 
Since the similarity function will calculate the distance between vectors, 
so the closest numbers will be more similar than the other ones.
So, in this example, `comedy (2)` will be closer to `action (3)` than to `drama (6)`.
Depending on the similarity function, this is not as direct as `abs(2-3) < abs(2-6)`,
but the basic principle is that the value I choose for the category code is not irrelevant.

Real world use cases will rely on more complex and nuanced functions. If you want to explore using LLMs with these, you can explore the [Open AI docs on embeddings](https://platform.openai.com/docs/guides/embeddings).

---
<br/>
Now that I have an embedding function, I can populate my database. 
I want to do this when I start my program, if the storage is empty, to make it easier for me to test.

```rust
#[derive(Serialize, Deserialize, Debug)]
struct Movie {
    name: String,
    category: String,
    runtime: usize,
}

fn initialize_db() -> Table {
    println!("Initializing vector database...\n");
    let mut db = storage::init_storage().expect("Failed to initialize storage");
    let mut movies = db
        .init_table("movies")
        .expect("error creating movies table");

    if movies.size == 0 {
        let movies_json = parse_movies_file();
        for (idx, movie) in movies_json.into_iter().enumerate() {
            movies
                .add_data(table::Record {
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
    }
    movies
}

fn parse_movies_file() -> Vec<Movie> {
    let file = fs::File::open("movies.json").expect("file should open read only");
    let movies: Vec<Movie> = serde_json::from_reader(file).expect("file should be proper JSON");
    movies
}

fn main() {
    let movies = initialize_db();
    // ...
}
```

To test this database, I will use some static data inserted from a json file.

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

# Search

With a database with some data, I want to start getting recommendations for similar movies.
For that, I'm going to need a similarity score function to power my search method.

## Similarity Score

The search function works by finding the closest vectors to the query. 
Distance can be calculated in several ways, with a common one being the [cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity).

Cosine similarity gives the cosine of the angle between 2 n-dimensional vectors.
This is calculated as the dot product of the vectors divided by the product of the vector length.

I'll create a method to return the cosine similarity between 2 vectors:

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

Now I can write a search function. It goes through each stored vector and calculates the distance between it and the query vector.
Then returns the top n results.

```rust
impl Table {
    // ...
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
}
```

And with this I can update my main method to return the list of similar movies:

```rust
fn main() {
    let movies = initialize_db();
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
```

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

Adding more and more data would show the limitations of my embeddings function,
as these 2 dimensions are hardly enough to properly represent the movies.
But the basic idea does not change.

--- 
<br/>
With the basic functionality down, I want to expand my system a little bit, adding metadata filters and indexes.

# Metadata filters

In most cases, the search query will have some filters that are not related with the embeddings vector.
This filtering would work as it does for other databases, by looking up some column value.

I might want to limit the similar movies to the ones with the same category or recommend movies within a specific runtime limit.
As an example I'll limit the runtime of the results to 180 minutes.
The search function would look like:

```rust
impl Table {
    // ...
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
}
```

This type of pre-filtering reduces the number of vectors the search method will look through. 

---

# Indexes

For any moderately sized dataset the topic of indexation will be relevant.
Indexation avoids searching through the entire dataset to get the results.
For this exercise, I will be introducing a basic **Inverted File Flat index** (IVF FLAT).

In IVF FLAT the data is grouped into clusters, using clustering techniques such as K means clustering. 
When searching for results, the centroid of each cluster is used to determine the candidate cluster to drill down.

![Image with initial search step: checking closes centroid](/assets/vector-database-demo/vector-database-index-1.png)

On that cluster, all vectors are processed for the search query.

![Image with second search step: brute force on cluster](/assets/vector-database-demo/vector-database-index-2.png)

### Creating an index file

To start, I'll create a file to store the index.
An index will be stored as a collection of index entries. Each entry will include:
- the filename of the file that stores the vectors of each cluster,
- the centroid that represents the cluster.

The file format we will use will be `{tablename}_partX.vdbx`, where X a random number representing the cluster.
With this change, the original `vdb` file will now store the index entries, while each part file will store the actual data.

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

I'll also change the data insertion method, to populate the index on insert:

```rust
impl Table {
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
    // ...
}
```

For the index not to become unbalanced (some clusters a lot larger than others) and to accurately represent the data it contains, it needs to be recalculated periodically.
This means doing the k-means algorithm on the entire dataset, or at least to a subset of it that had new data.
I'll leave this out of this example though.

With the new index, the search method can be updated to make use of it:

```rust
impl Table {
    // ...
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
}
```

With the indexes, the search will first find the closest centroid, and then search the data points of that cluster for the closest results.
This significantly reduces the amount of files looked up.

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

_The full source code for this article is in [my github repo](https://github.com/SLourenco/demo-vector-database)._
