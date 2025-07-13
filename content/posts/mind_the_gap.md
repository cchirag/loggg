---
title: "Mind the Gap: Understanding Go Struct Padding"
date: 2025-07-12T00:50:00+05:30
draft: false
description: "Thought your struct was tightly packed? Go might sneak in extra bytes when you're not looking. This post breaks down how struct layout and padding work, why it matters when working close to memory, and how I ran into this while building Mint"
tags: ["mint", "database", "storage", "OS", "Go"]
cover:
  image: "images/mind_the_gap.png"
  alt: "Visual representation of Go struct memory layout with gaps indicating padding between fields."
  relative: true
  hidden: false
---

With [page layouts](https://www.loggg.xyz/posts/pages-in-mint/) and an understanding of [page descriptors and `mmap`](https://www.loggg.xyz/posts/file-descriptors-and-mmap/) in place, I was ready to define the page structure for Mint. Like most things, it started with some basic struct definitions, methods, and interfaces. Everything seemed to be going smoothly.

The next step was to map the byte slice returned by `mmap` into something more meaningful. Since `mmap` reflects the file content as-is, working with raw bytes directly can be tedious and error-prone, especially as the structure grows. Even with a formal layout in mind, dealing with offsets and manual encoding felt clunky.

So I decided to represent the page header as a Go struct.

The header is a 64-byte block that stores metadata about a page. Here's how I initially defined it:

```go
type Header struct {
	HeaderVersion uint16      // 2 bytes
	PageType      uint16      // 2 bytes
	PageID        uint64      // 8 bytes
	NextPageID    uint64      // 8 bytes
	PrevPageID    uint64      // 8 bytes
	Checksum      uint64      // 8 bytes
	Reserved      [28]byte    // 28 bytes
}
```

This added up to 64 bytes on paper, which matched the intended layout. But let’s verify the actual offsets:

| Field Name    | Field Type | Size (Bytes) | Offset |
| ------------- | ---------- | ------------ | ------ |
| HeaderVersion | `uint16`   | 2            | 0      |
| PageType      | `uint16`   | 2            | 2      |
| PageID        | `uint64`   | 8            | 4      |
| NextPageID    | `uint64`   | 8            | 12     |
| PrevPageID    | `uint64`   | 8            | 20     |
| Checksum      | `uint64`   | 8            | 28     |
| Reserved      | `[28]byte` | 28           | 36     |

So far, it looked fine. But when I ran `unsafe.Sizeof(Header{})`, the result came out to be **72** bytes instead of the expected 64.

### Where Did the Extra 8 Bytes Come From?

The answer is padding.

### What Is Padding?

Padding refers to invisible bytes that the Go compiler inserts between struct fields to ensure proper alignment in memory. These alignments aren't just a Go thing. Most CPUs prefer (and sometimes require) that certain data types begin at memory addresses that are divisible by their size. This allows for faster access and better hardware compatibility.

Let’s take a look at how different types are aligned:

| Type          | Size (Bytes) | Aligned Offsets (Examples) | Alignment Rule |
| ------------- | ------------ | -------------------------- | -------------- |
| `uint8`       | 1            | 0, 1, 2, 3, ...            | every byte     |
| `uint16`      | 2            | 0, 2, 4, 6, ...            | multiple of 2  |
| `uint32`      | 4            | 0, 4, 8, 12, ...           | multiple of 4  |
| `uint64`      | 8            | 0, 8, 16, 24, ...          | multiple of 8  |
| `[]T`         | 24           | 0, 24, 48, 72, ...         | multiple of 8  |
| `interface{}` | 16           | 0, 16, 32, 48, ...         | multiple of 8  |

To satisfy these alignment rules, the Go compiler adds padding between fields where needed. In my case, the 72-byte result came from a few extra bytes that were automatically inserted by the compiler.

Here’s how the actual memory layout looked:

#### Original Layout (`72 bytes`)

| Field         | Type       | Size | Offset | Notes                                      |
| ------------- | ---------- | ---- | ------ | ------------------------------------------ |
| HeaderVersion | `uint16`   | 2    | 0      |                                            |
| PageType      | `uint16`   | 2    | 2      |                                            |
| *padding*     | —          | 4    | 4      | Added to align `uint64` on 8-byte boundary |
| PageID        | `uint64`   | 8    | 8      |                                            |
| NextPageID    | `uint64`   | 8    | 16     |                                            |
| PrevPageID    | `uint64`   | 8    | 24     |                                            |
| Checksum      | `uint64`   | 8    | 32     |                                            |
| Reserved      | `[28]byte` | 28   | 40     |                                            |
| *padding*     | —          | 4    | 68     | Aligns total size to multiple of 8         |

📏 **Total size: 72 bytes**

---

### Reducing or Eliminating Padding

Padding exists for a good reason, but that doesn’t mean we always have to live with it. There are a couple of ways to optimize our struct for space without giving up performance.

---

#### 1. Rearranging Fields

The easiest solution is to place larger fields first, followed by smaller ones. This gives the compiler a better chance of aligning things naturally without needing to insert padding.

Here’s the rearranged version of the struct:

```go
type Header struct {
	PageID        uint64
	NextPageID    uint64
	PrevPageID    uint64
	Checksum      uint64
	HeaderVersion uint16
	PageType      uint16
	Reserved      [28]byte
}
```

Now when I run `unsafe.Sizeof(Header{})`, I get **64** bytes, exactly what I wanted.

#### Rearranged Layout (`64 bytes`)

| Field         | Type       | Size | Offset |
| ------------- | ---------- | ---- | ------ |
| PageID        | `uint64`   | 8    | 0      |
| NextPageID    | `uint64`   | 8    | 8      |
| PrevPageID    | `uint64`   | 8    | 16     |
| Checksum      | `uint64`   | 8    | 24     |
| HeaderVersion | `uint16`   | 2    | 32     |
| PageType      | `uint16`   | 2    | 34     |
| Reserved      | `[28]byte` | 28   | 36     |

📏 **Total size: 64 bytes**

---

#### 2. Manual Encoding with `encoding/binary`

For full control over byte layout, especially when writing to disk, we can skip structs entirely and manually encode each field using the `encoding/binary` package.

```go
buf := make([]byte, 64)
binary.LittleEndian.PutUint16(buf[0:], 1)          // HeaderVersion
binary.LittleEndian.PutUint16(buf[2:], 2)          // PageType
binary.LittleEndian.PutUint64(buf[4:], 42)         // PageID
// ... and so on
```

This method avoids padding entirely and offers platform-independent encoding, but comes at the cost of convenience and type safety.

---

### Final Thoughts

Struct layout, padding, and alignment in Go aren't always obvious at first glance. It’s easy to assume structs will be packed exactly the way we write them, but the compiler has its own alignment rules that we need to be aware of.

For Mint, I chose to stick with the **rearranged struct** approach. It keeps things type-safe, easy to understand, and aligned with my goals of predictability and performance.

You can try this out yourself in the Go Playground:
👉 [https://go.dev/play/p/Blqm-jO4OT9](https://go.dev/play/p/Blqm-jO4OT9)
