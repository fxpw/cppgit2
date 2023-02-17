
# cppgit2

<p align="center">
  <img height="100" src="img/logo.png"/>
</p>

<p align="center">
  <a href="https://en.wikipedia.org/wiki/C%2B%2B11">
    <img src="https://img.shields.io/badge/C%2B%2B-11-blue.svg" alt="standard"/>
  </a>
  <a href="https://github.com/p-ranav/tabulate/blob/master/LICENSE">
    <img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="license"/>
  </a>
  <img src="https://img.shields.io/badge/version-0.1.0-blue.svg?cacheSeconds=2592000" alt="version"/>
</p>

`cppgit2` is a `libgit2` wrapper library for use in modern C++ `( >= C++11)`. See the [Build and Integration](#build-and-integration) section for details on how to build and integrate `cppgit2` in your projects.

```cpp
// Create new repo
std::string repo_name = "my_project";
auto repo = repository::init(repo_name, false);

// Write README file
std::string file_name = "README.md";
auto readme = std::ofstream(repo_name + "/" + file_name);
readme << "Hello, World!\n";
readme.close();

// Stage README.md
auto index = repo.index();
index.add_entry_by_path(file_name);
index.write();

// Prepare signatures
auto author = signature("foobar", "foo.bar@baz.com");
auto committer = author;

// Create commit
auto tree_oid = index.write_tree();
repo.create_commit("HEAD", author, committer, "utf-8", "Update README",
    repo.lookup_tree(tree_oid), {});
```

## Table of Contents

*   [Build and Integration](#build-and-integration)
*   [Sample Programs](#sample-programs)
    *   [Initialize a new repository (`git init`)](#initialize-a-new-repository-git-init)
    *   [Clone a repository and checkout specific branch (`git clone --branch`)](#clone-a-repository-and-checkout-specific-branch-git-clone---branch)
    *   [Open an existing repository](#open-an-existing-repository)
    *   [Add and Commit a File (`git add`, `git commit`)](#add-and-commit-a-file-git-add-git-commit)
    *   [Walk Repository Tree (`git ls-tree`)](#walk-repository-tree-git-ls-tree)
    *   [Print Branches (`git branch`)](#print-branches-git-branch)
    *   [Print Commits (`git log`)](#print-commits-git-log)
    *   [Print Repository Tags (`git tag`)](#print-commits-git-log)
    *   [Inspect Repository Objects (`git cat-file (-s|-p|-t)`)](#inspect-repository-objects-git-cat-file)
*   [Design Notes](#design-notes)
    *   [Interoperability with libgit2](#interoperability-with-libgit2)
    *   [Ownership and Memory Management](#ownership-and-memory-management)
    *   [Error Handling](#error-handling)
*   [Version Compatibility](#version-compatibility)
*   [API Coverage](#api-coverage)
*   [Contributing](#contributing)
*   [License](#license)

## Build and Integration

Run the following commands to build `cppgit2`.

**NOTE**: This also builds `libgit2` from source. `libgit2` is a submodule in the `ext/` directory that points to a stable release commit, e.g., [v1.0.0](https://github.com/libgit2/libgit2/releases/tag/v1.0.0).

```bash
git clone --recurse-submodules -j8 https://github.com/fxpw/cppgit2
cd cppgit2
mkdir build && cd build
cmake .. && make
```

The build output is in four directories: `include`, `lib`, `samples` and `test`:

```
include/
├── cppgit2/
├── git2/
└── git2.h
lib/
├── libcppgit2.so -> libcppgit2.so.1
├── libcppgit2.so.0.1.0
├── libcppgit2.so.1 -> libcppgit2.so.0.1.0
├── libcppgit2.static.a
├── libgit2_clar
├── libgit2.pc
├── libgit2.so -> libgit2.so.0
├── libgit2.so.1.0.0
├── libgit2.so.0 -> libgit2.so.1.0.0
└── ...
samples/
test/
```

For integration in your projects,

* Add `build/include` to your `include_directories`
* Add `build/lib` to your `link_directories`
* Build your application, linking with `cppgit2`
* Add `build/lib` to your `LD_LIBRARY_PATH` to load the shared libraries at runtime.

Here's an example using `g++`:

```bash
g++ -std=c++11 -Ibuild/include -Lbuild/lib -o my_sample my_sample.cpp -lcppgit2
export LD_LIBRARY_PATH=build/lib:$LD_LIBRARY_PATH
./my_sample
```

and the same example with `CMake`:

```cmake
PROJECT(my_sample)
CMAKE_MINIMUM_REQUIRED(VERSION 3.8)

INCLUDE_DIRECTORIES("build/include")
ADD_EXECUTABLE(my_sample my_sample.cpp)
find_library(CPPGIT2_LIBRARY cppgit2 HINTS ./build/lib)
TARGET_LINK_LIBRARIES(my_sample ${CPPGIT2_LIBRARY})
SET_PROPERTY(TARGET my_sample PROPERTY CXX_STANDARD 11)
```

## Sample Programs

This section presents some simple examples illustrating various `cppgit2` features. You can find the full set of available examples in the `/samples` directory. Samples are still a work-in-progress. Pull requests are welcome here.

### Initialize a new repository (`git init`)

To initialize a new repository, simply call `repository::init`.

```cpp
#include <cppgit2/repository.hpp>
using namespace cppgit2;

int main() {
  auto repo = repository::init("hello_world", false);
}
```

If you want to create a bare repository, set the second argument to `true`.

### Clone a repository and checkout specific branch (`git clone --branch`)

Let's say you want to clone a repository and checkout a specific branch. Construct an `options` object using `clone::options`, set the checkout branch name, and then use `repository::clone` to clone the repository.

```cpp
#include <cppgit2/repository.hpp>
using namespace cppgit2;

int main() {
  auto url = "https://github.com/fffaraz/awesome-cpp";
  auto branch_name = "gh-pages";
  auto path = "awesome_cpp";

  // Prepare clone options
  clone::options options;
  options.set_checkout_branch_name(branch_name);

  // Clone repository
  auto repo = repository::clone(url, path, options);
}
```

### Open an existing repository

You can open an existing repository with `repository::open`.

```cpp
#include <cppgit2/repository.hpp>
using namespace cppgit2;

int main() {
  auto path = "~/dev/foo/bar";          // bar must contain a .git directory
  auto repo = repository::open(path);
}
```

Use `repository::open_bare` to open a bare repository.

### Create Remote (`git remote`)

```cpp
#include <cppgit2/repository.hpp>
#include <iostream>
using namespace cppgit2;

int main(int argc, char **argv) {
  if (argc == 2) {

    // Create new repo
    auto repo = repository::init(argv[1], false);

    // Create remote
    auto remote = repo.create_remote("origin", "https://github.com/p-ranav/test");

  } else {
    std::cout << "Usage: ./executable <new_repo_path>\n";
  }
}
```

```bash
$ ./create_remote foo

$ cd foo

$ git remote -v
origin	https://github.com/p-ranav/test (fetch)
origin	https://github.com/p-ranav/test (push)
```

### Add and Commit a File (`git add`, `git commit`)

```cpp
#include <cppgit2/repository.hpp>
#include <fstream>
#include <iostream>
using namespace cppgit2;

int main(int argc, char **argv) {
  if (argc == 2) {
    // Create new repo
    auto repo = repository::init(argv[1], false);

    // Write README file
    std::ofstream readme;
    readme.open(std::string{argv[1]} + "/README.md");
    readme << "Hello, World!";
    readme.close();

    // Stage README.md
    auto index = repo.index();
    index.add_entry_by_path("README.md");
    index.write();
    auto tree_oid = index.write_tree();

    // Prepare signatures
    auto author = signature("foobar", "foo.bar@baz.com");
    auto committer = signature("foobar", "foo.bar@baz.com");

    // Create commit
    auto commit_oid =
        repo.create_commit("HEAD", author, committer, "utf-8", "Update README",
                           repo.lookup_tree(tree_oid), {});

    std::cout << "Created commit with ID: " << commit_oid.to_hex_string()
              << std::endl;

  } else {
    std::cout << "Usage: ./executable <new_repo_path>\n";
  }
}
```

```bash
$ ./commit_file foo
Created commit with ID: 34614c460ee9dd6a6e56c1a90c5533b7e284b197

$ cd foo

$ cat README.md
Hello, World!

$ git log --stat
commit 34614c460ee9dd6a6e56c1a90c5533b7e284b197 (HEAD -> master)
Author: foobar <foo.bar@baz.com>
Date:   Thu Mar 19 20:48:07 2020 -0500

    Update README

 README.md | 1 +
 1 file changed, 1 insertion(+)


```

### Walk Repository Tree (`git ls-tree`)

```cpp
#include <cppgit2/repository.hpp>
#include <iostream>
using namespace cppgit2;

int main(int argc, char **argv) {
  if (argc == 2) {
    auto repo = repository::open(argv[1]);

    auto head = repo.head();
    auto head_commit = repo.lookup_commit(head.target());
    auto tree = head_commit.tree();

    tree.walk(tree::traversal_mode::preorder,
              [](const std::string &root, const tree::entry &entry) {
                  auto type = entry.type();
                  std::string type_string{""};
                  switch (type) {
                  case object::object_type::blob:
                    type_string = " - blob";
                    break;
                  case object::object_type::tree:
                    type_string = "tree";
                    break;
                  case object::object_type::commit:
                    type_string = " - commit";
                    break;
                  default:
                    type_string = "other";
                    break;
                  }
                  std::cout << type_string << " [" << entry.id().to_hex_string(8)
                            << "] " << entry.filename() << std::endl;
                  return 0;
              });

  } else {
    std::cout << "Usage: ./executable <repo_path>\n";
  }
}
```

Running this program on the cppgit2 repository yields the following:

```bash
$ cd cppgit2

$ ./build/samples/walk_tree .
 - blob [ae28a6af] .clang-format
 - blob [e4bbfcd3] .gitignore
 - blob [7f2703f2] .gitmodules
 - blob [3ed1714f] CMakeLists.txt
 - blob [f6857659] README.md
 - blob [9f435d50] clang-format.bash
tree [4352ee62] ext
 - commit [17223902] libgit2
tree [7eed768f] img
 - blob [d0fa9dbe] init_add_commit.png
 - blob [dc19ed13] logo.png
tree [4d47c532] include
tree [c9adc194] cppgit2
 - blob [ca1b6723] annotated_commit.hpp
 - blob [4f168526] apply.hpp
 - blob [79ac5ed9] attribute.hpp
 - blob [5bf06b5a] bitmask_operators.hpp
 - blob [10546242] blame.hpp
 - blob [1a9107ab] blob.hpp
 - blob [2bce809e] branch.hpp
 - blob [a56ff9cd] checkout.hpp
 - blob [37bd0139] cherrypick.hpp
 - blob [c30215b9] clone.hpp
...
...
...
```

### Print Branches (`git branch`)

The `repository` class has a number of `for_each_` methods that you can use to iterate over objects. Here's an example that iterates over all the branches in the repository.

```cpp
#include <cppgit2/repository.hpp>
#include <iostream>
using namespace cppgit2;

int main(int argc, char **argv) {
  if (argc == 2) {
    auto repo = repository::open(argv[1]);

    std::cout << "Local branches:\n";
    repo.for_each_branch([](const reference &ref) {
          std::cout << "* " << ref.name() << std::endl;
        },
        branch::branch_type::local);

    std::cout << "Remote branches:\n";
    repo.for_each_branch([](const reference &ref) {
          std::cout << "* " << ref.name() << std::endl;
        },
        branch::branch_type::remote);

  } else {
    std::cout << "Usage: ./executable <repo_path>\n";
  }
}
```

Here's the output when running this program against `libgit2` source code.

```bash
$ ./build/samples/print_branches ext/libgit2
Local branches:
* refs/heads/master
Remote branches:
* refs/remotes/origin/ethomson/checkout_pathspecs
* refs/remotes/origin/gh-pages
* refs/remotes/origin/HEAD
* refs/remotes/origin/maint/v0.99
* refs/remotes/origin/master
* refs/remotes/origin/pks/coverity-fix-sudo
* refs/remotes/origin/bindings/libgit2sharp/020_2
...
...
...
```

### Print Commits (`git log`)

```cpp
#include <cppgit2/repository.hpp>
#include <iostream>
using namespace cppgit2;

int main(int argc, char **argv) {
  if (argc == 2) {
    auto repo = repository::open(argv[1]);

    repo.for_each_commit([](const commit &c) {
      std::cout << c.id().to_hex_string(8)
                << " [" << c.committer().name() << "]"
                << " " << c.summary() << std::endl;
    });

  } else {
    std::cout << "Usage: ./executable <repo_path>\n";
  }
}
```
Running this on the `libgit2` repository yields the following:

```bash
$ ./build/samples/print_commits ext/libgit2
17223902 [GitHub] Merge pull request #5291 from libgit2/ethomson/0_99
b31cd05f [GitHub] Merge pull request #5372 from pks-t/pks/release-script
70062e28 [Patrick Steinhardt] version: update the version number to v0.99
a552c103 [Patrick Steinhardt] docs: update changelog for v0.99
1256b462 [GitHub] Merge pull request #5406 from libgit2/pks/azure-fix-arm32
5254c9bb [GitHub] Merge pull request #5398 from libgit2/pks/valgrind-openssl
e8660708 [GitHub] Merge pull request #5400 from lhchavez/fix-packfile-fuzzer
eaa70c6c [Patrick Steinhardt] tests: object: decrease number of concurrent cache accesses
01a83406 [Patrick Steinhardt] azure: docker: fix ARM builds by replacing gosu(1)
76b49caf [Patrick Steinhardt] azure: docker: synchronize Xenial/Bionic build instructions
f9985688 [Patrick Steinhardt] azure: docker: detect errors when building images
68bfacb1 [Patrick Steinhardt] azure: remove unused Linux setup script
795a5b2c [lhchavez] fuzzers: Fix the documentation
0119e57d [Patrick Steinhardt] streams: openssl: switch approach to silence Valgrind errors
...
...
...
```

### Print Repository Tags (`git tag`)

The `repository` class has a number of `for_each_` methods that you can use to iterate over objects. Here's an example that iterates over all the tags in the repository, printing the name and OID hash for each tag.

```cpp
#include <cppgit2/repository.hpp>
#include <iostream>
using namespace cppgit2;

int main(int argc, char **argv) {
  if (argc == 2) {
    auto repo = repository::open(argv[1]);

    repo.for_each_tag([](const std::string &name, const oid &id) {
      std::cout << "[" << id.to_hex_string(8) << "] " << name << std::endl;
    });

  } else {
    std::cout << "Usage: ./executable <repo_path>\n";
  }
}
```

Running this on the `libgit2` repository yields the following:

```bash
$ ./build/samples/print_tags ext/libgit2
[17223902] refs/tags/v0.99.0
[23f8588d] refs/tags/v0.1.0
[7064938b] refs/tags/v0.10.0
[6dcb09b5] refs/tags/v0.11.0
[40774549] refs/tags/v0.12.0
[37172582] refs/tags/v0.13.0
[52e50c1a] refs/tags/v0.14.0
[3eaf34f4] refs/tags/v0.15.0
[d286dfec] refs/tags/v0.16.0
[5b9fac39] refs/tags/v0.17.0
...
...
...
```

### Inspect Repository Objects (`git cat-file`)

Here's a simplified implementation of `git cat-file` with `cppgit2`

```cpp
#include <cppgit2/repository.hpp>
#include <cstdio>
#include <iomanip>
#include <iostream>
using namespace cppgit2;

void print_signature(const std::string &header, const signature &sig) {
  char sign;
  auto offset = sig.offset();
  if (offset < 0) {
    sign = '-';
    offset = -offset;
  } else {
    sign = '+';
  }

  auto hours = offset / 60;
  auto minutes = offset % 60;

  std::cout << header << " " << sig.name() << " " << "<" << sig.email() << "> "
    << sig.time() << " " << sign;
  std::cout << std::setfill('0') << std::setw(2) << hours;
  std::cout << std::setfill('0') << std::setw(2) << minutes << std::endl;
}

// Printing out a blob is simple, get the contents and print
void show_blob(const blob &blob) {
  std::fwrite(blob.raw_contents(), blob.raw_size(), 1, stdout);
}

// Show each entry with its type, id and attributes
void show_tree(const tree &tree) {
  size_t count = tree.size();
  for (size_t i = 0; i < tree.size(); ++i) {
    auto entry = tree.lookup_entry_by_index(i);

    std::cout << std::setfill('0') <<
        std::oct << std::setw(6) << static_cast<git_filemode_t>(entry.filemode());
    std::cout << " " << object::object_type_to_string(entry.type())
        << " " << entry.id().to_hex_string()
        << "\t" << entry.filename() << std::endl;
  }
}

// Commits and tags have a few interesting fields in their header.
void show_commit(const commit &commit) {
  std::cout << "tree " << commit.tree_id().to_hex_string() << std::endl;

  for (size_t i = 0; i < commit.parent_count(); ++i)
    std::cout << "parent " << commit.parent_id(i).to_hex_string() << std::endl;

  print_signature("author", commit.author());
  print_signature("committer", commit.committer());

  auto message = commit.message();
  if (!message.empty())
    std::cout << "\n" << message << std::endl;
}

void show_tag(const tag &tag) {
  std::cout << "object " << tag.id().to_hex_string() << std::endl;
  std::cout << "type " << object::object_type_to_string(tag.target_type()) << std::endl;
  std::cout << "tag " << tag.name() << std::endl;
  print_signature("tagger", tag.tagger());

  auto tag_message = tag.message();
  if (!tag_message.empty())
    std::cout << "\n" << tag_message << std::endl;
}

int main(int argc, char **argv) {
  if (argc == 3) {
    auto repo_path = repository::discover_path(".");
    auto repo = repository::open(repo_path);

    enum class actions { size, type, pretty };
    actions action;

    if (strncmp(argv[1], "-s", 2) == 0) {
      action = actions::size;
    } else if (strncmp(argv[1], "-t", 2) == 0) {
      action = actions::type;
    } else if (strncmp(argv[1], "-p", 2) == 0) {
      action = actions::pretty;
    }

    auto revision_str = argv[2];
    auto object = repo.revparse_to_object(revision_str);

    switch(action) {
    case actions::type:
        std::cout << object::object_type_to_string(object.type()) << std::endl;
        break;
    case actions::size:
        std::cout << repo.odb().read(object.id()).size() << std::endl;
        break;
    case actions::pretty:
        switch(object.type()) {
            case object::object_type::blob:
                show_blob(object.as_blob());
                break;
            case object::object_type::commit:
                show_commit(object.as_commit());
                break;
            case object::object_type::tree:
                show_tree(object.as_tree());
                break;
            case object::object_type::tag:
                show_tag(object.as_tag());
                break;
            default:
                std::cout << "unknown " << revision_str << std::endl;
                break;
        }
        break;
    }

  } else {
    std::cout << "Usage: ./executable (-s | -t | -p) <object>\n";
  }
}
```

Running this sample on one of the `libgit2` commits yields the following:

```
$ ./cat_file -p 01a8340662749943f3917505dc8ca65006495bec
tree 83d9bef2675178eeb3aa61d17e5c8b0f7b0ec1de
parent 76b49caf6a208e44d19c84caa6d42389f0de6194
author Patrick Steinhardt <ps@pks.im> 1582035643 +0100
committer Patrick Steinhardt <ps@pks.im> 1582040632 +0100

azure: docker: fix ARM builds by replacing gosu(1)

Our nightly builds are currently failing due to our ARM-based jobs.
These jobs crash immediately when entering the Docker container with a
exception thrown by Go's language runtime. As we're able to successfully
builds the Docker images in previous steps, it's unlikely to be a bug in
Docker itself. Instead, this exception is thrown by gosu(1), which is a
Go-based utility to drop privileges and run by our entrypoint.

Fix the issue by dropping gosu(1) in favor of sudo(1).

$ ./cat_file -p 83d9bef2675178eeb3aa61d17e5c8b0f7b0ec1de
100644 blob fd8430bc864cfcd5f10e5590f8a447e01b942bfe	.HEADER
100644 blob 34c5e9234ec18c69a16828dbc9633a95f0253fe9	.editorconfig
100644 blob 176a458f94e0ea5272ce67c36bf30b6be9caf623	.gitattributes
040000 tree e8bfe5af39579a7e4898bb23f3a76a72c368cee6	.github
100644 blob dec3dca06c8fdc1dd7d426bb148b7f99355eaaed	.gitignore
100644 blob 0b16a7e1f1a368d5ca42d580ba2256d1faecddb8	.mailmap
100644 blob 784bab3ee7da6133af679cae7527c4fe4a99b949	AUTHORS
100644 blob 8765a97b5b120259dd59262865ce166f382c0f9e	CMakeLists.txt
100644 blob c0f61fb9158945f7b41abfd640630c914b2eb8d9	COPYING
100644 blob 9dafffec02ef8d9cf8b97f547444f989ddbfa298	README.md
100644 blob f98eebf505a37f756e0ad9d7cc4744397368c436	SECURITY.md
100644 blob bf733273b8cd8b601aaee9a5c10d099a7f6a87e2	api.docurium
100644 blob 2b593dd2cc2c2c252548c7fae4d469c11dd08430	azure-pipelines.yml
040000 tree d9aba7f7d7e9651c176df311dd0489e89266b2b4	azure-pipelines
040000 tree 64e8fd349c9c1dd20f810c22c4e62fe52aab5f18	cmake
040000 tree 5c640a5abe072362ca4bbcf66ef66617c0be0466	deps
040000 tree c84b6d0def9b4b790ece70c7ee68aa3fdf6caa85	docs
040000 tree f852bee8c6bcc3e456f19aff773079eb30abf747	examples
040000 tree 37aaf5d4a9fb0d89d2716236c49474030e36dc93	fuzzers
100644 blob 905bdd24fa23c4d1a03e400a2ae8ecc639769da3	git.git-authors
040000 tree 7fdd111f708aad900604883ce1c161daf64ebb2d	include
100644 blob d33f31c303663dbdbb4baed08ec3cd6c83116367	package.json
040000 tree 97afcc9b6e4ca91001aadf8a3414d043f22918cf	script
040000 tree a08bd8a57d619b736ad2c300614b36ead8d0a333	src
040000 tree dcf5925f8bbda8062ef26ca427c5110868a7f041	tests

$ ./cat_file -s 8765a97b5b120259dd59262865ce166f382c0f9e
11957
```

## Design Notes

### Interoperability with `libgit2`

Most `cppgit2` data structures can be constructed using a `libgit2` C pointer.

```cpp
// Construct libgit2 signature
git_signature sig;
sig.name = (char *)"Foo Bar";
sig.email = (char *)"foo.bar@baz.com";

// Construct cppgit2 wrapper
cppgit2::signature sig2(&sig);

REQUIRE(sig2.name() == std::string(sig.name));
REQUIRE(sig2.email() == std::string(sig.email));
```

Similarly, a `libgit2` C pointer can be extracted from its wrapping `cppgit2` data structure using the `.c_ptr()` method.

```cpp
// Construct cppgit2 OID object
oid oid1("f9de917ac729414151fdce077d4098cfec9a45a5");

// Access libgit2 C ptr
const git_oid *oid1_cptr = oid1.c_ptr();

// Use the libgit2 C API to format
size_t n = 8;
char * oid1_formatted = (char *)malloc(sizeof(char) * n);
git_oid_tostr(oid1_formatted, n + 1, oid1_cptr);

// Results are the same
REQUIRE(oid1.to_hex_string(8) == std::string(oid1_formatted)); // f9de917
```

### Ownership and Memory Management

`libgit2` sometimes allocates memory and returns pointers to data structures that are owned by the user (required to be free'd by the user), and at other times returns a pointer to memory that is managed by the `libgit2` layer.

To properly cleanup memory that is owned by the user, use the `ownership` enum to explicitly specify the ownership when wrapping.

```cpp
cppgit2::tree tree1(&tree_cptr, ownership::user);
```

If the pointer being wrapped is owned by the user, the class destructor will call `git_<type>_free` on the pointer and clean up properly. If you specify the ownership as `ownership::libgit2`, the pointer is left alone.

```cpp
tree::tree(git_tree *c_ptr, ownership owner = ownership::libgit2)
  : c_ptr_(c_ptr), owner_(owner) {}

tree::~tree() {
  if (c_ptr_ && owner_ == ownership::user)
    git_tree_free(c_ptr_);
}
```

### Error Handling

At the moment, `cppgit2` throws a custom `git_exception` anytime the return value from `libgit2` indicates that an error has occurred. Typically `libgit2` functions respond with a return code (0 = good, anything else = error) and `git_error_last` provides the most recent error message. `cppgit2` uses this message when constructing the `git_exception` exception.

Here's a typical example of a wrapped function:

```cpp
void repository::delete_reflog(const std::string &name) {
  if (git_reflog_delete(c_ptr_, name.c_str()))
    throw git_exception();
}
```

where `git_exception` initializes its `what()` message like so:

```cpp
git_exception() {
  auto error = git_error_last();
  message_ = error ? error->message : "unknown error";
}

virtual const char *what() const throw() { return message_; }
```

## Version Compatibility

| libgit2 | cppgit2 |
| --- | --- |
| 0.99.0 | 0.1.0 |

## API Coverage

### annotated

| libgit2 | cppgit2:: |
| --- | --- |
| `git_annotated_commit_free` | `annotated_commit::~annotated_commit` |
| `git_annotated_commit_from_fetchhead` | `repository::create_annotated_commit` |
| `git_annotated_commit_from_ref` | `repository::create_annotated_commit` |
| `git_annotated_commit_from_revspec` | `epository::create_annotated_commit` |
| `git_annotated_commit_id` | `annotated_commit::id` |
| `git_annotated_commit_lookup` | `repository::lookup_annotated_commit` |
| `git_annotated_commit_ref` | `annotated_commit::refname` |


### apply

| libgit2 | cppgit2:: |
| --- | --- |
| `git_apply` | `repository::apply_diff` |
| `git_apply_to_tree` | `repository::apply_diff` |


### attr

| libgit2 | cppgit2:: |
| --- | --- |
| `git_attr_add_macro` | `repository::add_attribute_macro` |
| `git_attr_cache_flush` | `repository::flush_attrobutes_cache` |
| `git_attr_foreach` | `repository::for_each_attribute` |
| `git_attr_get` | `repository::lookup_attribute` |
| `git_attr_get_many` | `repository::lookup_multiple_attributes` |
| `git_attr_value` | `attribute::value` |


### blame

| libgit2 | cppgit2:: |
| --- | --- |
| `git_blame_buffer` | `blame::get_blame_for_buffer` |
| `git_blame_file` | `repository::blame_file` |
| `git_blame_free` | `blame::~blame` |
| `git_blame_get_hunk_byindex` | `blame::hunk_by_index` |
| `git_blame_get_hunk_byline` |`blame::hunk_by_line` |
| `git_blame_get_hunk_count` | `blame::hunk_count` |
| `git_blame_init_options` | `blame::options::options` |
| `git_blame_options_init` | `blame::options::options` |


### blob

| libgit2 | cppgit2:: |
| --- | --- |
| `git_blob_create_from_buffer` | `repository::create_blob_from_buffer` |
| `git_blob_create_from_disk` | `repository::create_blob_from_disk` |
| `git_blob_create_from_stream` | **Not implemented** |
| `git_blob_create_from_stream_commit` | **Not implemented** |
| `git_blob_create_from_workdir` | `repository::create_blobf=_from_workdir` |
| `git_blob_create_fromworkdir` | `repository::create_blobf=_from_workdir` |
| `git_blob_dup` | `blob::copy` |
| `git_blob_filter` | **Not implemented** |
| `git_blob_filtered_content` | **Not implemented** |
| `git_blob_free` | `blob::~blob` |
| `git_blob_id` | `blob::id` |
| `git_blob_is_binary` | `blob::is_binary` |
| `git_blob_lookup` | `repository::lookup_blob` |
| `git_blob_lookup_prefix` | `repository::lookup_blob` |
| `git_blob_owner` | `blob::owner` |
| `git_blob_rawcontent` | `blob::raw_content` |
| `git_blob_rawsize` | `blob::raw_size` |


### branch

| libgit2 | cppgit2:: |
| --- | --- |
| `git_branch_create` | `repository::create_branch` |
| `git_branch_create_from_annotated` | `repository::create_branch` |
| `git_branch_delete` | `repository::delete_branch` |
| `git_branch_is_checked_out` | `repository::is_branched_checked_out` |
| `git_branch_is_head` | `repository::is_head_pointing_to_branch` |
| `git_branch_iterator_free` | `repository::for_each_branch` |
| `git_branch_iterator_new` | `repository::for_each_branch` |
| `git_branch_lookup` | `repository::lookup_branch` |
| `git_branch_move` | `repository::rename_branch` |
| `git_branch_name` | `repository::branch_name` |
| `git_branch_next` | `repository::for_each_branch` |
| `git_branch_remote_name` | `repository::branch_remote_name` |
| `git_branch_set_upstream` | `repository::set_branch_upstream` |
| `git_branch_upstream` | `repository::branch_upstream` |
| `git_branch_upstream_name` | `repository::branch_upstream_name` |
| `git_branch_upstream_remote` | `repository::branch_upstream_remote` |


### buf

| libgit2 | cppgit2:: |
| --- | --- |
| `git_buf_contains_nul` |  `data_buffer::contains_nul` |
| `git_buf_dispose` | `data_buffer::~data_buffer` |
| `git_buf_is_binary` | `data_buffer::is_binary` |

### checkout

| libgit2 | cppgit2:: |
| --- | --- |
| `git_checkout_head` | `repository::checkout_head` |
| `git_checkout_index` | `repository::checkout_index` |
| `git_checkout_options_init` | `repository::checkout::options::options` |
| `git_checkout_tree` | `repository::checkout_tree` |


### cherrypick

| libgit2 | cppgit2:: |
| --- | --- |
| `git_cherrypick` | `repository::cherrypick_commit` |
| `git_cherrypick_commit` | `repository::cherrypick_commit` |
| `git_cherrypick_options_init` | `cherrypick::options::options` |


### clone

| libgit2 | cppgit2:: |
| --- | --- |
| `git_clone` | `repository::clone` |
| `git_clone_options_init` | `clone::options::options` |


### commit

| libgit2 | cppgit2:: |
| --- | --- |
| `git_commit_amend` | `commit::amend` |
| `git_commit_author` | `commit::author` |
| `git_commit_author_with_mailmap` | **Not implemented** |
| `git_commit_body` | `commit::body` |
| `git_commit_committer` | `commit::committer` |
| `git_commit_committer_with_mailmap` | **Not implemented** |
| `git_commit_create` | `repository::create_commit` |
| `git_commit_create_buffer` | `repository::create_commit` |
| `git_commit_create_v` | **Not implemented** |
| `git_commit_create_with_signature` | `repository::create_commit` |
| `git_commit_dup` | `commit::copy` |
| `git_commit_extract_signature` | `repository::extract_signature_from_commit` |
| `git_commit_free` | `commit::~commit` |
| `git_commit_header_field` | `commit::operator[]` |
| `git_commit_id` | `commit::id` |
| `git_commit_lookup` | `repository::lookup_commit` |
| `git_commit_lookup_prefix` | `repository::lookup_commit` |
| `git_commit_message` | `commit::message` |
| `git_commit_message_encoding` | `commit::message_encoding` |
| `git_commit_message_raw` | `commit::message_raw` |
| `git_commit_nth_gen_ancestor` | `commit::ancestor` |
| `git_commit_owner` | `commit::owner` |
| `git_commit_parent` | `commit::parent` |
| `git_commit_parent_id` | `commit::parent_id` |
| `git_commit_parentcount` | `commit::parent_count` |
| `git_commit_raw_header` | `commit::raw_header` |
| `git_commit_summary` | `commit::summary` |
| `git_commit_time` | `commit::time` |
| `git_commit_time_offset` | `commit::time_offset` |
| `git_commit_tree` | `commit::tree` |
| `git_commit_tree_id` | `commit::tree_id` |


### config

| libgit2 | cppgit2:: |
| --- | --- |
| `git_config_add_file_ondisk` | `repository::add_ondisk_config_file` |
| `git_config_backend_foreach_match` | **Not implemented** |
| `git_config_delete_entry` | `config::delete_entry` |
| `git_config_delete_multivar` | `config::delete_entry` |
| `git_config_entry_free` | `config::entry::~entry` |
| `git_config_find_global` | `config::locate_global_config` |
| `git_config_find_programdata` | `config::locate_global_config_in_programdata` |
| `git_config_find_system` | `config::locate_global_system_config` |
| `git_config_find_xdg` | `config::locate_global_xdg_compatible_config`  |
| `git_config_foreach` | `config::for_each` |
| `git_config_foreach_match` | `config::for_each` |
| `git_config_free` | `config::~config` |
| `git_config_get_bool` | `config::value_as_bool` |
| `git_config_get_entry` | `config::operator[]` |
| `git_config_get_int32` | `config::value_as_int32` |
| `git_config_get_int64` | `config::value_as_int64` |
| `git_config_get_mapped` | **Not implemented** |
| `git_config_get_multivar_foreach` | **Not implemented** |
| `git_config_get_path` | `config::path` |
| `git_config_get_string` | `config::value_as_string` |
| `git_config_get_string_buf` | `config::value_as_data_buffer` |
| `git_config_iterator_free` | `config::for_each_entry` |
| `git_config_iterator_glob_new` | **Not implemented** |
| `git_config_iterator_new` | `config::for_each_entry` |
| `git_config_lock` | `config::lock` |
| `git_config_lookup_map_value` | **Not implemented** |
| `git_config_multivar_iterator_new` | **Not implemented** |
| `git_config_new` | `config::new_config` |
| `git_config_next` | `config::for_each_entry` |
| `git_config_open_default` | `config::open_default_config`  |
| `git_config_open_global` | `config::open_global_config` |
| `git_config_open_level` | `config::open_config_at_level` |
| `git_config_open_ondisk` | **Not implemented** |
| `git_config_parse_bool` | `config::parse_as_bool` |
| `git_config_parse_int32` | `config::parse_as_int32` |
| `git_config_parse_int64` | `config::parse_as_int64` |
| `git_config_parse_path` | `config::parse_path` |
| `git_config_set_bool` | `config::insert_entry` |
| `git_config_set_int32` | `config::insert_entry` |
| `git_config_set_int64` | `config::insert_entry` |
| `git_config_set_multivar` | `config::insert_entry` |
| `git_config_set_string` | `config::insert_entry` |
| `git_config_snapshot` | `config::snapshot` |

### cred

| libgit2 | cppgit2:: |
| --- | --- |
| `git_cred_default_new` | `credential::credential` |
| `git_cred_free` | `credential::~credential` |
| `git_cred_get_username` | `credential::username` |
| `git_cred_has_username` | `credential::has_username` |
| `git_cred_ssh_custom_new` | `credential::credential` |
| `git_cred_ssh_interactive_new` | `credential::credential` |
| `git_cred_ssh_key_from_agent` | `credential::credential` |
| `git_cred_ssh_key_memory_new` | **Not Implemented** |
| `git_cred_ssh_key_new` | `credential::credential` |
| `git_cred_username_new` | **Not Implemented** |
| `git_cred_userpass` | **Not Implemented** |
| `git_cred_userpass_plaintext_new` | `credential::credential` |

### diff

| libgit2 | cppgit2:: |
| --- | --- |
| `git_diff_blob_to_buffer` | `diff::diff_blob_to_buffer` |
| `git_diff_blobs` | `diff::compare_files` |
| `git_diff_buffers` | `diff::diff_between_buffers` |
| `git_diff_commit_as_email` | `diff::create_diff_commit_as_email` |
| `git_diff_find_options_init` | `diff::find_options::find_options` |
| `git_diff_find_similar` | `diff::find_similar` |
| `git_diff_foreach` | `diff::for_each` |
| `git_diff_format_email` | `diff::format_email` |
| `git_diff_format_email_options_init` | `diff::format_email_options::format_email_options()` |
| `git_diff_free` | `diff::~diff` |
| `git_diff_from_buffer` | `diff::diff` |
| `git_diff_get_delta` | `diff::operator[]` |
| `git_diff_get_stats` | `diff::diff_stats` |
| `git_diff_index_to_index` | `repository::create_diff_index_to_index` |
| `git_diff_index_to_workdir` | `repository::create_diff_index_to_workdir` |
| `git_diff_is_sorted_icase` | `diff::is_sorted_case_sensitive` |
| `git_diff_merge` | `diff::merge` |
| `git_diff_num_deltas` | `diff::size` |
| `git_diff_num_deltas_of_type` | `diff::size` |
| `git_diff_options_init` | `diff::options::options` |
| `git_diff_patchid` | `diff::patchid` |
| `git_diff_patchid_options_init` | `diff::patchid_options::patchid_options` |
| `git_diff_print` | `diff::print` |
| `git_diff_stats_deletions` | `diff::stats::deletions` |
| `git_diff_stats_files_changed` | `diff::stats::files_changed` |
| `git_diff_stats_free` | `diff::stats::~stats` |
| `git_diff_stats_insertions` | `diff::stats::insertions` |
| `git_diff_stats_to_buf` | `diff::stats::to_buffer` |
| `git_diff_status_char` | `diff::status_char` |
| `git_diff_to_buf` | `diff::to_string` |
| `git_diff_tree_to_index` | `repository::create_diff_tree_to_index` |
| `git_diff_tree_to_tree` | `repository::create_diff_tree_to_tree` |
| `git_diff_tree_to_workdir` | `repository::create_diff_tree_to_workdir` |
| `git_diff_tree_to_workdir_with_index` | `create_diff_tree_to_workdir_with_index` |

### error

| libgit2 | cppgit2:: |
| --- | --- |
| `git_error_clear` | `git_exception::clear` |
| `git_error_last` | `git_exception::git_exception` |
| `git_error_set_oom` | **Not Implemented** |
| `git_error_set_str` | **Not Implemented** |

### fetch

| libgit2 | cppgit2:: |
| --- | --- |
| `git_fetch_options_init` | `fetch::options::options` |

### graph

| libgit2 | cppgit2:: |
| --- | --- |
| `git_graph_ahead_behind` | `repository::unique_commits_ahead_behind` |
| `git_graph_descendant_of` | `repository::is_descendant_of` |

### ignore

| libgit2 | cppgit2:: |
| --- | --- |
| `git_ignore_add_rule` | `repository::add_ignore_rules` |
| `git_ignore_clear_internal_rules` | `repository::clear_ignore_rules` |
| `git_ignore_path_is_ignored` | `repository::is_path_ignored` |

### index

| libgit2 | cppgit2:: |
| --- | --- |
| `git_index_add` | `index::add_entry` |
| `git_index_add_all` | `index::add_entries_that_match` |
| `git_index_add_bypath` | `index::add_entry_by_path` |
| `git_index_add_from_buffer` | `index::add_entry_from_buffer` |
| `git_index_caps` | `index::capability_flags` |
| `git_index_checksum` | `index::checksum` |
| `git_index_clear` | `index::clear` |
| `git_index_conflict_add` | `index::add_conflict_entry` |
| `git_index_conflict_cleanup` | `index::remove_all_conflicts` |
| `git_index_conflict_get` | **Not Implemented** |
| `git_index_conflict_iterator_free` | `index::for_each_conflict` |
| `git_index_conflict_iterator_new` | `index::for_each_conflict` |
| `git_index_conflict_next` | `index::for_each_conflict` |
| `git_index_conflict_remove` | `index::remove_conflict_entries` |
| `git_index_entry_is_conflict` | `index::entry::is_conflict` |
| `git_index_entry_stage` | `index::entry::entry_stage` |
| `git_index_entrycount` | `index::size` |
| `git_index_find` | `index::find_first` |
| `git_index_find_prefix` | `index::find_first_matching_prefix` |
| `git_index_free` | `index::~index` |
| `git_index_get_byindex` | `index::operator[]` |
| `git_index_get_bypath` | `index::entry_in_path` |
| `git_index_has_conflicts` | `index::has_conflicts` |
| `git_index_iterator_free` | `index::for_each` |
| `git_index_iterator_new` | `index::for_each` |
| `git_index_iterator_next` | `index::for_each` |
| `git_index_new` | `index::index` |
| `git_index_open` | `index::open` |
| `git_index_owner` | `index::owner` |
| `git_index_path` | `index::path` |
| `git_index_read` | `index::read` |
| `git_index_read_tree` | `index::read_tree` |
| `git_index_remove` | `index::remove_entry` |
| `git_index_remove_all` | `index::remove_entries_that_match` |
| `git_index_remove_bypath` | `index::remove_entry_by_path` |
| `git_index_remove_directory` | `index::remove_entries_in_directory` |
| `git_index_set_caps` | `index::set_index_capabilities` |
| `git_index_set_version` | `index::set_version` |
| `git_index_update_all` | `index::update_entries_that_match` |
| `git_index_version` | `index::version` |
| `git_index_write` | `index::write` |
| `git_index_write_tree` | `index::write_tree` |
| `git_index_write_tree_to` | `index::write_tree_to` |


### indexer

| libgit2 | cppgit2:: |
| --- | --- |
| `git_indexer_append` | `indexer::append` |
| `git_indexer_commit` | `indexer::commit` |
| `git_indexer_free` | `indexer::~indexer` |
| `git_indexer_hash` | `indexer::hash` |
| `git_indexer_new` | `indexer::indexer` |
| `git_indexer_options_init` | `indexer::options::options` |


### libgit2

| libgit2 | cppgit2:: |
| --- | --- |
| `git_libgit2_features` | **Not Implemented** |
| `git_libgit2_init` | `libgit2_api::libgit2_api` |
| `git_libgit2_opts` | **Not Implemented** |
| `git_libgit2_shutdown` | `libgit2_api::~libgit2_api` |
| `git_libgit2_version` | `libgit2_api::version` |


### merge

| libgit2 | cppgit2:: |
| --- | --- |
| `git_merge` | `repository::merge_commits` |
| `git_merge_analysis` | `repository::analyze_merge` |
| `git_merge_analysis_for_ref` | `repository::analyze_merge` |
| `git_merge_base` | `repository::find_merge_base` |
| `git_merge_base_many` | `repository::find_merge_bases` |
| `git_merge_base_octopus` | `repository::find_merge_base_for_octopus_merge` |
| `git_merge_bases` | `repository::find_merge_bases` |
| `git_merge_bases_many` | `repository::find_merge_bases` |
| `git_merge_commits` | `repository::merge_commits` |
| `git_merge_file` | `merge::merge_files` |
| `git_merge_file_from_index` | `repository::merge_file_from_index` |
| `git_merge_file_input_init` | `merge::file::input::input` |
| `git_merge_file_options_init` | `merge::file::options::options` |
| `git_merge_file_result_free` | `merge::file::result::~result` |
| `git_merge_options_init` | `merge::options::options` |
| `git_merge_trees` | `repository::merge_trees` |

### note

| libgit2 | cppgit2:: |
| --- | --- |
| `git_note_author` | `note::author` |
| `git_note_commit_create` | `repository::create_note` |
| `git_note_commit_iterator_new` | **Not Implemented** |
| `git_note_commit_read` | `repository::read_note` |
| `git_note_commit_remove` | `repository::remove_note` |
| `git_note_committer` | `note::committer` |
| `git_note_create` | `repository::create_note` |
| `git_note_default_ref` | `repository::default_notes_reference` |
| `git_note_foreach` | `repository::for_each_note` |
| `git_note_free` | `note::~note` |
| `git_note_id` | `note::id` |
| `git_note_iterator_free` | **Not Implemented** |
| `git_note_iterator_new` | **Not Implemented** |
| `git_note_message` | `note::message` |
| `git_note_next` | **Not Implemented** |
| `git_note_read` | `repository::read_note` |
| `git_note_remove` | `repository::remove_note` |


### object

| libgit2 | cppgit2:: |
| --- | --- |
| `git_object__size` | **Not implemented** |
| `git_object_dup` | `object::copy` |
| `git_object_free` | `object::~object` |
| `git_object_id` | `object::id` |
| `git_object_lookup` | `repository::lookup_object` |
| `git_object_lookup_bypath` | `repository::lookup_object` |
| `git_object_lookup_prefix` | `repository::lookup_object` |
| `git_object_owner` | `object::owner` |
| `git_object_peel` | `object::peel_until` |
| `git_object_short_id` | `object::short_id` |
| `git_object_string2type` | `object::type_from_string` |
| `git_object_type` | `object::type` |
| `git_object_type2string` | `object::string_from_type` |
| `git_object_typeisloose` | `object::is_type_loose` |

### odb

| libgit2 | cppgit2:: |
| --- | --- |
| `git_odb_add_alternate` | `odb::add_alternate_backend` |
| `git_odb_add_backend` | `odb::add_backend` |
| `git_odb_add_disk_alternate` | `odb::add_disk_alternate_backend` |
| `git_odb_backend_loose` | `odb::create_backend_for_loose_objects` |
| `git_odb_backend_one_pack` | `odb::create_backend_for_one_packfile` |
| `git_odb_backend_pack` | `odb::create_backend_for_packfiles` |
| `git_odb_exists` | `odb::exists` |
| `git_odb_exists_prefix` | `odb::exists` |
| `git_odb_expand_ids` | `odb::expand_ids` |
| `git_odb_foreach` | `odb::for_each` |
| `git_odb_free` | `odb::~odb` |
| `git_odb_get_backend` | `odb::operator[]` |
| `git_odb_hash` | `odb::hash` |
| `git_odb_hashfile` | `odb::hash_file` |
| `git_odb_new` | `odb::odb` |
| `git_odb_num_backends` | `odb::size` |
| `git_odb_object_data` | `odb::object::data` |
| `git_odb_object_dup` | `odb::object::copy` |
| `git_odb_object_free` | `odb::object::~object` |
| `git_odb_object_id` | `odb::object::id` |
| `git_odb_object_size` | `odb::object::size` |
| `git_odb_object_type` | `odb::object::type` |
| `git_odb_open` | `odb::open` |
| `git_odb_open_rstream` | `odb::open_rstream` |
| `git_odb_open_wstream` | `odb::open_wstream` |
| `git_odb_read` | `odb::read` |
| `git_odb_read_header` | `odb::read_header` |
| `git_odb_read_prefix` | `odb::read_prefix` |
| `git_odb_refresh` | `odb::refresh` |
| `git_odb_stream_finalize_write` | `odb::stream::finalize_write` |
| `git_odb_stream_free` | `odb::stream::~stream` |
| `git_odb_stream_read` | `odb::stream::read` |
| `git_odb_stream_write` | `odb::stream::write` |
| `git_odb_write` | `odb::write` |
| `git_odb_write_pack` | **Not Implemented** |


### oid

| libgit2 | cppgit2:: |
| --- | --- |
| `git_oid_cmp` | `oid::compare` |
| `git_oid_cpy` | `oid::copy` |
| `git_oid_equal` | `oid::operator==` |
| `git_oid_fmt` | **Not implemented** |
| `git_oid_fromraw` | `oid::oid` |
| `git_oid_fromstr` | `oid::oid` |
| `git_oid_fromstrn` | `oid::oid` |
| `git_oid_fromstrp` | **Not implemented** |
| `git_oid_is_zero` | `oid::is_zero` |
| `git_oid_iszero` | `oid::is_zero` |
| `git_oid_ncmp` | `oid::compare` |
| `git_oid_nfmt` | **Not implemented** |
| `git_oid_pathfmt` | `oid::to_path_string` |
| `git_oid_shorten_add` | `oid::shorten::add` |
| `git_oid_shorten_free` | `oid::shorten::~shorten` |
| `git_oid_shorten_new` | `oid::shorten::shorten` |
| `git_oid_strcmp` | `oid::compare` |
| `git_oid_streq` | `oid::operator==` |
| `git_oid_tostr` | `oid::to_hex_string` |
| `git_oid_tostr_s` | `oid::to_hex_string` |


### oidarray

| libgit2 | cppgit2:: |
| --- | --- |
| `git_oidarray_free` | **Not Implemented** |


### packbuilder

| libgit2 | cppgit2:: |
| --- | --- |
| `git_packbuilder_foreach` | `pack_builder::for_each_object` |
| `git_packbuilder_free` | `pack_builder::~pack_builder` |
| `git_packbuilder_hash` | `pack_builder::hash` |
| `git_packbuilder_insert` | `pack_builder::insert_object` |
| `git_packbuilder_insert_commit` | `pack_builder::insert_commit` |
| `git_packbuilder_insert_recur` | `pack_builder::insert_object_recursively` |
| `git_packbuilder_insert_tree` | `pack_builder::insert_tree` |
| `git_packbuilder_insert_walk` | `pack_builder::insert_revwalk` |
| `git_packbuilder_new` | `repository::initialize_pack_builder` |
| `git_packbuilder_object_count` | `pack_builder::size` |
| `git_packbuilder_set_callbacks` | `pack_builder::set_progress_callback` |
| `git_packbuilder_set_threads` | `pack_builder::set_threads` |
| `git_packbuilder_write` | `pack_builder::write` |
| `git_packbuilder_write_buf` | `pack_builder::write_to_buffer` |
| `git_packbuilder_written` | `pack_builder::written` |


### patch

| libgit2 | cppgit2:: |
| --- | --- |
| `git_patch_free` | `patch::~patch` |
| `git_patch_from_blob_and_buffer` | `patch::patch` |
| `git_patch_from_blobs` | `patch::patch` |
| `git_patch_from_buffers` | `patch::patch` |
| `git_patch_from_diff` | `patch::patch` |
| `git_patch_get_delta` | `patch::delta` |
| `git_patch_get_hunk` | `patch::hunk` |
| `git_patch_get_line_in_hunk` | `patch::line_in_hunk` |
| `git_patch_line_stats` | `patch::line_stats` |
| `git_patch_num_hunks` | `patch::num_hunks` |
| `git_patch_num_lines_in_hunk` | `patch::num_lines_in_hunk` |
| `git_patch_print` | `patch::print` |
| `git_patch_size` | `patch::size` |
| `git_patch_to_buf` | `patch::to_buffer` |


### pathspec

| libgit2 | cppgit2:: |
| --- | --- |
| `git_pathspec_free` | `pathspec::~pathspec` |
| `git_pathspec_match_diff` | `pathspec::match_diff` |
| `git_pathspec_match_index` | `pathspec::match_index` |
| `git_pathspec_match_list_diff_entry` | `pathspec::match_list::diff_entry` |
| `git_pathspec_match_list_entry` | `pathspec::match_list::entry` |
| `git_pathspec_match_list_entrycount` | `pathspec::match_list::size` |
| `git_pathspec_match_list_failed_entry` | `pathspec::match_list::failed_entry` |
| `git_pathspec_match_list_failed_entrycount` | `pathspec::match_list::failed_entrycount` |
| `git_pathspec_match_list_free` | `pathspec::match_list::~match_list` |
| `git_pathspec_match_tree` | `pathspec::match_free` |
| `git_pathspec_match_workdir` | `pathspec::match_workdir` |
| `git_pathspec_matches_path` | `pathspec::matches_path` |
| `git_pathspec_new` | `pathspec::compile` |

### proxy

| libgit2 | cppgit2:: |
| --- | --- |
| `git_proxy_options_init` | `proxy::options::options` |

### push

| libgit2 | cppgit2:: |
| --- | --- |
| `git_push_options_init` | `push::options::options` |

### rebase

| libgit2 | cppgit2:: |
| --- | --- |
| `git_rebase_abort` | `rebase::abort` |
| `git_rebase_commit` | `rebase::commit` |
| `git_rebase_finish` | `rebase::finish` |
| `git_rebase_free` | `rebase::~rebase` |
| `git_rebase_init` | `repository::init_rebase` |
| `git_rebase_inmemory_index` | `rebase::index` |
| `git_rebase_next` | `rebase::next` |
| `git_rebase_onto_id` | `rebase::onto_id` |
| `git_rebase_onto_name` | `rebase::onto_name` |
| `git_rebase_open` | `repository::open_rebase` |
| `git_rebase_operation_byindex` | `rebase::operator[]` |
| `git_rebase_operation_current` | `rebase::current_operation` |
| `git_rebase_operation_entrycount` | `rebase::size` |
| `git_rebase_options_init` | `rebase::options::options` |
| `git_rebase_orig_head_id` | `rebase::original_head_id` |
| `git_rebase_orig_head_name` | `rebase::original_head_name` |


### refdb

| libgit2 | cppgit2:: |
| --- | --- |
| `git_refdb_compress` | `refdb::compress` |
| `git_refdb_free` | `refdb::~refdb` |
| `git_refdb_new` | `repository::create_reference_database` |
| `git_refdb_open` | `repository::open_reference_database` |


### reference

| libgit2 | cppgit2:: |
| --- | --- |
| `git_reference_cmp` | `reference::compare` |
| `git_reference_create` | `repository::create_reference` |
| `git_reference_create_matching` | `repository::create_reference` |
| `git_reference_delete` | `reference::delete_reference` |
| `git_reference_dup` | `reference::copy` |
| `git_reference_dwim` | `repository::lookup_reference_by_dwim` |
| `git_reference_ensure_log` | `repository::ensure_reflog_for_reference` |
| `git_reference_foreach` | `repository::for_each_reference` |
| `git_reference_foreach_glob` | `repository::for_each_reference_glob` |
| `git_reference_foreach_name` | `repository::for_each_reference_name` |
| `git_reference_free` | `reference::~reference` |
| `git_reference_has_log` | `repository::reference_has_reflog` |
| `git_reference_is_branch` | `reference::is_branch` |
| `git_reference_is_note` | `reference::is_note` |
| `git_reference_is_remote` | `reference::is_remote` |
| `git_reference_is_tag` | `reference::is_tag` |
| `git_reference_is_valid_name` | `reference::is_valid_name` |
| `git_reference_iterator_free` | `repository::for_each_reference` |
| `git_reference_iterator_glob_new` | `repository::for_each_reference_glob` |
| `git_reference_iterator_new` | `repository::for_each_reference` |
| `git_reference_list` | `repository::reference_list` |
| `git_reference_lookup` | `repository::lookup_reference` |
| `git_reference_name` | `reference::name` |
| `git_reference_name_to_id` | `repository::reference_name_to_id` |
| `git_reference_next` | `repository::for_each_reference` |
| `git_reference_next_name` | `repository::for_each_reference_name` |
| `git_reference_normalize_name` | `reference::normalize_name` |
| `git_reference_owner` | `reference::owner` |
| `git_reference_peel` | reference::peel_until` |
| `git_reference_remove` | `repository::delete_reference` |
| `git_reference_rename` | `reference::rename` |
| `git_reference_resolve` | `reference::resolve` |
| `git_reference_set_target` | `reference::set_target` |
| `git_reference_shorthand` | `reference::shorthand_name` |
| `git_reference_symbolic_create` | `repository::create_symbolic_reference` |
| `git_reference_symbolic_create_matching` | `repository::create_symbolic_reference` |
| `git_reference_symbolic_set_target` | `reference::set_symbolic_target` |
| `git_reference_symbolic_target` | `reference::symbolic_target` |
| `git_reference_target` | `reference::target` |
| `git_reference_target_peel` | `reference::peeled_target` |
| `git_reference_type` | `reference::type` |


### reflog

| libgit2 | cppgit2:: |
| --- | --- |
| `git_reflog_append` | `reflog::append` |
| `git_reflog_delete` | `repository::delete_reflog` |
| `git_reflog_drop` | `reflog::remove` |
| `git_reflog_entry_byindex` | `reflog::operator[]` |
| `git_reflog_entry_committer` | `reflog::entry::committer` |
| `git_reflog_entry_id_new` | `reflog::entry::new_id` |
| `git_reflog_entry_id_old` | `reflog::entry::old_id` |
| `git_reflog_entry_message` | `reflog::entry::message` |
| `git_reflog_entrycount` | `reflog::size` |
| `git_reflog_free` | `reflog::~reflog` |
| `git_reflog_read` | `repository::read_reflog` |
| `git_reflog_rename` | `repository::rename_reflog` |
| `git_reflog_write` | `repository::write_to_disk` |


### refspec

| libgit2 | cppgit2:: |
| --- | --- |
| `git_refspec_direction` | `refspec::direction` |
| `git_refspec_dst` | `refspec::destination` |
| `git_refspec_dst_matches` | `refspec::destination_matches_reference` |
| `git_refspec_force` | `refspec::is_force_update_enabled` |
| `git_refspec_free` | `refspec::~refspec` |
| `git_refspec_parse` | `refspec::parse` |
| `git_refspec_rtransform` | `refspec::transform_target_to_source_reference` |
| `git_refspec_src` | `refspec::source` |
| `git_refspec_src_matches` | `refspec::source_matches_reference` |
| `git_refspec_string` | `refspec::to_string` |
| `git_refspec_transform` | `refspec::transform_reference` |


### remote

| libgit2 | cppgit2:: |
| --- | --- |
| `git_remote_add_fetch` | `repository::add_fetch_refspec_to_remote` |
| `git_remote_add_push` | `repository::add_push_refspec_to_remote` |
| `git_remote_autotag` | `remote::autotag_option` |
| `git_remote_connect` | `remote::connect` |
| `git_remote_connected` | `remote::is_connected` |
| `git_remote_create` | `repository::create_remote` |
| `git_remote_create_anonymous` | `repository::create_anonymous_remote` |
| `git_remote_create_detached` | `remote::create_detached_remote` |
| `git_remote_create_options_init` | `remote::create_options::create_options` |
| `git_remote_create_with_fetchspec` | `repository::create_remote` |
| `git_remote_create_with_opts` | `remote::create_remote` |
| `git_remote_default_branch` | `remote::default_branch` |
| `git_remote_delete` | `repository::delete_remote` |
| `git_remote_disconnect` | `remote::disconnect` |
| `git_remote_download` | `remote::download` |
| `git_remote_dup` | `remote::copy` |
| `git_remote_fetch` | `remote::fetch_` |
| `git_remote_free` | `remote::~remote` |
| `git_remote_get_fetch_refspecs` | `remote::fetch_refspec` |
| `git_remote_get_push_refspecs` | `remote::push_refspec` |
| `git_remote_get_refspec` | `remote::operator[]` |
| `git_remote_init_callbacks` | `remote::callbacks::callbacks` |
| `git_remote_is_valid_name` | `remote::is_valid_name` |
| `git_remote_list` | `repository::remote_list` |
| `git_remote_lookup` | `repository::lookup_remote` |
| `git_remote_ls` | `remote::reference_advertisement_list` |
| `git_remote_name` | `remote::name` |
| `git_remote_owner` | `remote::owner` |
| `git_remote_prune` | `remote::prune` |
| `git_remote_prune_refs` | `remote::prune_references` |
| `git_remote_push` | `remote::push` |
| `git_remote_pushurl` | `remote::push_url` |
| `git_remote_refspec_count` | `remote::size` |
| `git_remote_rename` | `repository::rename_remote` |
| `git_remote_set_autotag` | `repository::set_remote_autotag` |
| `git_remote_set_pushurl` | `repository::set_remote_push_url` |
| `git_remote_set_url` | `repository::set_remote_url` |
| `git_remote_stats` | `remote::stats` |
| `git_remote_stop` | `remote::stop` |
| `git_remote_update_tips` | `remote::update_tips` |
| `git_remote_upload` | `remote::upload` |
| `git_remote_url` | `remote::url` |


### repository

| libgit2 | cppgit2:: |
| --- | --- |
| `git_repository_commondir` | `repository::commondir` |
| `git_repository_config` | `repository::config` |
| `git_repository_config_snapshot` | `repository::config_snapshot` |
| `git_repository_detach_head` | `repository::detach_head` |
| `git_repository_discover` | `repository::discover_path` |
| `git_repository_fetchhead_foreach` | `repository::for_each_fetch_head` |
| `git_repository_free` | `repository::~repository` |
| `git_repository_get_namespace` | `repository::namespace_` |
| `git_repository_hashfile` | `repository::hashfile` |
| `git_repository_head` | `repository::head` |
| `git_repository_head_detached` | `repository::is_head_detached` |
| `git_repository_head_detached_for_worktree` | `repository::is_head_detached_for_worktree` |
| `git_repository_head_for_worktree` | `repository::head_for_worktree` |
| `git_repository_head_unborn` | `repository::is_head_unborn` |
| `git_repository_ident` | `repository::identity` |
| `git_repository_index` | `repository::index` |
| `git_repository_init` | `repository::init` |
| `git_repository_init_ext` | `repository::init_ext` |
| `git_repository_init_options_init` | `repository::init_options::init_options` |
| `git_repository_is_bare` | `repository::is_bare` |
| `git_repository_is_empty` | `repository::is_empty` |
| `git_repository_is_shallow` | `repository::is_shallow` |
| `git_repository_is_worktree` | `repository::is_worktree` |
| `git_repository_item_path` | `repository::path` |
| `git_repository_mergehead_foreach` | `repository::for_each_merge_head` |
| `git_repository_message` | `repository::message` |
| `git_repository_message_remove` | `repository::remove_message` |
| `git_repository_odb` | `repository::odb` |
| `git_repository_open` | `repository::open` |
| `git_repository_open_bare` | `repository::open_bare` |
| `git_repository_open_ext` | `repository::open_ext` |
| `git_repository_open_from_worktree` | `repository::open_from_worktree` |
| `git_repository_path` | `repository::path` |
| `git_repository_refdb` | `repository::refdb` |
| `git_repository_set_head` | `repository::set_head` |
| `git_repository_set_head_detached` | `repository::set_head_detached` |
| `git_repository_set_head_detached_from_annotated` | `repository::set_head_detached` |
| `git_repository_set_ident` | `repository::set_identity` |
| `git_repository_set_namespace` | `repository::set_namespace` |
| `git_repository_set_workdir` | `repository::set_workdir` |
| `git_repository_state` | `repository::state` |
| `git_repository_state_cleanup` | `repository::cleanup_state` |
| `git_repository_workdir` | `repository::workdir` |
| `git_repository_wrap_odb` | `repository::wrap_odb` |


### reset

| libgit2 | cppgit2:: |
| --- | --- |
| `git_reset` | `repository::reset` |
| `git_reset_default` | `repository::reset_default` |
| `git_reset_from_annotated` | `repository::reset` |


### revert

| libgit2 | cppgit2:: |
| --- | --- |
| `git_revert` | `repository::revert_commit` |
| `git_revert_commit` | `repository::revert_commit` |
| `git_revert_options_init` | `revert::options::options` |


### revparse

| libgit2 | cppgit2:: |
| --- | --- |
| `git_revparse` | `repository::revparse` |
| `git_revparse_ext` | `repository::revparse_to_object_and_reference` |
| `git_revparse_single` | `repository::revparse_to_object` |


### revwalk

| libgit2 | cppgit2:: |
| --- | --- |
| `git_revwalk_add_hide_cb` | `revwalk::add_hide_callback` |
| `git_revwalk_free` | `revwalk::~revwalk` |
| `git_revwalk_hide` | `revwalk::hide` |
| `git_revwalk_hide_glob` | `revwalk::hide_glob` |
| `git_revwalk_hide_head` | `revwalk::hide_head` |
| `git_revwalk_hide_ref` | `revwalk::hide_reference` |
| `git_revwalk_new` | `repository::create_revwalk` |
| `git_revwalk_next` | `revwalk::next` |
| `git_revwalk_push` | `revwalk::push` |
| `git_revwalk_push_glob` | `revwalk::push_glob` |
| `git_revwalk_push_head` | `revwalk::push_head` |
| `git_revwalk_push_range` | `revwalk::push_range` |
| `git_revwalk_push_ref` | `revwalk::push_reference` |
| `git_revwalk_repository` | `revwalk::repository` |
| `git_revwalk_reset` | `revwalk::reset` |
| `git_revwalk_simplify_first_parent` | `revwalk::simplify_first_parent` |
| `git_revwalk_sorting` | `revwalk::set_sorting_mode` |

### signature

| libgit2 | cppgit2:: |
| --- | --- |
| `git_signature_default` | `repository::default_signature` |
| `git_signature_dup` | `signature::copy` |
| `git_signature_free` | `signature::~signature` |
| `git_signature_from_buffer` | `signature::signature` |
| `git_signature_new` | `signature::signature` |
| `git_signature_now` | `signature::signature` |


### stash

| libgit2 | cppgit2:: |
| --- | --- |
| `git_stash_apply` | `repository::apply_stash` |
| `git_stash_apply_options_init` | `stash::options::options` |
| `git_stash_drop` | `repository::drop_stash` |
| `git_stash_foreach` | `repository::for_each_stash` |
| `git_stash_pop` | `repository::pop_stash` |
| `git_stash_save` | `repository::save_stash` |


### status

| libgit2 | cppgit2:: |
| --- | --- |
| `git_status_byindex` | `status::list::operator[]` |
| `git_status_file` | `repository::status_file` |
| `git_status_foreach` | `repository::for_each_status` |
| `git_status_foreach_ext` | `repository::for_each_status` |
| `git_status_list_entrycount` | `status::list::size` |
| `git_status_list_free` | `status::list::~list` |
| `git_status_list_new` | `repository::status_list` |
| `git_status_options_init` | `status::options::options` |
| `git_status_should_ignore` | `repository::should_ignore` |


### strarray

| libgit2 | cppgit2:: |
| --- | --- |
| `git_strarray_copy` | `strarray::copy` |
| `git_strarray_free` | `strarray::~strarray` |


### submodule

| libgit2 | cppgit2:: |
| --- | --- |
| `git_submodule_add_finalize` | `submodule::resolve_setup` |
| `git_submodule_add_setup` | `repository::setup_submodule` |
| `git_submodule_add_to_index` | `submodule::add_to_index` |
| `git_submodule_branch` | `submodule::branch_name` |
| `git_submodule_clone` | `submodule::clone` |
| `git_submodule_fetch_recurse_submodules` | `submodule::recuse_submodules_option` |
| `git_submodule_foreach` | `repository::for_each_submodule` |
| `git_submodule_free` | `submodule::~submodule` |
| `git_submodule_head_id` | `submodule::head_id` |
| `git_submodule_ignore` | `submodule::ignore_option` |
| `git_submodule_index_id` | `submodule::index_id` |
| `git_submodule_init` | `submodule::init` |
| `git_submodule_location` | `submodule::location_status` |
| `git_submodule_lookup` | `submodule::lookup_submodule` |
| `git_submodule_name` | `submodule::name` |
| `git_submodule_open` | `submodule::open_repository` |
| `git_submodule_owner` | `submodule::owner` |
| `git_submodule_path` | `submodule::path` |
| `git_submodule_reload` | `submodule::reload` |
| `git_submodule_repo_init` | `submodule::initialize_repository` |
| `git_submodule_resolve_url` | `repository::resolve_submodule_url` |
| `git_submodule_set_branch` | `repository::set_submodule_branch` |
| `git_submodule_set_fetch_recurse_submodules` | `repository::set_submodule_fetch_recurse_option` |
| `git_submodule_set_ignore` | `repository::set_submodule_ignore_option` |
| `git_submodule_set_update` | `repository::set_submodule_update_option` |
| `git_submodule_set_url` | `repository::set_submodule_url` |
| `git_submodule_status` | `repository::submodule_status` |
| `git_submodule_sync` | `submodule::sync` |
| `git_submodule_update` | `submodule::update` |
| `git_submodule_update_options_init` | `submodule::update_options::update_options` |
| `git_submodule_update_strategy` | `submodule::get_update_strategy` |
| `git_submodule_url` | `submodule::url` |
| `git_submodule_wd_id` | **Not implemented** |

### tag

| libgit2 | cppgit2:: |
| --- | --- |
| `git_tag_annotation_create` | `repository::create_tag_annotation` |
| `git_tag_create` | `repository::create_tag` |
| `git_tag_create_from_buffer` | `repository::create_tag` |
| `git_tag_create_lightweight` | `repository::create_lightweight_tag` |
| `git_tag_delete` | `repository::delete_tag` |
| `git_tag_dup` | `tag::copy` |
| `git_tag_foreach` | `repository::for_each_tag` |
| `git_tag_free` | `tag::~tag` |
| `git_tag_id` | `tag::id` |
| `git_tag_list` | `repository::tags` |
| `git_tag_list_match` | `repository::tags_that_match` |
| `git_tag_lookup` | `repository::lookup_tag` |
| `git_tag_lookup_prefix` | `repository::lookup_tag` |
| `git_tag_message` | `tag::message` |
| `git_tag_name` | `tag::name` |
| `git_tag_owner` | `tag::owner` |
| `git_tag_peel` | `tag::peel` |
| `git_tag_tagger` | `tag::tagger` |
| `git_tag_target` | `tag::target` |
| `git_tag_target_id` | `tag::target_id` |
| `git_tag_target_type` | `tag::target_type` |

### trace

| libgit2 | cppgit2:: |
| --- | --- |
| `git_trace_set` | **Not implemented** |

### transaction

| libgit2 | cppgit2:: |
| --- | --- |
| `git_transaction_commit` | `transaction::commit` |
| `git_transaction_free` | `transaction::~transaction` |
| `git_transaction_lock_ref` | `transaction::lock_reference` |
| `git_transaction_new` | `repository::create_transaction` |
| `git_transaction_remove` | `transaction::remove_reference` |
| `git_transaction_set_reflog` | `transaction::set_reflog` |
| `git_transaction_set_symbolic_target` | `transaction::set_symbolic_target` |
| `git_transaction_set_target` | `transaction::set_target` |

### tree

| libgit2 | cppgit2:: |
| --- | --- |
| `git_tree_create_updated` | `repository::create_updated_tree` |
| `git_tree_dup` | `tree::copy` |
| `git_tree_entry_byid` | `tree::lookup_entry_by_id` |
| `git_tree_entry_byindex` | `tree::lookup_entry_by_index` |
| `git_tree_entry_byname` | `tree::lookup_entry_by_name` |
| `git_tree_entry_bypath` | `tree::lookup_entry_by_path` |
| `git_tree_entry_cmp` | `tree::entry::compare` |
| `git_tree_entry_dup` | `tree::entry::copy` |
| `git_tree_entry_filemode` | `tree::entry::filemode` |
| `git_tree_entry_filemode_raw` | `tree::entry::raw_filemode` |
| `git_tree_entry_free` | `tree::entry::~entry` |
| `git_tree_entry_id` | `tree::entry::id` |
| `git_tree_entry_name` | `tree::entry::filename` |
| `git_tree_entry_to_object` | `repository::tree_entry_to_object` |
| `git_tree_entry_type` | `tree::entry::type` |
| `git_tree_entrycount` | `tree::size` |
| `git_tree_free` | `tree::~tree` |
| `git_tree_id` | `tree::id` |
| `git_tree_lookup` | `repository::lookup_tree` |
| `git_tree_lookup_prefix` | `repository::lookup_tree` |
| `git_tree_owner` | `tree::owner` |
| `git_tree_walk` | `tree::walk` |


### treebuilder

| libgit2 | cppgit2:: |
| --- | --- |
| `git_treebuilder_clear` | `tree_builder::clear` |
| `git_treebuilder_entrycount` | `tree_builder::size` |
| `git_treebuilder_filter` | `tree_builder::filter` |
| `git_treebuilder_free` | `tree_builder::~tree_builder` |
| `git_treebuilder_get` | `tree_builder::operator[]` |
| `git_treebuilder_insert` | `tree_builder::insert` |
| `git_treebuilder_new` | `tree_builder::tree_builder` |
| `git_treebuilder_remove` | `tree_builder::remove` |
| `git_treebuilder_write` | `tree_builder::write` |
| `git_treebuilder_write_with_buffer` | `tree_builder::write` |

### worktree

| libgit2 | cppgit2:: |
| --- | --- |
| `git_worktree_add` | `repository::add_worktree` |
| `git_worktree_add_options_init` | `worktree::add_options::add_options` |
| `git_worktree_free` | `worktree::~worktree` |
| `git_worktree_is_locked` | `worktree::is_prunable` |
| `git_worktree_is_prunable` | `worktree::is_prunable` |
| `git_worktree_list` | `repository::list_worktrees` |
| `git_worktree_lock` | `worktree::lock` |
| `git_worktree_lookup` | `repository::lookup_worktree` |
| `git_worktree_name` | `worktree::name` |
| `git_worktree_open_from_repository` | `repository::open_worktree` |
| `git_worktree_path` | `worktree::path` |
| `git_worktree_prune` | `worktree::prune` |
| `git_worktree_prune_options_init` | `worktree::prune_options::prune_options` |
| `git_worktree_unlock` | `worktree::unlock` |
| `git_worktree_validate` | `worktree::validate` |

## Contributing

Contributions are welcome, have a look at the [CONTRIBUTING.md](CONTRIBUTING.md) document for more information. If you notice any bugs while using/reviewing `cppgit2`, please report them. Suggestions w.r.t improving the code quality are also welcome.

## License
The project is available under the [MIT](https://opensource.org/licenses/MIT) license.
