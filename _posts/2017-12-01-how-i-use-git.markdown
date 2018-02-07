---
title: How I Use Git
layout: post
---

## Aliases

Explain that I use a lot of aliases. Explain that I use "shell" aliases _not_
"Git" aliases.

## Git Add Patch

``` sh
git add --patch
```

### Git Add Patch Aliases

## Git Commit

``` sh
$GIT_EDITOR || $EDITOR
git commit
```

## Git Commit Fixup

Talk about how this can be used to help reviewing a PR. Don't force push until _just_ before merge.

``` sh
git add --patch
git commit --fixup $SHA

# https://github.com/jasoncodes/dotfiles/blob/f7924cc3d04e617d2f4c1d1995e00e8864ef41c8/shell/aliases/git.sh#L184-L247
gcf # (function)
```

## Git Log

```
gl
```

## Commit Messages

Tpope (but a bit more lax, subject matches the GH length limit)
