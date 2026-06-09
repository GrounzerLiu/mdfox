# Contributing to MDFox

## How to contribute

1. Fork the repository on GitHub
2. Clone mozilla-central and apply the custom changes (see [BUILD.md](BUILD.md))
3. Create a feature branch from `md-ui-clean`: `git checkout -b my-feature`
4. Make your changes
5. Build and test: `./mach gradle :fenix:assembleDebug`
6. Commit and push to your fork
7. Open a pull request to `md-ui-clean`

## Code style

- **Kotlin**: Follow [Mozilla Kotlin style guide](https://firefox-source-docs.mozilla.org/code-quality/coding-style/index.html)
- **XML**: Use 4-space indentation, follow existing patterns
- **Resources**: Keep strings in `strings.xml`, use `static_strings.xml` only for untranslatable values

## Commit messages

```
<简短描述>

<详细说明（可选）>
```

Write commits in Chinese or English, be descriptive but concise.

## Scope of changes

This project focuses on UI modifications:
- Material Design UI (Compose migration)
- Animation and transitions
- Visual customization

Changes to GeckoView or browser engine internals are out of scope.

## Questions

Open an issue on GitHub or contact the maintainer.
