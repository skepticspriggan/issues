# [feat: show zoom percentage in statusbar resolving #2870](https://github.com/qutebrowser/qutebrowser/pull/8740)

- _Type:_ Pull Request
- _State:_ open
- _Repository:_ `qutebrowser`
- _Created at:_ 2025-10-21 08:30:34

Show zoom percentage in statusbar resolving #2870.

- The zoom widget uses `TextBase`.
- A new zoom factor changed signal was introduced to update the percentage after each change.
- Unit tests were added and passed succesfully.

## Comments

skepticspriggan on 2025-11-06 20:36

> Glad to help out.

I had some trouble satisfying the mypy-pyqt{5,6} linters. First I ignored the undefined attribute error and this worked in mypy-pyqt5, however it caused a redundant ignore error in mypy-pyqt6.

```python
try:
    page.zoomFactorChanged.connect(self.factor_changed)  # type: ignore[attr-defined]
except AttributeError:
    # Added in Qt 6.8
    pass
```

I solved it by adding a conditional. Not as clean as I would like. Is there a better way or is this a necessary evil?

skepticspriggan on 2025-11-07 09:39

> Thanks, that is cleaner and all linters passed.

skepticspriggan on 2025-11-07 09:46

> I see, the argument `--always-false=IS_QT6` in `mypy-pyqt5` ensures that code block for QT6 is not checked. Good to know.

The-Compiler on 2025-11-07 10:23

> Yup, indeed! Qt 5 support will be dropped soon-ish anyways though, see #8417. Hopefully that will make those kind of things a fair bit easier.

skepticspriggan on 2025-11-07 23:13

> Thanks! I do not mind the iterations, part of development. Especially since it is a pleasure to work in a neat codebase. Great work.
