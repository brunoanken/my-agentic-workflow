# Code comments

Coding agents add far more comments than code needs. Write fewer, smaller ones.

- **Comment the why, never the what.** The code already says what it does. A comment earns its
  place by recording what the code can't: a business rule, a decision and why it was made, a
  dependency on another system that's easy to miss, a quirk of an API integration. If deleting
  the comment loses nothing a reader couldn't get from the code, don't write it.
- **Put the comment next to the code it explains.** A single line that needs context gets an
  end-of-line comment. A short run of lines gets a short comment directly above it.

  ```ts
  const query = input.trim() // the search API returns 500 on whitespace-only strings
  if (age >= 18) { // state law: minors need guardian consent, which the intake flow handles
  ```

- **Split big blocks.** Don't explain a function in one large header comment. Break it into
  small comments placed at the code paths they describe.
- **A large top-level block is the exception.** Use one only when a reader needs the overall
  picture before the code makes sense, such as a non-obvious algorithm or an invariant that spans
  the whole function.

This covers explanatory comments. API doc comments a project requires (docstrings, JSDoc on
exports) follow the project's convention. Don't strip existing comments from code you aren't
otherwise changing.
