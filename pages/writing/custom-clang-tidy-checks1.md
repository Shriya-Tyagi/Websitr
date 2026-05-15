# Your Regex Is Not a Parser: A guide to custom clang-tidy checks

*Part 1 of 2. This covers what clang-tidy is, why custom checks exist, what the AST, how to build LLVM, and how to explore and learn the AST in order to write a clang-tidy check. Part 2 covers writing clang-tidy the custom check itself: matchers, diagnostics, fix-its, llvm-LIT tests, and the full linking guide.*
*If you are here to lint your own code rather than write checks for it, this is not that document. That document is: "So you want to lint your code: The Guide."*

---

## Introduction

Clang-tidy is a C++ linter and static analysis tool. It is, structurally, the compiler, doing the parsing and semantic analysis work, and then sending the result to a check instead of to a code generator.

The distinction between a tool that parses C++ and a tool that pattern-matches on C++ text is not cosmetic. It is the difference between a tool that knows that `auto x = vec.begin()` has type `std::vector<int>::iterator` and a tool that sees the string `"auto"` and does its best. Grep does the latter. Clang-tidy does not.

Clang-tidy checks are C++ extensions integrated into the clang-tidy framework that receive the parsed representation of source code — the AST (Abstract syntax tree), covered below — and can inspect it, and emit diagnostics with automatic Fix-it hints that can be applied automatically via `clang-tidy --fix`.

The llvm-project comes with several hundred built-in checks organized into modules: `modernize` for updating old unsafe code to newer safe methods, `readability` for style consistency, `bugprone` for common error patterns (prone: while some cross-TU functionality exists, clang-tidy usually goes translation unit by translation unit) etc. These cover a substantial fraction of what any codebase needs. 

The built-in checks handle general C++. They do not know about internal APIs, a company's specific needs (during my first work term, I implemented and ported checks which helped embedded c++ code comply to standards like Cert C++ and AUTOSAR, for instance), or the deprecated subsystems in your codebase specifically. These are yours to write.


---

## What is the AST

The Abstract Syntax Tree is the data structure Clang builds when it parses a C++ source file. It is a tree of nodes where each node represents a code element.

### The major node categories

Clang's AST has 3 primary node categories that matter for check writing. Everything else is a subcategory. The following is not exhaustive but covers the most common categories.

**Declarations (`Decl`).** A `Decl` node represents something that introduces a name. Function declarations, variable declarations, class definitions, template declarations, namespace definitions etc. The hierarchy under `Decl` is extensive. The ones encountered most frequently:

- `VarDecl` — a variable declaration; carries type, storage class, initializer
- `FunctionDecl` — a function or method declaration, including its body if defined
- `CXXMethodDecl` — a method on a class; subclass of `FunctionDecl`
- `CXXConstructorDecl` / `CXXDestructorDecl` — constructor and destructor; subclasses of `CXXMethodDecl`
- `CXXRecordDecl` — a class, struct, or union definition
- `NamespaceDecl` — a namespace definition

Every `Decl` node carries source location information — where in the file it appears — and access to the `DeclContext` it lives in (the enclosing scope it is inside).

**Statements (`Stmt`).** A `Stmt` node represents an executable unit. The common ones:

- `CompoundStmt` — a brace-enclosed block `{ ... }`
- `IfStmt` — an if statement with condition, then-branch, optional else-branch
- `ForStmt` / `WhileStmt` / `DoStmt` — loop constructs
- `ReturnStmt` — a return statement with optional return value

`Expr` is a subclass of `Stmt` that produces a value and has a type. This is the category with the most nodes, and the category checks interact with most heavily.

- `CallExpr` — a function call; carries the callee and arguments
- `CXXMemberCallExpr` — a method call on an object; subclass of `CallExpr`
- `MemberExpr` — access to a member of an object (`obj.field`, `ptr->field`)
- `DeclRefExpr` — a reference to a previously declared name (a variable, function etc.)
- `BinaryOperator` — a binary expression with an opcode (`+`, `-`, `==`, `&&`)
- `UnaryOperator` — a unary expression (`!`, `*`, `&`, `++`)
- `CXXNewExpr` / `CXXDeleteExpr` — `new` and `delete` expressions
- `CXXConstructExpr` — a constructor call (not always visible syntactically)

The `ImplicitCastExpr` node deserves special mention because it is invisible in syntax code and present everywhere in the AST and needs to be stripped often when searching for a specific expr. When the compiler inserts an implicit conversion it wraps the original expression in an `ImplicitCastExpr`. Matchers that do not account for this will fail silently with no explanation.
In addition to implicit casts. there are also paranthesis casts that may be the reason a matcher misses cases. The helper `IgnoreParenImpCasts()` fixes this issue.

