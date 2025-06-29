---
title: "Pages and Layouts in Mint"
date: 2025-06-29T12:00:00+05:30
draft: false
description: "Understand how memory pages work in operating systems, and how Mint uses them to store structured data safely."
tags: ["mint", "database", "storage", "OS"]
cover:
  image: "images/pages_and_layout_in_mint.png"
  alt: "A diagram of memory pages with a subtle notebook aesthetic"
  relative: true
  hidden: false
---
## Pages

Pages are fixed-size blocks of memory. Think of them like pages in a notebook. Each page has a fixed size, and any data written to memory is written to one or more pages.

Now let’s take a step back and look at a computer’s memory. There are two types:

- Persistent (like SSD or HDD)
- Volatile (RAM)

If we want some data to persist even after our program ends or the system restarts, it must be saved in persistent storage. RAM is fast but temporary. The downside with persistent storage is that it's slower. HDDs used to be slow due to their moving parts. SSDs fixed that, but even now, they’re still slower than RAM.

Let’s say we want to read 12 bytes from a file, change them, and save them back.

Here’s what happens:

- The program asks the OS for the data.
- OS first checks something called the **page cache**, which is just a memory area in RAM that stores recently used disk data.
- If it’s already in the page cache, we get it from RAM.
- If not, the OS goes to the disk, loads the page into RAM, and then gives it to the program.
- We change the data. That memory page is marked **dirty**, which means it has been changed but not yet saved to disk.
- At some point, the OS flushes the dirty page to disk. If we want to make sure it’s really saved, we need to tell it explicitly by calling something like `fsync`.

Sounds like a lot, but that’s where **page size** comes into play.

If the OS fetched data one byte at a time, reading 12 bytes would need 12 trips to the disk. That’s bad because the data you need might not be stored sequentially on the disk. The 12 bytes could be scattered throughout the disc, and fetching from a scattered source is time-consuming than from a sequential source.

But say the page size is 4 bytes. Then it just needs 3 trips. A little better.

Now, someone might ask, why not just make page size 12 bytes and get all of it in one go? That sounds fine, but there are downsides.

- If the OS loads a large page from disk and only a tiny part of it is used, the rest is wasted. That’s called **internal fragmentation**.
- Every time a page is not in memory, it has to be fetched from disk. The bigger the page, the more time it takes.

So the OS picks a fixed size (usually 4 KB), which is a tradeoff between too many trips and too much wasted space.

Every program then uses this page size when reading or writing data.

---

## Page layout

Now that we understand what a page is, we need to understand how we can structure a page so that it makes sense to us. Without a structure, a page is just a continuous block of memory with some data in it. It is the work of the user (programmer) to give some meaning to a page.

Consider a page from a book; it generally has 3 sections - header, body, and footer. This is a structure. If the writer decided to jumble these sections on each page of his/her book, readers would be confused and clueless.

Page layouts serve a similar purpose. They give meaning/context to the data that resides in a page. OS does not direct a user on what the layout should be. It is dependent on the program and its purpose.

Let's try to define a page layout for a fictional program that stores 3 numbers - 2 are addends (2 numbers used for addition), and one is the sum of those 2 numbers.
- Size of the page - 4KB (4096 Bytes)
- Size of a single addend - 8B (64-bit integer)
- Size of the sum - 8B (64-bit integer)

With the given information, we can understand that:
- Each entry is 24 bytes in size (8 bytes + 8 bytes + 8 bytes)
- A single page can hold up to 170 entries (4096 bytes / 24 bytes)

But we also might need some metadata for each page - page number. Let's add this to a section called header. So, each header of a page starts at the 0th byte of the page to the 7th byte of the page. This gives us 4088 bytes to write our data, and the data section starts from the 8th byte.

So, the layout looks something like:

| Header (8 bytes)       | Operation 1 (24 bytes)     | Operation 2 (24 bytes)     | Operation 3 (24 bytes)     | Operation 170 (24 bytes)                    |
| ---------------------- | -------------------------- | -------------------------- | -------------------------- | -------------------------------- |
| metadata - page number | addend 1 + addend 2 = sum 1 | addend 3 + addend 4 = sum 2 | addend 5 + addend 6 = sum 3 | addend 339+ addend 340 = sum 170 |

---

## Mint's page layout

