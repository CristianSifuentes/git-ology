# Advanced Explanation of Git’s Internal Mechanisms: SHA-1 Hashing and Object Storage, Advanced Technical Analysis of Git’s Internal Structure and the `hash-object` Command

## Table of Contents

1. [Git as a Content Manager](#git-as-a-content-manager)
   - [What is Content Addressability?](#what-is-content-addressability)
2. [How Git Uses SHA-1 Hashing](#how-git-uses-sha-1-hashing)
   - [Example of SHA-1 Hash in Git](#example-of-sha-1-hash-in-git)
   - [Why SHA-1?](#why-sha-1)
3. [The Git Object Model](#the-git-object-model)
   - [Four Types of Objects in Git](#four-types-of-objects-in-git)
   - [Example: Storing a Blob in Git](#example-storing-a-blob-in-git)
4. [How Git Ensures Data Integrity](#how-git-ensures-data-integrity)
   - [Mechanisms Ensuring Data Integrity](#mechanisms-ensuring-data-integrity)
   - [Example: Checking an Object's Type](#example-checking-an-objects-type)
5. [Git’s Plumbing vs. Porcelain Commands](#gits-plumbing-vs-porcelain-commands)
   - [Plumbing Commands (Low-Level)](#plumbing-commands-low-level)
   - [Porcelain Commands (High-Level)](#porcelain-commands-high-level)
6. [How Git Uses SHA-1 for Commit History](#how-git-uses-sha-1-for-commit-history)
7. [SHA-1 Collision Resistance in Git](#sha-1-collision-resistance-in-git)
8. [Summary: Why Git is a Content Manager](#summary-why-git-is-a-content-manager)
9. [git hash-object: A Low-Level Command](#git-hash-object-a-low-level-command)
    - [Understanding `git hash-object`](#understanding-git-hash-object)
    - [Example 1: Running `hash-object` Outside a Git Repository](#example-1-running-hash-object-outside-a-git-repository)
    - [Example 2: Running `hash-object` Inside a Git Repository](#example-2-running-hash-object-inside-a-git-repository)
10. [Understanding the `.git` Directory Structure](#understanding-the-git-directory-structure)
    - [Tree Structure of a Freshly Initialized Git Repository](#tree-structure-of-a-freshly-initialized-git-repository)
    - [Detailed Breakdown of `.git` Subdirectories](#detailed-breakdown-of-git-subdirectories)
11. [The Importance of the `objects/` Directory](#the-importance-of-the-objects-directory)
    - [Structure of `.git/objects/`](#structure-of-gitobjects)
    - [Understanding the `.git/objects` Directory](#understanding-the-gitobjects-directory)
       - [How Git Stores Objects Efficiently](#how-git-stores-objects-efficiently)
       - [Why This Structure?](#why-this-structure)
    - [Types of Objects Stored in `objects/`](#types-of-objects-stored-in-objects)
    - [Example: Storing and Retrieving a File Manually](#example-storing-and-retrieving-a-file-manually)
    - [Retrieving Stored Objects (`cat-file`)](#retrieving-stored-objects-cat-file)
       - [Key Insights on Shortened Hashes](#key-insights-on-shortened-hashes)
12. [Advanced Insights: What Happens Internally?](#advanced-insights-what-happens-internally)
    - [Compression and Storage](#compression-and-storage)
    - [Object Format in Git](#object-format-in-git)
13. [Visualizing Git's Object Storage](#visualizing-gits-object-storage)   
14. [Advanced Git Internals: Reconstructing a Real Git Scenery](#advanced-git-internals-reconstructing-a-real-git-scenery)
    - [Step 1: Retrieving a File Manually (Plumbing Commands)](#step-1-retrieving-a-file-manually-plumbing-commands)
    - [Step 2: Write a Blob (File Content)](#step-2-write-a-blob-file-content) 
    - [Step 3: Understanding the First Commit in Git ](#step-3-understanding-the-first-commit-in-git) 
    <!-- - [Step 2: Write a Tree Object](#step-2-write-a-tree-object)
    - [Step 3: Create a Commit](#step-3-create-a-commit)
    - [Step 4: Move `HEAD` to the Commit](#step-4-move-head-to-the-commit)
    - [Retrieving and Inspecting Each Object](#retrieving-and-inspecting-each-object)
    - [(1) Checking the Commit Object](#1-checking-the-commit-object)
    - [(2) Examining the Tree Object](#2-examining-the-tree-object)
    - [(3) Verifying the Blob Object](#3-verifying-the-blob-object) -->
16. [Advanced Commands for Exploring Git Internals](#advanced-commands-for-exploring-git-internals) 
    - [Inspecting Git Tree and Commit Information](#inspecting-the-git-tree-git-ls-tree-head)
       - [Inspecting the Git Tree: `git ls-tree HEAD`](#inspecting-the-git-tree-git-ls-tree-head)
            - [Understanding `git ls-tree`](#understanding-git-ls-tree)
            - [Example Usage](#example-usage)
            - [Interpreting the Output](#interpreting-the-output)
       - [Retrieving Commit Hash: `git rev-parse HEAD`](#retrieving-commit-hash-git-rev-parse-head)
            - [Understanding `git rev-parse`](#understanding-git-rev-parse)
            - [Example Usage](#example-usage-1)
       - [Verifying Repository Integrity: `git fsck --full`](#verifying-repository-integrity-git-fsck--full)
            - [Understanding `git fsck`](#understanding-git-fsck)
            - [Example Usage](#example-usage-2)
       - [Conclusion](#conclusion)
    - [Viewing Commit History](#viewing-commit-history)
       - [Basic Commit Log](#basic-commit-log)
       - [Viewing Patch Details](#viewing-patch-details)
       - [Displaying File-Level Changes](#displaying-file-level-changes)
    - [Customizing Git Log Output](#customizing-git-log-output)
       - [Single-Line Commit Format](#single-line-commit-format)
       - [Custom Commit Formatting](#custom-commit-formatting)
       - [Visualizing Commit Graphs](#visualizing-commit-graphs)
    - [Filtering Commit History](#filtering-commit-history)
       - [Filtering by Date](#filtering-by-date)
       - [Searching for Specific Changes](#searching-for-specific-changes)
       - [Filtering by File Path](#filtering-by-file-path)
    - [Advanced Filtering & Searching](#advanced-filtering--searching)
       - [Filtering by Author and Date](#filtering-by-author-and-date)
       - [Excluding Merge Commits](#excluding-merge-commits)
    - [Summary](#summary)   
17. [Key Takeaways for Software Engineers](#key-takeaways-for-software-engineers)

---

## Git as a Content Manager
Git is fundamentally a **content-addressable file system**. Rather than tracking file names, That means Git stores and retrieves content using a key-value data structure, Git identifies content using a **SHA-1 hash**, making it efficient, immutable, and storage-friendly.

### What is Content Addressability?
- Content-addressability means that **data is referenced by its hash value rather than by filename**.
- Every object in Git  (blob, tree, commit, tag) is stored under a **SHA-1 hash**, making it immutable, ensuring its uniqueness and preventing redundancy.
- This approach ensures **integrity**: identical content will always have the same hash.

---

## How Git Uses SHA-1 Hashing
Git assigns a **SHA-1 (Secure Hash Algorithm 1) hash** to every object it stores, making it a **distributed and cryptographically secure system**. A SHA-1 hash is a 40-character hexadecimal string, representing a 160-bit digest of input data.

### Example of SHA-1 Hash in Git
```bash
echo "Hello World" | git hash-object --stdin
```
Output:
```nginx
557db03de997c86a4a028e1ebd3a1ceb225be238
```
- The `git hash-object` command computes the SHA-1 hash of "Hello World", ensuring idempotency—this command will always produce the same hash for the same input.
- This hash is the identifier that Git will use to store the object. If the content is modified, **a completely different SHA-1 hash will be generated**.

### Why SHA-1?
- **Ensures integrity**: Any alteration changes the hash.
- **Efficient deduplication**: Identical files are stored once, saving space.
- **Fast lookup**: SHA-1 allows Git to quickly retrieve content.

---

## The Git Object Model
Git stores content as objects in the `.git/objects/` directory. These objects are compressed, stored, and referenced via SHA-1 hashes. Git stores content in **four object types**, forming the foundation of its version control system.

### Four Types of Objects in Git
| Object Type | Description |
|-------------|-------------|
| **Blob**  | Stores raw file contents  (not filenames or metadata). |
| **Tree**  | Represents a directory structure (links blobs and trees). |
| **Commit** | Points to a tree and includes metadata (author, message, parent commit, etc.). |
| **Tag**    | Stores referenceable tags for commits (annotated tags). |

### Example: Storing a Blob in Git
```bash
echo "Hello World" | git hash-object -w --stdin
```
Output:
```nginx
557db03de997c86a4a028e1ebd3a1ceb225be238
```
- This command **stores** "Hello World" inside `.git/objects/` and assigns it a **SHA-1 hash**. The `-w` flag writes the object into Git’s object database.
- The blob **does not track filenames**, only raw content. The object is now stored inside `.git/objects/55/7db03de997c86a4a028e1ebd3a1ceb225be238`.

### To retrieve the content, use:

```bash
git cat-file -p 557db03de997c86a4a028e1ebd3a1ceb225be238
```

### Output:

```nginx
Hello World
```

---

## How Git Ensures Data Integrity
Git ensures data integrity using SHA-1 hashing, immutability, object compression, and content deduplication.

### Mechanisms Ensuring Data Integrity
- **SHA-1 hashing** guarantees uniqueness.
  - Every object in Git is identified by a 160-bit SHA-1 hash.
  - Any modification to an object results in a completely new hash.
  - This prevents accidental overwrites or corruption.
- **Delta compression** reduces redundancy.
  - Git compresses objects and stores differences (deltas) rather than full copies.
  - `git gc` (garbage collection) optimizes this process for large repositories.
- **Git’s immutable data model** prevents accidental corruption.
- **Verifying Repository Integrity** prevents accidental corruption.
  - Detects missing objects or corruption.
    ```bash
    git fsck --full
    ```

### Example: Checking an Object's Type
```bash
git cat-file -t <hash>
```
### Output:

```nginx
blob
```

---

## Git’s Plumbing vs. Porcelain Commands
Git provides **two levels of commands**:

### Plumbing Commands (Low-Level)
| Command | Description |
|---------|-------------|
| `git hash-object` | Computes and writes SHA-1 hash for content. |
| `git cat-file` | Displays object content by SHA-1 hash. |
| `git rev-parse` | Retrieves commit hashes and refs. |
| `git update-index` | Stages files manually into Git’s index. |
| `git write-tree` | Creates a tree object from the index. |

### Porcelain Commands (High-Level)
| Command | Description |
|---------|-------------|
| `git commit` | Creates a commit object with metadata. |
| `git add` | Stages files. |
| `git status` | Displays working directory status. |
| `git checkout` | Switches branches or restores files. |

---

## How Git Uses SHA-1 for Commit History
A **Git commit** is essentially a SHA-1 hash that represents the project state at a specific time. The commit includes a tree (file structure), parent commit(s), and metadata.

Example:
```bash
git log --pretty=raw
```
Output:
```
commit 9fceb02b21337d3025f69e22f68c82d20a000000
tree 36b74b3b8f6a...
parent cf23df2207d9...
author John Doe <johndoe@example.com>
committer John Doe <johndoe@example.com>
```
- **Commit SHA-1**: Identifies the commit.
- **Tree SHA-1**: Represents the directory structure.
- **Parent SHA-1**: Tracks commit history.

---

## SHA-1 Collision Resistance in Git
- **Git uses additional security mechanisms** to detect hash collisions.
- Git has moved towards **SHA-256 support** to further enhance security. Git now supports SHA-256 hashes (`git init --object-format=sha256`).
- `git fsck` ensures object integrity.
-  **Git content structure**: Even if a SHA-1 collision were found, Git’s reliance on tree structures and commit history prevents serious exploits.

---

## Summary: Why Git is a Content Manager
- **Git stores and retrieves data based on SHA-1 hashes**,  acting as a key-value content store.
- **Content-addressable storage** ensures deduplication and efficiency.
- **Plumbing commands** allow direct interaction with the object storage.
- **Git’s object model (blobs, trees, commits, tags) enables version tracking.**
- **Commit history is a chain of hashes**, creating an immutable ledger of project changes.


---

## git hash-object: A Low-Level Command

### Understanding `git hash-object`
Git's `hash-object` command computes a **SHA-1 hash** for given content, allowing it to be stored in Git’s object database. It is a **plumbing command**, meaning it operates at a low level and does not track files.

### Key Properties:
- **Can be used outside of a Git repository.**
- **Idempotent:** Given the same input, it will always return the same SHA-1 hash.
- Supports **writing data to Git’s object store** (`-w` flag).
- Stores objects in compressed format (`zlib`).


### Example 1: Running `hash-object` Outside a Git Repository
```bash
echo "Hello Git" | git hash-object --stdin
```
Output:
```nginx
8cf2d8a03c123f8824ac46aa20a6b924ad44f0c8
```
- This command generates a **SHA-1 hash** but does **not store** it.

### Example 2: Running `hash-object` Inside a Git Repository

- **Initialize a Git repository**

    ```bash
    mkdir my-repo && cd my-repo
    git init
    ```
- **Generate and store an object in `.git/objects`**:

    ```bash
    echo "Hello Git" | git hash-object -w --stdin
    ```
    Output:
    ```nginx
    8cf2d8a03c123f8824ac46aa20a6b924ad44f0c8
    ```
- **Locate the stored object:**

    ```bash
    ls .git/objects/e6/
    ```
    Output:
    ```nginx
    9de29bb2d1d6434b8b29ae775ad8c2e48c5391
    ```

- **This command **stores the content** in `.git/objects/` and assigns a SHA-1 hash.**

---

## Understanding the `.git` Directory Structure

When a Git repository is initialized (git init), a .git directory is created to manage version history and metadata.

### Tree Structure of a Freshly Initialized Git Repository
```
.git
├── HEAD
├── branches
├── config
├── description
├── index
├── hooks/
│   ├── pre-commit.sample
│   ├── pre-push.sample
│   ├── post-update.sample
│   └── (other hook samples)
├── info/
│   ├── exclude
├── objects/
│   ├── info/
│   ├── pack/
├── refs/
│   ├── heads/
│   ├── tags/
└── logs/
```

### Detailed Breakdown of `.git` Subdirectories
| Directory | Purpose |
|-----------|---------|
| `HEAD` | Stores reference to the current branch. |
| `branches/` | (Legacy) Used for alternative branch management. |	
| `config` | Stores repository-level Git settings. |
| `description` | Used for descriptions in GitWeb interfaces (rarely used). |
| `hooks/` | Contains scripts for automation (pre-commit, post-push, etc.). |
| `info/` | Contains exclude, which defines ignored files (similar to .gitignore). |
| `objects/` | Contains all Git objects (blobs, trees, commits, tags). |
| `refs/` | Stores branch (`heads/`) and tag (`tags/`) references. |
---

## The Importance of the `objects/` Directory

The `objects/` folder is the heart of Git’s key-value store, containing all historical content as hashed objects.

### Structure of `.git/objects/`
```
.git/objects
├── info
├── pack
└── (hashed objects, e.g., 2e/a1f2b3c4d5e6…)
```

## Understanding the `.git/objects` Directory

### How Git Stores Objects Efficiently
- The SHA-1 hash is split into:
  - **First two characters** → Directory name `(55)`.
  - **Remaining 38 characters** → File name `(7db03de997c86a4a028e1ebd3a1ceb225be238)`.
- This prevents **large numbers of files in a single directory**.

### Why This Structure?
  - **Speeds up lookups:** By reducing the number of files per directory.
  - **Ensures unique content:** The same content always produces the same hash, preventing duplicate storage.
  - **Supports distributed versioning:** Each object is self-contained and immutable.

### Types of Objects Stored in `objects/`
Git stores four object types, all identified by SHA-1 hashes:


| Object Type | Description |
|-------------|------------|
| **Blob** | Stores file contents without metadata (not filenames). |
| **Tree** | Represents a directory structure (contains pointers to blobs and trees). |
| **Commit** | Points to a tree object and includes metadata (author, message, parent commit, etc.). |
| **Tag** | Stores tag data and references commits (annotated tags).|

### Example: Storing and Retrieving a File Manually
```bash
echo "Hello Git Internals!" | git hash-object -w --stdin
```
Output:
```nginx
8cf2d8a03c123f8824ac46aa20a6b924ad44f0c8
```
Now, retrieve the content:

```bash
git cat-file -p <hash>
```
Output:
```nginx
Hello, Git Internals!
```
---
## Retrieving Stored Objects (`cat-file`)
Since the object is now in  `.git/objects/`, we can inspect its content using:

```bash
git cat-file -p 557db03de997c86a4a028e1ebd3a1ceb225be238
```
Output:
```nginx
Hello World
```
Git allows us to reference objects with shortened hashes, provided they are unique in the repository:

```bash
git cat-file -p 557db03
```
Output:
```nginx
Hello World
```

### Key Insights on Shortened Hashes
- Git allows **shortened hashes** as long as they are unique.
- Minimum **6–7 characters** are usually enough.

---

## Advanced Insights: What Happens Internally?
### Compression and Storage
- Objects are **compressed** using **Zlib** before storage.
- Git **stores only differences (deltas)** when optimizing storage.
- Running the following command retrieves **raw binary data:**
    ```bash
    zcat .git/objects/55/7db03de997c86a4a028e1ebd3a1ceb225be238
    ```
    * This is why `.git/objects` files **cannot be read directly** in a text editor.


### Object Format in Git
A stored object follows the format:
```
<type> <size>\0<content>
```
For `"Hello World"`:
```
blob 11\0Hello World
```


* `blob` → Object type.
* `11` → Number of bytes in `"Hello World"`.
* `\0` → Null separator before content.

---

## Visualizing Git's Object Storage
Using **`ls`**, we can see how objects are structured:
```bash
ls -R .git/objects/
```
Example output:
```
.git
├── HEAD  (Pointer to the current branch)
├── objects
│   ├── 55
│   │   └── 7db03de997c86a4a028e1ebd3a1ceb225be238 (Stored Blob)
│   ├── info
│   └── pack
├── refs
│   ├── heads
│   └── tags

```

---

## Advanced Git Internals: Reconstructing a Real Git Scenery

### Step 1: Retrieving a File Manually (Plumbing Commands)

```bash
echo "Hello World" | git hash-object --stdin
```
* **Output:**
   ```nginx
   2b929a919511b881a7643785c73a74c8e59db018
   ```

* **Checking an Object's Type**
   ```bash
   git cat-file -t 2b929a919511b881a7643785c73a74c8e59db018
   ```
* **Output:**
   ```nginx
   blob
   ```

* **Now, retrieve the content:**
   ```bash
   git cat-file -p 2b929a919511b881a7643785c73a74c8e59db018
   ```

* **Output:**
   ```nginx
   Hello World
   ```



### Step 2: Write a Blob (File Content)

```bash
 echo "Hello World" | git hash-object -w --stdin
```

* **Output:**
   ```nginx
   56172c3d5042ca5ec65c34b5d1e91093ecee5648
   ```

* **Checking an Object's Type**
   ```bash
   git cat-file -t 56172c3d5042ca5ec65c34b5d1e91093ecee5648
   ```
* **Output:**
   ```nginx
   blob
   ```

* **Now, retrieve the content:**
   ```bash
   git cat-file -p 56172c3d5042ca5ec65c34b5d1e91093ecee5648
   ```

* **Output:**
   ```nginx
    Hello World
   ```




## Step 3: Understanding the First Commit in Git
When creating a new Git repository and making the first commit, Git initializes its **object database** with a structured reference to **blobs (files), trees (directories), and commits (history).**

```bash
echo "Hello World" > hello.txt

git add .

git commit -m "First commit"

[master (root-commit) c7a78d3] First commit
 1 file changed, 1 insertion(+)
 create mode 100644 hello.txt

```

Output:
```nginx
 56172c3d5042ca5ec65c34b5d1e91093ecee5648
```

After committing, Git creates multiple objects inside `.git/objects/`, structured as follows:

```
.git/objects
├── 2b
│   └── 929a919511b881a7643785c73a74c8e59db018  (Blob for "Hello World")
├── 56
│   └── 172c3d5042ca5ec65c34b5d1e91093ecee5648  (Blob for "Hello World")
├── 8b
│   └── 8483151bd62dcbf7de98ce2c56afcca33ac816  (Tree containing hello.txt)
├── c7
│   └── a78d35d46ef5bca39ac78e6b1661189a34ebd1  (Commit object)
```
### Step 6: Branches
### Step 7: Merge
### Step 8: Recursive
### Step 9: Rebase
### Step 10: Conflict resolution

Synchronizing with the remote repository

Scenario 1

Scenario 2

Scenario 3: the same, but with conflicts

Conclusions


---

### Step 2: Write a Tree Object
```bash
git update-index --add --cacheinfo 100644 557db03de997c86a4a028e1ebd3a1ceb225be238 hello.txt
git write-tree

```
Output:
```nginx
 9a3e5b4d8a6c7f12e6aef3c1b2e5c4d1a9a8c7e6
```

* The tree object represents a directory structure and references blob objects.

### Step 3: Create a Commit
```bash
 echo "Initial commit" | git commit-tree 9a3e5b4d8a6c7f12e6aef3c1b2e5c4d1a9a8c7e6 -m "Initial commit"
```
Output:
```nginx
 d2f3a5e4b6c2a1e7d8c9b0e2f3a7c6d5e4b3a2e1
```
* This manually creates a commit object!


### Step 4: Move `HEAD` to the Commit
```bash
git branch -f main d1ee121
git checkout main
```
* Now the repository **has a valid commit history!**

---




### Step 4: Tags
### Step 5: Create a Second commit

## Retrieving and Inspecting Each Object

### (1) Checking the Commit Object
```bash
git cat-file -t d1ee121
```
Output:
```
commit
```

```bash
git cat-file -p d1ee121
```
Output:
```
tree 97b49d4c943e3715fe30f141cc6f27a8548cee0e
author John Doe <johndoe@example.com> 1485111055 +0000
committer John Doe <johndoe@example.com> 1485111055 +0000

First commit
```

### (2) Examining the Tree Object
```bash
git cat-file -t 97b49d4
```
Output:
```
tree
```

```bash
git cat-file -p 97b49d4
```
Output:
```
100644 blob 557db03de997c86a4a028e1ebd3a1ceb225be238 hello.txt
```

<!-- ### (3) Verifying the Blob Object
```bash
git cat-file -t 557db03
```
Output:
```
blob
```

```bash
git cat-file -p 557db03
```
Output:
```
Hello World
``` -->

---

## Inspecting the Git Tree: `git ls-tree HEAD`

### **Understanding `git ls-tree`**
- `git ls-tree` displays **the contents of a tree object**.
- `HEAD` refers to the **current commit**, so `git ls-tree HEAD` lists files in the latest commit.
- The output includes:
  - **File mode** (permissions).
  - **Type** (`blob` for files, `tree` for directories).
  - **SHA-1 hash** of each object.
  - **File or directory name**.

### **Example Usage**
```bash
git ls-tree HEAD
```
### **Interpreting the Output**
Example output:

```bash
100644 blob a4c2e3e file1.txt
100644 blob b6f9d5f file2.txt
040000 tree 3f5e2a7 src

```
## Retrieving Commit Hash: `git rev-parse HEAD`

**Understanding** `git rev-parse`
* `git rev-parse HEAD` returns the **full commit hash** of the latest commit.
* Useful for **scripting, automation, and debugging.**

**Example Usage**
```bash
git rev-parse HEAD
```

**Example output:**

```bash
9b2e3d7a8f00c6e4f88d70a9c2e7fcbf97e6c9c5
```

This hash uniquely identifies the **latest commit** in the current branch.



## Verifying Repository Integrity: `git fsck --full`

**Understanding** `git fsck`
* `git fsck` checks the **validity of Git objects.**
* `--full` performs a **deep integrity check.**
* Identifies **corrupt objects, missing references, or structural issues.**

**Example Usage**

```bash
git fsck --full
```


**Example output:**

```bash
Checking object directories: 100% (256/256), done.
Checking objects: 100% (1500/1500), done.
```

If corruption exists:

```vbnet
error: missing blob 8f00c6e4
fatal: object 8f00c6e4 missing
```

* This indicates that an object is missing and might need **recovery from a remote repository.**


## Conclusion

These **low-level Git commands** are essential for **inspecting, debugging, and verifying repositories:**

| Command | Description |
|---------|-------------|
| `git ls-tree HEAD` | Lists files and directories at the latest commit. |
| `git rev-parse HEAD` | Retrieves the latest commit hash. |
| `git fsck --full` | 	Checks repository integrity and detects corruption.|

## Advanced Commands for Exploring Git Internals

### Viewing Commit History

#### Basic Commit Log
```bash
git log
```
* Displays commit history **with details such as author, date, and commit message.**

* **Default output:**
  ```sql
   commit 9b2e3d7a8f00c6e4f88d70a9c2e7fcbf97e6c9c5
   Author: John Doe <johndoe@example.com>
   Date:   Mon Mar 4 10:15:32 2024 +0000

      Fixed a critical bug in authentication

  ```

#### Viewing Patch Details
```bash
git log -p -2
```
* Shows **patch details** (actual code changes) for the last **two** commits.

* **Default output:**
  ```sql
   commit 4a7e8d9c6a...
   Author: Alice Smith
   Date:   Tue Mar 5 14:20:01 2024

      Refactored user authentication

   diff --git a/hello.txt b/hello.txt
   --- a/hello.txt
   +++ b/hello.txt
   @@ -34,7 +34,7 @@
   -    return False
   +    return True

  ```


#### Displaying File-Level Changes
```bash
git log --stat
```
* Displays commit history **with file-level change statistics.**

* **Default output:**
  ```sql
   commit 9b2e3d7a8f00c6e4f88d70a9c2e7fcbf97e6c9c5
   Author: John Doe <johndoe@example.com>
   Date:   Mon Mar 4 10:15:32 2024 +0000

      Fixed login bug

   hello.txt | 3 +--
   hello2.txt | 2 ++
   2 files changed, 3 insertions(+), 2 deletions(-)

  ```

### Customizing Git Log Output

#### Single-Line Commit Format
```bash
git log --pretty=oneline
```
* Displays commit history **in a compact single-line format**.

* **Example output:**
  ```pgsql
   9b2e3d7a Fixed login bug
   4a7e8d9c Refactored authentication
   8f00c6e4 Added password encryption
  ```


#### Custom Commit Formatting
```bash
git log --pretty=format:"%h - %an, %ar : %s"
```
- `%h` → Short commit hash
- `%an` → Author name
- `%ar` → Relative commit date
- `%s` → Commit message

* **Example output:**
  ```yaml
   9b2e3d7 - John Doe, 3 days ago : Fixed login bug
   4a7e8d9 - Alice Smith, 1 week ago : Refactored authentication
  ```

#### Visualizing Commit Graphs
```bash
git log --pretty=format:"%h %s" --graph
```
* Displays commit history **as a graph** (branching visualization).

* **Example output:**
  ```markdown
   * 9b2e3d7 Fixed login bug
   * 4a7e8d9 Refactored authentication
   * 8f00c6e Added password encryption
  ```

---

### Filtering Commit History

#### Filtering by Date
```bash
git log --since=2.weeks
```
* Shows commits from the last **two weeks**.

* **Example usage:**
  ```markdown
  git log --since="2024-02-20"
  ```
* Useful for **reviewing recent changes.**


#### Searching for Specific Changes
```bash
git log -S function_name
```
* Finds commits where a **specific function or string was added or removed**.

* **Example:**
  ```bash
   git log -S "validate_login"
  ```
* Useful for **tracking down when a function was introduced or modified.**


#### Filtering by File Path
```bash
git log -- path/to/file
```
* Shows commit history **for a specific file.**
* **Example:**
  ```bash
   git log -- app/models/user.py
  ```
* Useful for **tracking changes to a particular file.**

---

### Advanced Filtering & Searching

#### Filtering by Author and Date
```bash
git log --pretty="%h - %s" --author='Junio C Hamano' --since="2008-10-01" \
   --before="2008-11-01" --no-merges -- t/
```
- Filters commits **by author, date range, and file path**.
- `--no-merges` excludes merge commits.

* Explanation:
   * `--pretty="%h - %s"` → Show hash and commit message.
   * `--author='Junio C Hamano` → Filter by author.
   * `--since="2008-10-01"` → Commits after October 1, 2008.
   * `--before="2008-11-01"` → Commits before November 1, 2008.
   * `--no-merges` → Exclude merge commits.
   * `--t/` → Limit search to files in the t/ directory.

* **Example output:**
  ```bash
   4a7e8d9 - Improved test coverage
   8f00c6e - Fixed security vulnerability
  ```


#### Excluding Merge Commits
```bash
git log --no-merges
```
* Lists commits **without showing merge commits**.

---

### Summary

| **Command** | **Description** |
|-------------|---------------|
| `git log` | Shows commit history. |
| `git log -p -2` | Displays code changes for last 2 commits. |
| `git log --stat` | Shows file-level changes per commit. |
| `git log --pretty=oneline` | Displays commits in single-line format. |
| `git log --pretty=format:"%h - %an, %ar : %s"` | Custom commit format with hash, author, date, and message. |
| `git log --pretty=format:"%h %s" --graph` | Displays a visual commit graph. |
| `git log --since=2.weeks` | Shows commits from the last 2 weeks. |
| `git log -S function_name` | Finds commits where a specific function was added/removed. |
| `git log -- path/to/file` | Shows commit history for a specific file. |
| `git log --pretty="%h - %s" --author='...' --since="..." --before="..." --no-merges -- t/` | Filters commits by author, date, and file path. |

---

<!-- ```bash
git ls-tree HEAD  # List tree structure
```

```bash
git rev-parse HEAD  # Show latest commit hash
```

```bash
git fsck --full  # Verify Git object database
```

```bash
git log
git log -p -2
git log --stat
git log --pretty=oneline
git log --pretty=format:"%h - %an, %ar : %s"
git log --pretty=format:"%h %s" --graph
git log --since=2.weeks
git log -S function_name
git log -- path/to/file
git log --pretty="%h - %s" --author='Junio C Hamano' --since="2008-10-01" \
   --before="2008-11-01" --no-merges -- t/
``` -->

---


## Key Takeaways for Software Engineers
- **Git is a key-value store**, where SHA-1 hashes uniquely identify objects.
- **Git’s internal model is based on blobs, trees, commits, and tags**.
- **Objects are immutable**, ensuring data integrity. The `objects/` directory contains Git’s entire history, structured as blobs, trees, commits, and tags.
- **Blobs do not track filenames**, only raw content. Filenames are managed via tree objects.
- **Commit history is built on top of objects,** linking blobs → trees → commits.
- **Git plumbing commands (`hash-object`, `cat-file`, `write-tree`, etc.) reveal how Git operates internally.**
- **Shortened hashes can be used**, provided they remain unique.
- **Understanding Git internals enables debugging, optimization, and advanced workflows.**
- **File contents and filenames are stored separately**, allowing efficient tracking.
- **Git avoids duplicate storage** by reusing content if it already exists.
- **Git avoids data duplication** by using content-addressable storage.
- **Git’s commit structure ensures immutability**, making it reliable for version control.
- **Understanding SHA-1 hashing helps in debugging and optimizing Git workflows.**

---

