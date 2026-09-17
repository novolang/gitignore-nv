# gitignore-nv

A gitignore file is a list of patterns that decide which paths a tool
ignores. The format is specified in
[gitignore(5)](https://git-scm.com/docs/gitignore), and the same file
is read by tools that are not git at all — linters, formatters,
bundlers and search tools all honour it. This package answers what
gitignore(5) says about a path, and it never opens a directory.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What the format is

A gitignore file holds one pattern per line. A blank line matches
nothing. A line beginning with `#` is a comment, and a pattern that
really begins with a hash is written `\#`. Trailing spaces are ignored
unless the last one is escaped with a backslash.

Three characters change what a line means.

| Written | Meaning |
| --- | --- |
| `!` at the start | A match **re-includes** the path instead of ignoring it |
| `/` at the end | The pattern matches a **directory** and nothing else |
| `/` anywhere else | The pattern is **anchored**: it is matched against the whole path relative to the file it is in |

Anchoring is the rule that decides most questions. A pattern with no
slash, or with a slash only at the end, is matched against the name of
any one path component, at any depth: `frotz` matches `frotz` and
`a/doc/frotz`. A pattern holding a slash anywhere else is matched
against the whole path relative to the directory its file sits in:
`doc/frotz` matches `doc/frotz` and does not match `a/doc/frotz`. A
leading slash is the same rule from the other side — it puts a slash in
the pattern, so the pattern anchors, and the slash itself is then
dropped.

The wildcards are fnmatch(3) with `FNM_PATHNAME`. `?` is one character
and `*` is any run of characters, and neither of them crosses a `/`.
`**` written as a whole path component crosses any number of
components, including none. `[abc]` and `[a-z]` are character classes,
`[!abc]` is a negated one, and `\` makes the character after it
literal.

Git reads patterns from four places and consults them in this order,
the first that decides winning:

| Order | Source |
| --- | --- |
| 1 | Patterns given on the command line |
| 2 | The gitignore file in the path's own directory, then its parent, up to the top of the working tree |
| 3 | `$GIT_DIR/info/exclude` |
| 4 | The file named by `core.excludesFile` |

Within one file, the **last** matching pattern decides.

## Install

```
novo pkg add gitignore-nv
```

## Example

```novo
use gilist
use gimatcher

fn main() [io]
    // The repository's top-level file: ignore every log, keep one.
    let top = gilist.parse("", "*.log\n!keep.log\nbuild/\n")

    // A file in docs/, whose rules are relative to docs/. It was
    // added first, so it has the higher precedence.
    let docs = gilist.parse("docs", "!build/\n")

    let matcher = gimatcher.add(gimatcher.add(gimatcher.empty(), docs), top)

    // The second argument of each call says whether the path names a
    // directory. Nothing here can go and look.
    println("${gimatcher.is_ignored(matcher, "run.log", false)}")     // true
    println("${gimatcher.is_ignored(matcher, "keep.log", false)}")    // false
    println("${gimatcher.is_ignored(matcher, "build", true)}")        // true
    println("${gimatcher.is_ignored(matcher, "docs/build", true)}")   // false

    // Which source and which line decided, as `git check-ignore -v`
    // prints them.
    let which = gimatcher.deciding_source(matcher, "run.log", false)
    let line = gilist.decision_line(gimatcher.decide(matcher, "run.log", false))
    println("source ${which}, line ${line}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: gitignore-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `gipattern` | One line of a gitignore file: the three flags read off it, what it matches, and the glob pattern it is matched with. |
| `gilist` | One gitignore file, with the directory it was found in, and what it says about a path. |
| `gimatcher` | Several files in precedence order, the ancestor rule, and the question a directory walker asks. |

## How to choose an entry point

**`gimatcher.is_ignored` answers the question a tool actually has.** It
consults every source in order and applies the ancestor rule.

**`gimatcher.decide` answers the same question with the reason.** With
`gimatcher.deciding_source` and `gilist.decision_line` it is what
`git check-ignore -v` prints.

**`gimatcher.decide_alone` asks only about the path.** It skips the
ancestor rule, so it answers "does any rule name this path". A tool
explaining a gitignore file wants it; a tool deciding whether to read a
file does not.

**`gilist.decide` asks one file.** Use it when the rules came from one
place and precedence does not arise.

**`gipattern.matches` asks one rule.** Use it when the rule is the
subject — a linter, a test, an editor showing what a line would do.

## The rules a user needs

1. **Whether a path is a directory is an argument.** Every matching
   function takes `is_dir`, because a pattern ending in `/` matches
   only a directory and this package performs no input or output. A
   caller that guesses disagrees with git on every rule written that
   way.
2. **A path is relative to the root, with `/` as its separator, and
   has no leading slash.** `"src/main.o"`, never `"/src/main.o"` and
   never `"src\\main.o"`.
3. **A slash anywhere except at the end anchors the pattern.** See the
   table above. `gipattern.glob_pattern` shows the translation, so a
   rule that does not do what its author expected can be read rather
   than guessed at. gitignore(5), PATTERN FORMAT.
4. **The last matching line in a file wins.** `*.log` followed by
   `!keep.log` re-includes one file; the same two lines in the other
   order do not. gitignore(5), PATTERN FORMAT.
5. **A file cannot be re-included under an excluded directory.**
   `src/` followed by `!src/keep.log` ignores `src/keep.log`, because
   git never descends into an ignored directory and so never reads the
   second rule. `gimatcher.decide` checks the ancestors first and
   `gimatcher.decide_alone` does not, and the two differ exactly here.
   gitignore(5), PATTERN FORMAT.
6. **A re-inclusion is a decision, not a shrug.** `GiIncluded` stops a
   lower-precedence source from ignoring the path. `GiUnmatched` sends
   the caller on to the next one.
7. **`gimatcher.add` appends a source of lower precedence.** So the
   sources are added in the order the table above lists them: command
   line, then the deepest gitignore file, upwards, then
   `info/exclude`, then the global file.
8. **Every rule is relative to its own file's directory.** A `GiList`
   carries that directory, and a path outside it is `GiUnmatched`. The
   top-level file's base is `""`.
9. **A `*` matches a name beginning with a dot.** A shell's does not.
   This is one flag, and getting it wrong ignores every hidden file in
   a repository or none of them.
10. **Case sensitivity is the repository's, and the caller passes it.**
    `gimatcher.case_insensitive` is `core.ignoreCase`. This package
    reads no configuration file.
11. **A gitignore file has no syntax errors.** `gipattern.parse`
    answers a value for every line and never a `Result`. The one line
    gitignore(5) calls undefined — a rule ending in a single backslash
    — is `GiMalformed` and matches nothing, so a tool can warn about it
    rather than silently doing nothing.
12. **Blank lines and comments are kept in a `GiList`.** A pattern's
    index is its line number less one, which is what a message pointing
    at a rule needs. `gilist.rules_of` is the list without them.
13. **A trailing `\r` is stripped.** A gitignore file written on
    Windows is still a gitignore file.

## What is not included

- **A directory walk.** Nothing here opens a directory or reads a file.
  `gimatcher.should_descend` and `gipattern.may_contain_match` are the
  two questions a walker asks, and both are answered from the patterns
  and the path alone. A walker that uses them lives in the program that
  has the `[fs]` effect.
- **Finding the gitignore files.** A caller reads them and says which
  directory each came from. Locating `$GIT_DIR`, reading
  `core.excludesFile` and honouring `$XDG_CONFIG_HOME` all need the
  filesystem and the environment.
- **`core.ignoreCase`, and every other configuration key.** See rule
  10.
- **`.gitattributes` and sparse-checkout patterns.** They use a related
  but different pathspec syntax, and a matcher that accepted both would
  accept a file that is valid as neither.
- **Attribute macros and `git check-ignore`'s exclude-from option.**
  Those are a program's interface, not the format's.

## Related packages

- [glob-nv](https://novo-lang.org/packages/glob-nv) is the wildcard
  matching underneath, and it is a good deal more useful on its own
  than this package is: a program that wants `*.nv` wants glob-nv. This
  package depends on it.
- [path-nv](https://novo-lang.org/packages/path-nv) normalises and
  joins paths. Use it to produce the relative path this package takes.

## Tests

```bash
novo test tests/gipattern_tests.nv   # the grammar of one line
novo test tests/gilist_tests.nv      # one file, its base, last match wins
novo test tests/gimatcher_tests.nv   # precedence, and the ancestor rule
```

The normative source is gitignore(5), and the reference
implementations are the Rust crate `ignore` and the Python package
`pathspec`. The behaviour every case is checked against is what
`git check-ignore -v` prints for the same path.

The suite asserts that a slash in the middle anchors a pattern and a
slash at the end does not, that a directory-only rule does not match a
file, that the last matching line in a file wins, that a file under an
excluded directory cannot be re-included, that a higher-precedence
source wins outright, and that `*` reaches a name beginning with a dot.

The tests compile today and fail at run, each on the
`not implemented: gitignore-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `gipattern.GiPattern`, `.GiPatternKind`, `gilist.GiList`, `.GiDecision`, `gimatcher.GiMatcher` | the types are declared |
| `gipattern.parse`, `.kind_name`, `.is_rule`, `.has_wildcard` | no |
| `gipattern.matches`, `.may_contain_match` | no |
| `gipattern.glob_pattern`, `.options`, `.ignore_case` | no |
| `gilist.parse`, `.of`, `.rules_of` | no |
| `gilist.decide`, `.may_contain_match` | no |
| `gilist.decision_name`, `.decision_line`, `.is_decided` | no |
| `gimatcher.empty`, `.add`, `.case_insensitive`, `.options_of` | no |
| `gimatcher.decide`, `.decide_alone`, `.is_ignored`, `.deciding_source`, `.matching_rules` | no |
| `gimatcher.should_descend`, `.ancestors_of`, `.under_base` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
