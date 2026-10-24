# Contributing

This repository is the specification. Implementations follow it, so a change to the wire starts here, as a pull request against [`CCP.md`](CCP.md), before any implementation changes.

- Open an issue first for anything beyond a correction, so implementers can weigh in.
- Write requirements with [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) keywords in capitals (`MUST`, `SHOULD`, `MAY`), as the [Cashu NUTs](https://github.com/cashubtc/nuts/blob/main/CONTRIBUTING.md) do.
- Add or update test vectors in [`tests/CCP-tests.md`](tests/CCP-tests.md) for any change to what is signed, hashed or serialized.
- A change to the core needs review from maintainers of at least two implementations.
- Format with Prettier before pushing; CI checks it:

```bash
npx prettier@3.9.1 --write .
```
