# Contributing

This repository is the specification. Implementations follow it, so a change to the wire starts here, as a pull request against [`CCP.md`](CCP.md), before any implementation changes.

- Open an issue first for anything beyond a correction, so implementers can weigh in.
- Write requirements with [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) keywords in capitals (`MUST`, `SHOULD`, `MAY`), as the [Cashu NUTs](https://github.com/cashubtc/nuts/blob/main/CONTRIBUTING.md) do.
- Add or update test vectors in [`tests/CCP-tests.md`](tests/CCP-tests.md) for any change to what is signed, hashed or serialized.
- A change merges once both the Flash and Foxy maintainers have signed off: the author's side by opening it, the other side by approving it. Every file is owned by both teams ([`.github/CODEOWNERS`](.github/CODEOWNERS)), and `main` takes a pull request with one code owner's approval from someone other than its last pusher.
- Format with Prettier before pushing; CI checks it:

```bash
npx prettier@3.9.1 --write .
```