Each page in Mint has a specific layout. Before getting into that, let's see what are the types of pages in Mint.
- Metadata Page - This page contains all the references to other pages in the database. Meta page is analogous to an index page in a book. This is the first page in the entire collection of pages.
- Free pages list - This page references all the pages that were created but are not being used to read data from. This page is used during database writes, or new page creation when an index tree has to split, or any other page overflows, and needs extension. Consider this like a recycle bin for pages.
- Index pages - This page references the nodes of the index tree. They contain internal, or leaf nodes of an index tree. All the reads and writes to the database happen here. This forms the core of our database and needs the most attention.
- Overflow page - Any data inserted into the database might have the potential to overflow beyond the scope of a page. This overflow could be due to a lengthy key or value. Overflow pages provide a way to extend the index pages to contain relevant data.

These are the basic types of pages in our database. More can be added later if needed, as long as they’re documented. Any new type of page can be added to the database, provided the existing ones do not facilitate the required feature. Make sure that these changes are well documented.

With the meaning of a page and the types of pages in place, we can define the size of a single page. Choosing the right page size can be a tricky question to answer, and each answer has its trade-offs. Databases like SQLite allow the user to pick a specific size (a number which is a power of 2, between 512 and 32768). This provides flexibility, but the tradeoff is that if the OS's page cache size is less than the database's page size, fetching a single page means multiple disk IOs. If the OS's page cache size is greater than the database's page size, fetching a single page means overfetching, for a small portion of relevant data. Mint uses the OS’s default page size. That means fewer surprises. It also avoids under-fetching or over-fetching during disk I/O. This reduces flexibility but improves performance by avoiding under-fetching and over-fetching disk IOs. Go provides a convenient method to fetch the underlying OS's page size: ```pageSize := os.Getpagesize()```.

### Page header
Coming to the layout of a page, each page has a generic header that contains all the metadata for easy navigation through the collection. Below is a definition of the current metadata. These can change to facilitate certain features and requirements. Always remember to document these changes and make sure that the changes are backward compatible. Data points can be added, but removing any is highly discouraged.

Now, let's define a generic header for a page:
| Offset (in bytes) | Size (in bytes) | Key           | Description                                                                 |
|--------|------------------|---------------|-----------------------------------------------------------------------------|
| 0      | 2                | `HeaderVersion` | A version tag for the page header format. Helps with backward compatibility. |
| 2      | 2                | `PageType`      | Encodes the type of page (e.g., internal, leaf, metadata) using constants.  |
| 4      | 8                | `PageID`        | Unique identifier for this page. Used to find the page offset in the file. It is a logical entry which helps us fetch the page offset with the formula ```PageSize*PageID```  |
| 12     | 8                | `NextPageID`    | ID of the next page in a linked list or sequence. `0` indicates end.        |
| 20     | 8                | `PrevPageID`    | ID of the previous page. `0` indicates the start of the sequence.           |
| 28     | 8                | `Checksum`      | Checksum of the page content (excluding this header) for integrity checks.  |
| 36     | 27               | `Reserved`      | Reserved space for future header extensions. Ignored during read/write now.|

Each field serves a specific purpose (mentioned in the description) and is assigned to a specific place in memory. There is a reserved section from offset 36 to byte 63. These can be used to extend the header further. Extending the header should mean that the header_version is incremented, serialization and deserialization logics are updated, and making sure that the whole change is backward compatible.

The length of the header is fixed to 64 bytes (0 - 63 bytes), and this forms the first 64 bytes of a page.

### Page definitions

#### Metadata Page
The metadata page is the first page of any Mint database files. They function like an anchor point for multiple other types of pages. Let's define the metadata page's structure.

| Offset (in bytes) | Size (in bytes) | Key                  | Description |
|-------------------|------------------|-----------------------|-------------|
| 0                 | 64               | `Header`             | Standard page header shared by all pages. Contains metadata like page ID, type, version, etc.
| 64                | 8                | `FreePageRootID`  | Page ID of the root of the free page list. Used to track reusable pages. |
| 72                | 8                | `IndexRootPageID` | Page ID of the root of the index structure (e.g., B-Tree). |
| 80                | PageSize - 80    | `Reserved`           | Reserved space for future metadata extensions without breaking compatibility. |

#### Free Pages List Page
The free pages list page contains a list of all the free pages available to be written. These pages are added to the list when data is added or deleted from the database. Maintaining a free pages list helps us reuse unused or stale pages, without creating new ones. This ensures that minimal resources are allocated and any allocated resource is utilized to their maximum potential. Let's define the free page list page's structure.

