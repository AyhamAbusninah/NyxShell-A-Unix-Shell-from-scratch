<div align="center">

# NyxShell

**A production-style Unix command runtime implemented from scratch in C**

![C](https://img.shields.io/badge/C-00599C?style=for-the-badge&logo=c&logoColor=white)
![Shell](https://img.shields.io/badge/Shell-FFD500?style=for-the-badge&logo=gnubash&logoColor=black)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![macOS](https://img.shields.io/badge/macOS-000000?style=for-the-badge&logo=apple&logoColor=white)

</div>

---

NyxShell is a fully functional Unix shell implemented in ~4,000 lines of C with zero external dependencies beyond POSIX and GNU Readline. It tokenizes, parses into an AST, expands variables, preprocesses heredocs, and walks the tree to fork/exec pipelines — handling file descriptor ownership, signal semantics, and memory lifecycle manually at every stage.

---

## Systems Engineering Focus

- **This is not a wrapper around `system()`.** NyxShell manages the full process lifecycle — `fork`, `execve`, `waitpid`, `pipe`, `dup2` — with explicit control over every file descriptor and child process.
- **Memory safety without a runtime.** All allocations are tracked and freed across a multi-stage pipeline (lexer → parser → expander → executor), including error paths and signal interrupts. Verified leak-free under Valgrind.
- **Concurrency and IPC.** Pipeline execution creates process trees with coordinated pipe plumbing, correct close semantics, and proper wait ordering to propagate exit codes.
- **Signal correctness.** Interactive vs. non-interactive signal disposition is handled per POSIX, including edge cases around heredoc interruption and foreground process group management.

This project demonstrates the ability to design, implement, and debug non-trivial systems software in C under real constraints.

---

## Features

| Category | Details |
|---|---|
| **Pipelines** | Arbitrary-length command chains with correct fd plumbing and exit-status propagation |
| **Redirections** | Input `<`, output `>`, append `>>`, heredoc `<<` with delimiter and optional expansion |
| **Logical Operators** | `&&` and `||` with short-circuit evaluation based on exit codes |
| **Variable Expansion** | `$VAR`, `$?` (last exit status), with correct expansion inside double quotes |
| **Quote Semantics** | Single quotes preserve literals; double quotes allow expansion — matching Bash behaviour |
| **Builtins** | `echo -n`, `cd`, `pwd`, `export`, `unset`, `env`, `exit` — executed in-process without forking |
| **Signal Handling** | `Ctrl-C` (SIGINT), `Ctrl-\` (SIGQUIT), `Ctrl-D` (EOF) — correct in interactive and child contexts |
| **Error Handling** | Descriptive diagnostics to stderr with POSIX-conformant exit codes |

---

## Architecture

Each input line flows through five discrete stages. No stage has knowledge of the stages that follow it — data flows forward through well-defined intermediate representations.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    readline / input loop                     │
  └────────────────────────────┬─────────────────────────────────┘
                               │  raw input string
                               ▼
                    ┌─────────────────────┐
                    │     TOKENIZER       │  Lexical analysis: splits input
                    │     (Lexer)         │  into typed tokens (WORD, PIPE,
                    │                     │  REDIR, AND, OR, etc.)
                    └──────────┬──────────┘
                               │  token linked list
                               ▼
                    ┌─────────────────────┐
                    │      PARSER         │  Recursive-descent construction
                    │    (AST Builder)    │  of a binary tree with CMD, PIPE,
                    │                     │  AND, and OR node types
                    └──────────┬──────────┘
                               │  abstract syntax tree
                               ▼
                    ┌─────────────────────┐
                    │     EXPANDER        │  Resolves $VAR / $?, strips
                    │                     │  quotes, produces final argv
                    └──────────┬──────────┘
                               │  expanded AST
                               ▼
                    ┌─────────────────────┐
                    │   HEREDOC HANDLER   │  Reads heredoc input before
                    │                     │  execution; expands variables
                    │                     │  when delimiter is unquoted
                    └──────────┬──────────┘
                               │  AST with heredoc fds
                               ▼
                    ┌─────────────────────┐
                    │     EXECUTOR        │  Recursive AST walk: forks
                    │                     │  processes, wires pipes/redirs,
                    │                     │  runs builtins in-process
                    └─────────────────────┘
```

### Source Layout

```
src/
├── main.c                 # REPL loop, readline integration, history
├── tokenizer/             # Lexical analysis — 6 source files
├── parsing/               # AST construction — 8 source files
├── expander/              # Variable expansion & quote removal — 6 source files
├── heredoc/               # Heredoc preprocessing — 2 source files
├── execute/               # Process execution & pipeline management — 8 source files
├── built_in/              # Builtin commands — 8 source files
├── signals/               # Signal handler installation & dispatch
├── libft/                 # Minimal C standard library subset (no glibc extensions)
include/
├── minishell.h            # Core types, AST node definitions, shell state
├── tokenizer.h            # Token types, lexer interface
├── parser.h               # Parser interface, AST node constructors
├── execute.h              # Executor interface, pipe/redir helpers
└── built_in.h             # Builtin dispatch table
```

---

## Implementation Highlights

- **AST-driven execution.** The executor performs a recursive post-order walk of a binary tree. PIPE nodes fork left and right children with a shared pipe; AND/OR nodes short-circuit based on the left child's exit status. CMD leaf nodes apply redirections and either dispatch to a builtin or call `execve`.

- **Pipeline fd ownership discipline.** Each pipe's read and write ends are opened in the parent, duplicated into the appropriate child via `dup2`, and immediately closed in both parent and child after duplication. No file descriptor is left dangling — verified through systematic auditing of every `pipe()`/`dup2()`/`close()` call path.

- **Heredoc preprocessing before execution.** All heredocs in the AST are collected and read *before* any process is forked. Input is stored in a pipe fd attached to the relevant AST node. This avoids interleaving user-facing heredoc prompts with concurrent child process output.

- **Signal model with context switching.** In interactive mode, `SIGINT` triggers a prompt redisplay via `rl_replace_line` / `rl_redisplay` and `SIGQUIT` is ignored. During child execution, signals are reset to default disposition in the child and the parent blocks/unblocks appropriately to avoid race conditions. Heredoc input handles `SIGINT` by discarding the partial document and returning control to the main loop.

- **Environment as a linked list with copy-on-write semantics.** The environment is maintained as a singly linked list cloned from `envp` at startup. `export` and `unset` modify the list in place; the list is serialized to a `char**` array only when `execve` requires it, avoiding unnecessary allocations on every command.

- **Recursive-descent parser with explicit precedence.** The parser implements operator precedence through mutually recursive functions: `parse_or` → `parse_and` → `parse_pipe` → `parse_cmd`. This naturally produces correct associativity and precedence without a table-driven approach.

- **Quote-aware tokenization in a single pass.** The lexer tracks quote state (none / single / double) as it scans, toggling on quote characters and handling nested semantics. Tokens are emitted with quote metadata preserved so the expander can decide where variable expansion applies.

- **Robust cleanup on all exit paths.** Every allocation is reachable from the shell's top-level state struct. On error, signal interruption, or normal exit, a single teardown pass frees the token list, AST, expanded argv arrays, and any open heredoc fds. Validated with `valgrind --leak-check=full --track-fds=yes`.

---

## Build & Run

### Prerequisites

| Requirement | Notes |
|---|---|
| **C compiler** | `gcc` or `clang` with C99 support |
| **GNU Make** | Build orchestration |
| **GNU Readline** | Line editing, history, signal-safe prompt redisplay |

### Installing Readline

**Debian / Ubuntu:**
```bash
sudo apt-get install libreadline-dev
```

**macOS (Homebrew):**
```bash
brew install readline
```

> On macOS, you may need to pass Readline's path to the compiler if Homebrew installs it in a non-default location. The Makefile handles this automatically if `brew --prefix readline` is available.

### Compile & Launch

```bash
git clone https://github.com/AyhamAbusninah/NyxShell.git
cd NyxShell
make
./nyxshell
```

| Make Target | Description |
|---|---|
| `make` | Build the project (compiles libft, then NyxShell) |
| `make clean` | Remove object files |
| `make fclean` | Remove object files and the binary |
| `make re` | Full clean rebuild |

### Example Session

```bash
NyxShell$ echo "hello world" | wc -w
2
NyxShell$ ls -la > out.txt && cat out.txt | head -5
total 48
drwxr-xr-x  5 user user 4096 Jan 12 09:30 .
drwxr-xr-x 11 user user 4096 Jan 12 09:28 ..
-rw-r--r--  1 user user 1247 Jan 12 09:30 Makefile
-rw-r--r--  1 user user 3891 Jan 12 09:30 README.md
NyxShell$ cat << EOF
> line one
> $USER says hello
> EOF
line one
ayham says hello
NyxShell$ false || echo "fallback executed"
fallback executed
NyxShell$ exit
```

---

## Future Work

- **Job control** — `bg`, `fg`, `jobs`, and process group management with `SIGTSTP`/`SIGCONT`.
- **Wildcard expansion** — glob matching (`*`, `?`, `[...]`) resolved before execution.
- **Subshells** — parenthesized command groups `(cmd1 && cmd2)` executing in a child process.
- **Test suite & fuzzing** — property-based tests against Bash output, plus AFL/libFuzzer harness on the parser to catch edge cases.
- **Command substitution** — `$(cmd)` and backtick expansion.
- **Tilde expansion** — `~` and `~user` resolved from the passwd database.

---

## 🎨 UI & UX Features

NyxShell includes a set of visual enhancements designed to improve readability and provide a professional interactive experience, without altering any core logic.

### Custom Color-Coded Prompt

The prompt displays the current **user**, shell name, and **working directory** using distinct ANSI colors — cyan for the username, green for `NyxShell`, bold blue for the directory, and yellow for the `$` indicator. All color escape sequences are wrapped in `\001`/`\002` (readline ignore markers) to prevent cursor miscalculation and line-wrapping glitches.

```
user@NyxShell:projects$ 
```

### Startup ASCII Banner

On launch, NyxShell displays a compact ASCII art logo with a `>>> System Ready <<<` indicator, giving immediate visual confirmation that the shell initialized successfully.

### Colorized Error Output

All error messages — from syntax errors to execution failures — are prefixed with a bold red `NyxShell:` tag, followed by the actual diagnostic in the default terminal color. This makes errors instantly distinguishable from normal command output without modifying error logic or exit codes.

### Centralized Color Definitions

All ANSI codes are managed through a single `colors.h` header using semantic macro names (`COLOR_ERR`, `COLOR_PRMPT_USER`, `COLOR_PRMPT_DIR`, etc.), making it trivial to retheme the entire shell by editing one file.

---

## Engineering Context

NyxShell was built as part of a deeper focus on low-level systems engineering:
process models, memory behavior, concurrency, and distributed runtime design.

It serves as a foundation for more advanced work in:
- execution engines
- distributed compute runtimes
- systems infrastructure for large-scale computation
 
---

## Authors

| | Name | GitHub |
|---|---|---|
| 🛠 | **Ayham Abusnineh** | [github.com/AyhamAbusninah](https://github.com/AyhamAbusninah) |
| 🛠 | **Jihad Aljubeh** | [github.com/jihad7aljubeh](https://github.com/jihad7aljubeh) |

---

<div align="center">

*NyxShell — because understanding systems means building them.*

</div>
