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
>
> I had some trouble satisfying the mypy-pyqt{5,6} linters. First I ignored the undefined attribute error and this worked in mypy-pyqt5, however it caused a redundant ignore error in mypy-pyqt6.
>
> ```python
> try:
>     page.zoomFactorChanged.connect(self.factor_changed)  # type: ignore[attr-defined]
> except AttributeError:
>     # Added in Qt 6.8
>     pass
> ```
>
> I solved it by adding a conditional. Not as clean as I would like. Is there a better way or is this a necessary evil?

skepticspriggan on 2025-11-07 09:39

> Thanks, that is cleaner and all linters passed.

skepticspriggan on 2025-11-07 09:46

> I see, the argument `--always-false=IS_QT6` in `mypy-pyqt5` ensures that code block for QT6 is not checked. Good to know.

The-Compiler on 2025-11-07 10:23

> Yup, indeed! Qt 5 support will be dropped soon-ish anyways though, see #8417. Hopefully that will make those kind of things a fair bit easier.

## Reviews

The-Compiler on 2025-11-04 13:29

> Nice work overall with tests and such, thanks! Two small changes to make things more robust hopefully.

The-Compiler on 2025-11-06 20:47

> This should work better I think?

The-Compiler on 2025-11-07 10:34

> Some more considerations regarding functionality:
>
> - Should `zoom` be added to the default `statusbar.widgets` (probably between `url` and `scroll`)? My vote would be yes, for discoverability and such.
> - Should the widget hide itself if the zoom is 100%? Either with a config option, or by default? IMHO yes, especially if enabled by default.
> - (Perhaps something for outside of this PR): Should the commands in `qutebrowser/components/zoomcommands.py` and `TabEventFilter._handle_wheel` in `qutebrowser/browser/eventfilter.py` stop showing messages? For the commands it's easy to rebind them to pass `--quiet`, but for the mouse wheel right now it's not possible to hide them. Maybe at least the mouse wheel one could be hidden by default if the `zoom` widget is configured to show?

## Review Comments

The-Compiler on 2025-11-04 11:17

qutebrowser/mainwindow/statusbar/zoom.py

```
@@ -0,0 +1,26 @@
+"""Zoom percentage displayed in the statusbar."""
+
+from qutebrowser.browser import browsertab
+from qutebrowser.mainwindow.statusbar import textbase
+from qutebrowser.qt.core import pyqtSlot, QObject
+
+
+class Zoom(textbase.TextBase):
+
+    """Shows zoom percentage in current tab."""
+
+    def __init__(self, parent: QObject = None) -> None:
+        super().__init__(parent)
+        self.setText("100%")
+
+    @pyqtSlot(float)
+    def on_zoom_changed(self, factor: float) -> None:
+        """Update percentage when factor changed."""
+        percentage = int(100 * factor)
+        self.setText(f"{percentage}%")
+
+    def on_tab_changed(self, tab: browsertab.AbstractTab) -> None:
+        """Update percentage when tab changed."""
+        percentage = int(100 * tab.zoom.factor())
+
+        self.setText(f"{percentage}%")
```

> ```suggestion
>         self.on_zoom_changed(tab.zoom.factor())
> ```
>
> to avoid those two getting out of sync if ever changing something in the future.

The-Compiler on 2025-11-04 13:24

qutebrowser/browser/webengine/webenginetab.py

```
@@ -730,6 +730,7 @@ class WebEngineZoom(browsertab.AbstractZoom):

     def _set_factor_internal(self, factor):
         self._widget.setZoomFactor(factor)
+        self.factor_changed.emit(factor)
```

