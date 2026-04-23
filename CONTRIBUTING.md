# 🤝 Contributing to WebMiner

Thanks for wanting to contribute! WebMiner is built with vanilla HTML, CSS, and JavaScript—no frameworks, no build steps, no npm.

## 🐛 Bugs & Feature Requests

Open a [GitHub Issue](../../issues). Please include:
- Your browser and OS.
- Steps to reproduce the bug.
- Console errors (if any).

## 🔒 Security Rules (Strict)

Because WebMiner handles browser compute, security is critical. **We will reject any PR that includes:**
- `eval()`, `new Function()`, or dynamic script injection.
- Access to `document.cookie`, `localStorage`, or `sessionStorage`.
- DOM manipulation from the mining script (WebMiner must run silently).
- Outbound requests to anything other than the user-configured mining pool.

## 💻 Making Changes

1. **Fork** the repo.
2. **Create a branch** (`git checkout -b feature/my-idea`).
3. **Make your changes** (just edit the HTML/JS files).
4. **Test locally** (just open `index.html` in your browser).
5. **Commit** with a clear message.
6. **Open a Pull Request**.

## 🎨 Style Guide

- **Vanilla only.** No jQuery, no React, no frameworks.
- **ES6+ syntax.** Use `const`/`let`, arrow functions, and template literals.
- **Web Workers.** Heavy compute logic goes in workers, never the main thread.
- **Formatting.** 2 spaces, single quotes, no trailing commas.

## 📄 License

By contributing, you agree your code will be licensed under the [MIT License](LICENSE).

For security vulnerabilities, please read [SECURITY.md](SECURITY.md) instead of opening a public issue.
