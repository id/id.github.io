+++
title = "Emacs on macOS with higher max open files"
date = "2025-08-23"
description = "Raise Emacs' practical file descriptor ceiling on macOS by compiling with FD_SETSIZE and DARWIN_UNLIMITED_SELECT via a custom Homebrew tap. Step‑by‑step, reproducible, and easy to upgrade."
tags = ["emacs", "homebrew", "macos", "fd_setsiz", "select", "build", "tap", "reproducible"]
+++

When Emacs runs lots of subprocesses (LSP servers, file watchers, TRAMP, Magit, terminals), the default macOS descriptor limits (1024) can surface as "Too many open files." A proven mitigation is to compile Emacs with larger descriptor set size and a Darwin-specific define: `-DFD_SETSIZE=10000 -DDARWIN_UNLIMITED_SELECT` [1].

This post shows how to make that build reproducible with a small, local Homebrew tap that carries a lightly patched `emacs` formula.

What you need to do:
- **Create a local tap** to host your customized `emacs` formula.
- **Patch the formula** to append `CFLAGS="-DFD_SETSIZE=10000 -DDARWIN_UNLIMITED_SELECT"` during build [1].
- **Install from your tap** now, and **upgrade later** with the same commands.

Prerequisites:
- Homebrew set up on macOS.
- Command Line Tools for Xcode: `xcode-select --install` (if you haven’t already).

## Create a local tap

Initialize an empty tap repository you control:

```bash
brew tap-new $(id -u)/local
```

## Seed your formula

```
brew extract emacs $(id -u)/local
```

This creates `$(brew --repository)/Library/Taps/$(id -u)/homebrew-local/Formula/emacs.rb` for you to modify.

## Patch the formula to raise FD limits

Open your tap’s `emacs.rb` (you'll need existing emacs installed for this to work, otherwise just find the formula it in the path above and edit it):

```bash
brew edit $(id -u)/local/emacs
```

Inside `def install`, append the `CFLAGS` before `./configure` step (this is how it looks like as of `emacs@30.2`):

```
def install
  # ...
  ENV.append "CFLAGS", "-DFD_SETSIZE=1048576 -DDARWIN_UNLIMITED_SELECT"
  system "./configure", *args
  system "make"
  system "make", "install"
  # ...
end
```

## Install from your tap

```bash
brew install $(id -u)/local/emacs
```

## Verify the build behaves as expected

Launch Emacs and check the max open files:

```elisp
M-! ulimit -n RET
```

# References

[1] [Fix annoying max open files for Emacs](https://en.liujiacai.net/2022/09/03/emacs-maxopenfiles/)