> To avoid things getting out of sync when the zoom factor is changed elsewhere (I think e.g. QtWebEngine handles pinch zooming? Didn't try though...), I think it would be good to connect to the [existing `zoomFactorChanged` signal](https://doc.qt.io/qt-6/qwebenginepage.html#zoomFactorChanged) from `self._widget.page()`. See e.g. `WebEnginePrinting.connect_signals` for some inspiration. You can probably hook things up in a place like that as well and then call it in `WebEngineTab.connect_signals`.
>
> However, that signal was only added in Qt 6.8, so you should most likely connect it with a `except AttributeError:` and still fall back to manual emitting here if it doesn't exist (possibly using something like `if not hasattr(self._widget.page(), "zoomFactorChanged"):` or somesuch).

skepticspriggan on 2025-11-06 20:01

qutebrowser/mainwindow/statusbar/zoom.py

```
@@ -0,0 +1,26 @@
+"""Zoom percentage displayed in the statusbar."""
+
+from qutebrowser.browser import browsertab
+from qutebrowser.mainwindow.statusbar import textbase
+from qutebrowser.qt.core import pyqtSlot, QObject
+
+
+class Zoom(textbase.TextBase):
+
+    """Shows zoom percentage in current tab."""
+
+    def __init__(self, parent: QObject = None) -> None:
+        super().__init__(parent)
+        self.setText("100%")
+
+    @pyqtSlot(float)
+    def on_zoom_changed(self, factor: float) -> None:
+        """Update percentage when factor changed."""
+        percentage = int(100 * factor)
+        self.setText(f"{percentage}%")
+
+    def on_tab_changed(self, tab: browsertab.AbstractTab) -> None:
+        """Update percentage when tab changed."""
+        percentage = int(100 * tab.zoom.factor())
+
+        self.setText(f"{percentage}%")
```

> Good point. It has been centralized.

skepticspriggan on 2025-11-06 20:04

qutebrowser/browser/webengine/webenginetab.py

```
@@ -730,6 +730,7 @@ class WebEngineZoom(browsertab.AbstractZoom):

     def _set_factor_internal(self, factor):
         self._widget.setZoomFactor(factor)
+        self.factor_changed.emit(factor)
```

> Yes, that seems more reliable. I have connected the signals based on the `WebEnginePrinting` class.

The-Compiler on 2025-11-06 20:45

qutebrowser/browser/webengine/webenginetab.py

```
@@ -728,8 +728,22 @@ class WebEngineZoom(browsertab.AbstractZoom):

     _widget: webview.WebEngineView

+    def connect_signals(self):
+        """Called from WebEngineTab.connect_signals."""
+        page = self._widget.page()
+        try:
+            if machinery.IS_QT5:
+                page.zoomFactorChanged.connect(self.factor_changed)  # type: ignore[attr-defined]
+            else:
+                page.zoomFactorChanged.connect(self.factor_changed)
+        except AttributeError:
+            # Added in Qt 6.8
+            pass
```

> ```suggestion
>         if machinery.IS_QT6:
>             try:
>                 page.zoomFactorChanged.connect(self.factor_changed)
>             except AttributeError:
>                 # Added in Qt 6.8
>                 pass
> ```
>

The-Compiler on 2025-11-07 10:15

qutebrowser/browser/webengine/webenginetab.py

```
@@ -728,8 +728,20 @@ class WebEngineZoom(browsertab.AbstractZoom):

     _widget: webview.WebEngineView

+    def connect_signals(self):
```

> This doesn't seem to be called yet, so the zoom widget doesn't update at all.

The-Compiler on 2025-11-07 10:19

qutebrowser/mainwindow/statusbar/zoom.py

```
@@ -0,0 +1,24 @@
+"""Zoom percentage displayed in the statusbar."""
+
+from qutebrowser.browser import browsertab
+from qutebrowser.mainwindow.statusbar import textbase
+from qutebrowser.qt.core import pyqtSlot, QObject
+
+
+class Zoom(textbase.TextBase):
+
+    """Shows zoom percentage in current tab."""
+
+    def __init__(self, parent: QObject = None) -> None:
+        super().__init__(parent)
+        self.setText("100%")
```

> ```suggestion
>         self.on_zoom_changed(1)
> ```
>
> to avoid duplication with the text formatting?

The-Compiler on 2025-11-07 10:23

qutebrowser/mainwindow/statusbar/zoom.py

```
@@ -0,0 +1,24 @@
+"""Zoom percentage displayed in the statusbar."""
+
+from qutebrowser.browser import browsertab
+from qutebrowser.mainwindow.statusbar import textbase
+from qutebrowser.qt.core import pyqtSlot, QObject
+
+
+class Zoom(textbase.TextBase):
+
+    """Shows zoom percentage in current tab."""
+
+    def __init__(self, parent: QObject = None) -> None:
+        super().__init__(parent)
+        self.setText("100%")
+
+    @pyqtSlot(float)
+    def on_zoom_changed(self, factor: float) -> None:
+        """Update percentage when factor changed."""
+        percentage = int(100 * factor)
+        self.setText(f"{percentage}%")
```

> ```suggestion
>         self.setText(f"[{percentage}%]")
> ```
>
> For consistent formatting with other widgets in the statusbar.

skepticspriggan on 2025-11-07 11:04

qutebrowser/mainwindow/statusbar/zoom.py

```
@@ -0,0 +1,24 @@
+"""Zoom percentage displayed in the statusbar."""
+
+from qutebrowser.browser import browsertab
+from qutebrowser.mainwindow.statusbar import textbase
+from qutebrowser.qt.core import pyqtSlot, QObject
+
+
+class Zoom(textbase.TextBase):
+
+    """Shows zoom percentage in current tab."""
+
+    def __init__(self, parent: QObject = None) -> None:
+        super().__init__(parent)
+        self.setText("100%")
+
+    @pyqtSlot(float)
+    def on_zoom_changed(self, factor: float) -> None:
+        """Update percentage when factor changed."""
+        percentage = int(100 * factor)
+        self.setText(f"{percentage}%")
```

> I prefer a miminalistic style, however, I agree it is more important to remain consistent.

skepticspriggan on 2025-11-07 11:15

qutebrowser/mainwindow/statusbar/zoom.py

```
@@ -0,0 +1,24 @@
+"""Zoom percentage displayed in the statusbar."""
+
+from qutebrowser.browser import browsertab
+from qutebrowser.mainwindow.statusbar import textbase
+from qutebrowser.qt.core import pyqtSlot, QObject
+
+
+class Zoom(textbase.TextBase):
+
+    """Shows zoom percentage in current tab."""
+
+    def __init__(self, parent: QObject = None) -> None:
+        super().__init__(parent)
+        self.setText("100%")
```

> I think a little duplication can sometimes cause less mental overhead than another level of indirection, but I don't mind. Will change it.

The-Compiler on 2025-11-07 11:38

qutebrowser/mainwindow/statusbar/zoom.py

```
@@ -0,0 +1,24 @@
+"""Zoom percentage displayed in the statusbar."""
+
+from qutebrowser.browser import browsertab
+from qutebrowser.mainwindow.statusbar import textbase
+from qutebrowser.qt.core import pyqtSlot, QObject
+
+
+class Zoom(textbase.TextBase):
+
+    """Shows zoom percentage in current tab."""
+
+    def __init__(self, parent: QObject = None) -> None:
+        super().__init__(parent)
+        self.setText("100%")
```

> Fair point, I'm fine with either.

skepticspriggan on 2025-11-07 12:58

qutebrowser/browser/webengine/webenginetab.py

```
@@ -728,8 +728,20 @@ class WebEngineZoom(browsertab.AbstractZoom):

     _widget: webview.WebEngineView

+    def connect_signals(self):
```

> Yes, I should have done a final test. Fixed now.