**Types** They are not nodes in the tree in the same way, they are a parallel structure that Decl/Stmt nodes refer to. Every VarDecl/Expr has a QualType. QualType is the wrapper Clang uses for everything type-related. It pairs a Type pointer with qualifiers — const, volatile. Most type-related matchers match via QualType.

- `BuiltinType` — int, float, bool, void etc.
- `PointerType` — a pointer type also has the pointee type
- `RecordType` — a class, struct, or union type linked with CXXRecordDecl
- `AutoType` — an auto-typed declaration before or after deduction

Check code rarely inspects type nodes directly. The matchers handle most type queries — hasType(), pointsTo(), references(), isInteger(). Direct type inspection becomes relevant when the matcher system cannot express the condition cleanly, and then check() can queries the QualType directly via the AST context.

[For the complete list of nodes:](https://clang.llvm.org/doxygen/namespaceclang.html)

### How nodes connect

The tree is a parent-child relationship. A `FunctionDecl` contains a `CompoundStmt` as its body. The `CompoundStmt` contains a sequence of `Stmt` children — the statements in the function body. Any of these can contain expressions, which can in turn contain subexpressiosn.

Example:

```cpp
int add(int a, int b) {
    return a + b;
}
```
The AST for this (simplified):

```
FunctionDecl 'add' -> returns 'int'
├── ParmVarDecl 'a' -> type 'int'
├── ParmVarDecl 'b' -> type 'int'
└── CompoundStmt
    └── ReturnStmt
        └── BinaryOperator '+'
            ├── ImplicitCastExpr -> LValueToRValue
            │   └── DeclRefExpr 'a'
            └── ImplicitCastExpr -> LValueToRValue
                └── DeclRefExpr 'b'
```

### To see the actual AST for any piece of code:

```bash
clang -Xclang -ast-dump -fsyntax-only YourFile.cpp
```
(Guide below on how to build llvm-porject to run this.)

Do this before writing any check for the specific pattern the check needs to match. The tree structure of the code being targeted is not always easy to guess, and source-code pattern matching intuition does not transfer cleanly to ASTs. One time I spent hours debugging why a bad AST node was not throwing. The query quietly returned 0 instead, and the matcher failed silently downstream.

Every node has information. For a `CallExpr`, this may include:

- The callee expression (what is being called)
- The callee declaration (the `FunctionDecl` being called, with full type information)
- The argument expressions, in order
- The return type of fun
- The source range of the entire expression
- Whether the call is on a dependent type (relevant for templates).

A `CXXMemberCallExpr` additionally carries:

- The implicit object argument (the object the method is being called on)
- The `CXXMethodDecl` being called
- The `CXXRecordDecl` of the class the method belongs to.

This is semantically complete information available for every node type. The types are already known. The check code does not parse anything. It queries the ast data structure built.

*[For an excellent and more thorough look at the AST:](https://clang.llvm.org/docs/IntroductionToTheClangAST.html) (This tutorial uses clang-query. Building llvm as shown below may help follow the tutorial's clang-query examples better.)*

---

## Building LLVM

A local LLVM build is required to write custom checks. This section covers getting the build into a state where incremental rebuilds are fast.

### Apt/brew install requirements

Install (via apt/brew etc.) the following:
- CMake
- Ninja (can use Unix Makefiles but this is faster)
- Clang/gcc — Using Clang to build Clang is supported
- lld — the LLVM linker. Not strictly required, but my default linker was `ld`, and it was much slower.

### Git clone llvm-project

```bash
git clone https://github.com/llvm/llvm-project.git
cd llvm-project
```

The full respository takes a long time. To write downstream checks rather than contributing to LLVM, a shallow clone is sufficient and fast, `git clone --depth=1 https://github.com/llvm/llvm-project.git`.


### Repo structure

The LLVM monorepo contains a lot of code that is irrelevant to writing clang-tidy checks. The following is the relevant portion.

```
llvm-project/
├── clang/
│   ├── include/clang/
│   │   ├── AST/                    //Header files defining every AST node type
│   │   │   ├── Decl.h              //Decl node types
│   │   │   ├── Stmt.h              //Stmt node types
│   │   │   ├── Expr.h              //Expr node types
│   │   │   ├── Type.h              //Type system
│   │   │   └── ASTContext.h        //AST context: type creation, global info
│   │   └── ASTMatchers/
│   │       └── ASTMatchers.h       //Complete list of available AST matchers. When searching for a matcher pattern, this file can be grepped.
│   └── lib/
│       └── ASTMatchers/
│           └── ASTMatchFinder.cpp  # Matcher engine implementation
├── clang-tools-extra/
│   ├── clang-tidy/                 //Check modules live here
│   │   ├── ClangTidy.h             //Core check interface
│   │   ├── ClangTidyCheck.h        //Base class for all checks
│   │   ├── ClangTidyModule.h       //Module registration interface
│   │   ├── abseil/                 //Modules
│   │   ├── bugprone/
│   │   ├── cert/  
│   │   ├── modernize/
│   │   ├── readability/
│   │   └── YOUR_MODULE/            //A new module goes here
│   │       ├── CMakeLists.txt
│   │       ├── YourModuleTidyModule.h
│   │       ├── YourModuleTidyModule.cpp
│   │       ├── YourSpecificCheck.h
│   │       └── YourSpecificCheck.cpp
│   └── test/clang-tidy/
│       └── checkers/               //Lit tests to verify each check
│           ├── bugprone/
│           ├── cert/
│           └── YOUR_MODULE/ 
│               └── your-specific-check.cpp
└── llvm/
    └── include/llvm/
        ├── Support/
        └── ADT/                    //Data structures (StringRef, SmallVector, etc.).
```


### Configuring the build

Inside llvm-project:

```bash
Mkdir build
cd build
cmake -S ../llvm -B . \
  -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra" \
  -DLLVM_USE_LINKER=lld \
  -DLLVM_TARGETS_TO_BUILD=Native \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

`BUILD_TYPE=Release` — builds with optimizations enabled. `Debug`, produces binaries that are very slow but make LLVM debugging easier. `RelWithDebInfo` enables optimizations while retaining debug symbols. `ENABLE_ASSERTIONS=ON` — keeps runtime assertions in Release build. Not needed with Debug.

If not using ninja as your build system, replace Ninja with "Unix Makefiles".

Enabled projects are only clang and clang-tools-extra, the 2 needed for clang tidy.

If not using lld, USE_LINKER flag can be omitted and default linker will be used.

`TARGETS_TO_BUILD=Native` builds only for your machine, the default builds backends for all supported architectures.


### Building

```bash
ninja clang-tidy

./build/bin/clang-tidy --version    #verify it build the binary

```

Add -j8 or -j16 depending on cores increases parallelism. More neccessary with makefiles, ninja is pretty fast by itself. First builds take ~1.5h. Following incremental builds after modifying a single check file are 1min to 15 mins.


### Common build failures and their causes

**CMake cannot find lld.** Either lld is not installed, or it is installed under a different name (e.g., `lld-14` on systems with multiple LLVM versions). Install lld, or remove `-DLLVM_USE_LINKER=lld` from the CMake invocation and accept slower link times.

**`clang` not found.** CMake needs a working C++ compiler to configure the build. If the system compiler is GCC and CMake cannot find it, set `CC` and `CXX` explicitly: `CC=gcc CXX=g++ cmake ...`.


---

## Exploring the AST for your code

Before writing a matcher, we need to see what the AST actually looks like for the code being targeted. Find the relevant nodes, note their types and the information they carry to effectively write a matcher. There are two tools for this.

### clang-ast-dump

The faster option:

```bash
clang -Xclang -ast-dump -fsyntax-only -std=c++17 YourFile.cpp
```

This dumps the entire AST for the file to stdout. The output is a tree with indentation indicating nesting level. It is verbose. For a non-trivial file, it is very verbose. Redirect to a file and search.

```bash
clang -Xclang -ast-dump -fsyntax-only -std=c++17 YourFile.cpp > ast.txt
```

The dump format uses consistent node type names that correspond directly to the C++ class names in the AST headers. A node labeled `CXXMemberCallExpr` in the dump is a `CXXMemberCallExpr` in the C++ API. The names transfer directly.

### clang-query

Clang-query is an interactive tool for writing and testing AST matchers against real code. It is the correct tool for developing matchers before putting them in a check.

To build:

```bash
ninja clang-query
```

To run it on a file:

```bash
./build/bin/clang-query YourFile.cpp --
```

The `--` at the end separates the source file from compiler flags. If the file requires specific flags, they're added at the end.

```bash
./build/bin/clang-query YourFile.cpp -- -std=c++17 -I/path/to/includes
```

Example: Explore AST to write specific code's matcher:

Inside the clang-query REPL:

```
clang-query> match callExpr()
```

This matches every function call in the file and prints each match with its source location. Refine the matcher:

```
clang-query> match callExpr(callee(functionDecl(hasName("valueFinderFunc"))))
```

Now it matches only calls to functions named `valueFinderFunc`. Add more constraints:

```
clang-query> match cxxMemberCallExpr(
  callee(cxxMethodDecl(hasName("valueFinderFunc"))),
  on(hasType(cxxRecordDecl(hasName("vector"))))
)
```

The matcher iterations can be tested against the actual AST of the actual file before the check code is written. The set of available matchers is at [clang.llvm.org/docs/LibASTMatchersReference.html](https://clang.llvm.org/docs/LibASTMatchersReference.html) and the full signatures in `ASTMatchers.h`. 


---

* Part 2 covers writing clang-tidy the custom check itself: `registerMatchers()` and `check()` in detail, debugging, diagnostics, fix-its, llvm-LIT tests, and the full linking guide.*++++
