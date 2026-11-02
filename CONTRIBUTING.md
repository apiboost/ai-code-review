# Contributing to Apiboost AI Code Review

Thanks for your interest in improving **Apiboost AI Code Review**. Contributions are welcome. This
workflow guards secrets for every repository that calls it, so changes get a careful review.

If you plan a significant change (a new engine, new inputs, or a change to the security model), please
**open an issue first** to discuss it.

## Pull request process

1. Fork the repository and create a topic branch from `main`.
2. Keep commits focused, and use **[Conventional Commits](https://www.conventionalcommits.org/)**
   (`type(scope): summary`). A change to an input, output, default or the security behaviour is breaking:
   mark it with `!` or a `BREAKING CHANGE:` footer.
3. Make sure `actionlint` and `zizmor --offline --persona=auditor .github/workflows` pass. CI runs both.
4. Update `README.md` for any change to inputs, secrets, setup or the threat model.
5. Sign off every commit (see below), and fill in the pull request template.

## Developer Certificate of Origin (DCO) sign-off

All contributions are made under the [Developer Certificate of Origin](./DEVELOPER_CERTIFICATE_OF_ORIGIN.md).
Add a `Signed-off-by` line to every commit, using your real name and an email address you can be reached at:

```
Signed-off-by: Jane Developer <jane@example.com>
```

`git commit -s` adds it for you. Pull requests with unsigned commits can't be merged.

## AI-assisted contributions

AI-assisted contributions are welcome. You are responsible for understanding, testing and having the right
to submit everything in your pull request. If AI tools materially contributed, add an `Assisted-by:` trailer
naming the **exact model and version** to each affected commit, and say so in the pull request:

```
Assisted-by: <model name and version>
Signed-off-by: Jane Developer <jane@example.com>
```

## Security issues

Don't report vulnerabilities in public issues or pull requests. See [SECURITY.md](./SECURITY.md).

## License

By contributing, you agree that your contributions are licensed under the [Apache License 2.0](./LICENSE).
