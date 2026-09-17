# Changelog

All notable changes to gitignore-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `gipattern` — one line of a gitignore file, with the three flags
  gitignore(5) reads off it: `!` negation, a trailing slash for
  directory-only, and a slash anywhere else for anchoring. `parse`
  answers a value for every line and never a `Result`, because git
  refuses no line; the one line gitignore(5) calls undefined is
  `GiMalformed` and says so. `glob_pattern` and `options` show exactly
  what is handed to glob-nv, so the translation is inspectable.
- `gilist` — one file with the directory it was found in, because a
  rule without its base is a rule nobody can apply. The last matching
  line decides, so the scan runs to the end, and the decision carries
  the line number `git check-ignore -v` prints. Blanks and comments are
  kept so that a pattern's index is its line number less one.
- `gimatcher` — the four sources in gitignore(5)'s order, with `add`
  appending a source of lower precedence so a caller never reasons
  about the direction. `decide` checks the path's ancestors first,
  which is gitignore(5)'s rule that a file under an excluded directory
  cannot be re-included; `decide_alone` is the other question, kept
  separate because the answers differ.
- No directory is ever opened. Whether a path is a directory is an
  argument, and `should_descend` and `may_contain_match` answer a
  walker's questions from the patterns and the path alone.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  gitignore-nv.<module>.<fn>`.
- glob-nv supplies the wildcard half and nothing else. Everything
  gitignore adds — negation, anchoring, directory-only, several bases,
  precedence — is carried here, because glob has no notion of any of
  it.
