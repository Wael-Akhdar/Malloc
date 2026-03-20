# Malloc - Custom Dynamic Memory Allocator

> **Privacy Note:** The source code of this project is kept private to comply with EPITA's anti-plagiarism regulations. The code is structured, commented, and available upon request for recruiters during an interview. This repository serves as a technical showcase and provides a testable compiled version.

## 📖 Project Context

The **Malloc** project is an EPITA system programming assignment focused on understanding virtual memory management. The objective is to build a custom shared library (`libmalloc.so`) that replaces the standard C library (libc) memory allocation functions (`malloc`, `free`, `calloc`, `realloc`). This requires interacting directly with the kernel to request memory pages and efficiently sub-dividing them for user applications.

## ✨ Implemented Features

The allocator provides a fully functional drop-in replacement for standard memory management functions:

* **`malloc`:** Allocates memory by requesting pages directly from the OS via `mmap`. It handles small allocations using a specialized bucket system and maps large allocations directly.
* **`free`:** Releases memory back to the allocator. If an entire page becomes empty, it is aggressively returned to the operating system using `munmap` to minimize the memory footprint.
* **`calloc`:** Allocates and zero-initializes memory arrays, including safe integer overflow detection using `__builtin_mul_overflow` to prevent security vulnerabilities.
* **`realloc`:** Resizes previously allocated memory blocks, reusing the existing block if sufficient space is available, or seamlessly migrating data to a new block.
* **Shared Library Support:** All core functions use the `__attribute__((visibility("default")))` directive to ensure they can be properly exported and intercepted by dynamic linkers.

## 🏗️ Technical Architecture

The allocator is built upon a **Segregated Fits / Bucket Allocator** design:

1.  **Bucket Segregation:** Memory requests are rounded up to the nearest power of 2. The allocator maintains different "buckets," where each bucket manages a full memory page (`sysconf(_SC_PAGESIZE)`) subdivided into fixed-size blocks.
2.  **Intrusive Free Lists:** To minimize metadata overhead, free blocks do not use external tracking structures. Instead, the available memory space *inside* the freed block is cast into a `struct free_list` to store a pointer to the next available block.
3.  **O(1) Metadata Retrieval:** The system avoids storing headers directly before allocated user pointers. Instead, it locates block metadata by masking the pointer address down to the nearest page boundary (`s_addr - (s_addr % page_len)`), ensuring fast and secure metadata lookups.
4.  **Large Block Handling:** Allocations exceeding the standard page capacity bypass the bucket system entirely. They are flagged as `is_bigSmr` and mapped/unmapped directly with their exact required length.

## 🧠 Technical Challenges Solved

* **Fragmentation Mitigation:** By grouping allocations of identical sizes into specific buckets, external fragmentation is heavily reduced.
* **Minimizing System Calls:** `mmap` is notoriously slow if called for every small allocation. The bucket system requests entire pages at once, serving dozens or hundreds of `malloc` calls from a single system call.
* **Alignment Constraints:** Padding mathematics are implemented to ensure all structures and returned pointers are strictly aligned to hardware requirements (16-byte boundaries) to prevent segmentation faults on modern architectures.

## 🎯 How to test the project?

A pre-compiled shared library is available for testing.

1.  Go to the **[Releases](../../releases)** section.
2.  Download the archive containing the `libmalloc.so` shared object.
3.  Open a terminal in a Linux environment and use the `LD_PRELOAD` environment variable to force standard UNIX commands to use this custom allocator instead of the system's default one:

```bash
# Example: Injecting the allocator into standard commands
LD_PRELOAD=./libmalloc.so ls -la
LD_PRELOAD=./libmalloc.so tree
LD_PRELOAD=./libmalloc.so cat some_file.txt