| Offset (in bytes) | Size (in bytes)  | Key          | Description |
|-------------------|------------------|--------------|-------------|
| 0                 | 64               | `Header`    | Standard page header. Contains `page_type`, `page_id`, `next_page_id`, etc. |
| 64                | variable         | `[]Entries` | List of reusable page entries. Each is 10 bytes long. |

Entry Format

| Field       | Size (bytes) | Description                                  |
|-------------|---------------|----------------------------------------------|
| `PageType` | 2             | Type of the reusable page (same enum used in header) |
| `PageID`   | 8             | ID of the page that can be reused            |

#### Index Page
The index page is the most important element in the database. It contains all the data, distributed across multiple pages for fast and efficient queries. Specific data structures are chosen to arrange the data across pages. Some of the widely used structures are: B+ Tree, LSM Tree, Hash Table, Inverted Index, Bitmap Index, etc. The definition and the layout of the index page depend on the implementation of the underlying data structure. To facilitate flexibility and extensibility, we declare a metadata region. This region encapsulates all the necessary metadata required for the data structure's implementation. Let's define a generic structure for an index page.

| Offset (in bytes) | Size (in bytes) | Key       | Description                                                                 |
|-------------------|-------------------|-----------|-----------------------------------------------------------------------------|
| 0                 | 64                | `Header`    | Contains standard page header (page ID, type, next/prev page ID, etc.)     |
| 64                | 64                | `Metadata`  | Index-specific metadata (algorithm, node type, number of keys, flags, etc.)           |
| 128               | PageSize - 128    | `Data`      | Actual index data: keys, values, or child pointers depending on the algorithm's implementation  |

#### Overflow Page
The overflow page is an extension of the index page for storing data that overflowed from the initial page. This should not be used to create a new entry, but only to complete an incomplete entry. It is the responsibility of the Index Page developer to utilize overflow pages as they see fit. Let's define a generic structure for an overflow page.

| Offset (in bytes) | Size (in bytes) | Key      | Description                                                                 |
|-------------------|-------------------|----------|-----------------------------------------------------------------------------|
| 0                 | 64                | `Header`   | Standard page header (includes page_id, page_type, next_page_id, etc.)     |
| 64                | PageSize - 64     | `Data`     | Overflow content — spillover data from another page (usually large values) |

---

### Notes

-  All of this is **specific to Mint**, but the concepts are universal. If you’re building your own store, you might tweak the layout or add more page types, and that’s totally fine.
-  All integers (like `PageID`, `PageType`, etc.) are encoded in **little endian** format — the same as what most modern machines (and Go’s binary encoding) expect. This ensures cross-platform readability.
-  **PageID math is important** — `PageOffset = PageSize * PageID`. This lets us jump directly to a page using basic arithmetic.
-  You **should not remove** existing header fields. If you really need to change something, add a new field in the reserved section and bump up the `HeaderVersion`.
-  **Overflow pages aren’t standalone** — they’re sidekicks, used only when the primary page can’t fit everything. Don’t start writing fresh entries directly into overflow pages.
-  **Free page lists matter.** Without them, every delete would just waste space. With them, we get recycling — fewer file writes, less bloat.
-  If you're experimenting with different **index structures** (like B+ Trees or LSM Trees), the index page metadata section is your playground. Define what you need, document it, and keep it extensible.
-  Mint sticks to the OS's page size. It keeps things simple and avoids the pain of partial reads or misaligned IO.
> 🧩 **Footnote:** This post is just one piece of a bigger puzzle. *Mint* isn’t fully built yet. This article is part of its ongoing design and development process.

---
If you want to dive deeper:
---

- [Page - computer memory (Wikipedia)](https://en.wikipedia.org/wiki/Page_(computer_memory))

- [Paging in OS (Scaler)](https://www.scaler.com/topics/paging-in-os/)

- [Computer data storage (Wikipedia)](https://en.wikipedia.org/wiki/Computer_data_storage)

- [Breaking Down the Anatomy of a Database Page (Medium)](https://medium.com/databases-in-simple-words/breaking-down-the-anatomy-of-a-database-page-90e751d018fe)

- [Database Pages — A deep dive (Medium)](https://medium.com/@hnasr/database-pages-a-deep-dive-38cdb2c79eb5)

- [Database Page Layout (Postgres)](https://www.postgresql.org/docs/current/storage-page-layout.html)

- [Database File Format (SQLite)](https://www.sqlite.org/fileformat.html)
