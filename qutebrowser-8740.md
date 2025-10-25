# [feat: show zoom percentage in statusbar resolving #2870](https://github.com/qutebrowser/qutebrowser/pull/8740)

- _Type:_ Pull Request
- _State:_ open
- _Repository:_ `qutebrowser`
- _Created at:_ 2025-10-21 08:30:34

Show zoom percentage in statusbar resolving #2870.

- The zoom widget uses `TextBase`.
- A new zoom factor changed signal was introduced to update the percentage after each change.
- Unit tests were added and passed succesfully.
