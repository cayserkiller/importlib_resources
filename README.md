# `importlib.resources`
This repository is to house the design and implementation of a planned
`importlib.resources` module for Python's stdlib -- aiming for
Python 3.7 -- along with a backport to target Python 2.7 - 3.6.

The key goal of this module is to replace
[`pkg_resources`](https://setuptools.readthedocs.io/en/latest/pkg_resources.html)
with a solution in Python's stdlib that relies on well-defined APIs.
This should not only make reading resources included in packages easier,
but have the semantics be stable and consistent.

## Goals
- Provide a reasonable replacement for `pkg_resources.resource_stream()`
- Provide a reasonable replacement for `pkg_resources.resource_string()`
- Provide a reasonable replacement for `pkg_resources.resource_filename()`
- Define an ABC for loaders to implement for reading resources
- Implement this in the stdlib for Python 3.7
- Implement a package for PyPI which will work on Python ==2.7,>=3.4

## Non-goals
- Replace all of `pkg_resources`
- For what is replaced in `pkg_resources`, provide an **exact**
  replacement

# Design

## Low-level
For [`importlib.abc`](https://docs.python.org/3/library/importlib.html#module-importlib.abc):
```python
from typing import Optional
from typing.io import BinaryIO


class ResourceReader(abc.ABC):

    def open_resource(self, path) -> BinaryIO:
        """Return a file-like object opened for binary reading.

        The path is expected to be relative to the location of the
        package this loader represents.
        """
        raise FileNotFoundError

    def resource_path(self, path) -> str:
        """Return the file system path to the specified resource.

        If the resource does not exist on the file system, raise
        FileNotFoundError.
        """
        raise FileNotFoundError
```

## High-level
For `importlib.resources`:
```python
import contextlib
import importlib
import os
import pathlib
import tempfile
from typing import ContextManager, Iterator, Union
from typing.io import BinaryIO


Path = Union[str, os.PathLike]


def _get_package(module_name):
    module = importlib.import_module(module_name)
    if module.__spec__.submodule_search_locations is None:
        raise TypeError(f"{module_name!r} is not a package")
    else:
        return module


def _normalize_path(path):
    if os.path.isabs(path):
        raise ValueError(f"{path!r} is absolute")
    normalized_path = os.path.normpath(path)
    if normalized_path.startswith(".."):
        raise ValueError(f"{path!r} attempts to traverse past package")
    else:
        return normalized_path


def open(module_name: str, path: Path) -> BinaryIO:
    """Return a file-like object opened for binary-reading of the resource."""
    normalized_path = _normalize_path(path)
    module = _get_package(module_name)
    return module.__spec__.loader.open_resource(normalized_path)


@contextlib.contextmanager
def path(module_name: str, path: Path) -> Iterator[pathlib.Path]:
    """A context manager providing a file path object to the resource.

    If the resource does not already exist on its own on the file system,
    a temporary file will be created. If the file was created, the file
    will be deleted upon exiting the context manager (no exception is
    raised if the file was deleted prior to the context manager
    exiting).
    """
    normalized_path = _normalize_path(path)
    module = _get_package(module_name)
    try:
        yield pathlib.Path(module.__spec__.resource_path(normalized_path))
    except FileNotFoundError:
        with module.__spec__.open_resource(normalized_path) as file:
            data = file.read()
        raw_path = tempfile.mkstemp()
        try:
            with open(raw_path, 'wb') as file:
                file.write(data)
            yield pathlib.Path(raw_path)
        finally:
            try:
                os.delete(raw_path)
            except FileNotFoundError:
                pass
```

If *module_name* has not been imported yet then it will be as a
side-effect of the call. The specified module is expected to be a
package, otherwise `TypeError` is raised. The module is expected to
have a loader specified on `__spec__.loader` which

For the *path* argument, it is expected to be a relative path. If
there are implicit references to the parent directory (i.e. `..`), they
will be resolved. If the normalized, relative path attempts to reference
beyond the location of the specified module, a `ValueError` will be
raised. The provided path is expected to be UNIX-style (i.e. to use
`/` as its path separator). Bytes-based paths are not supported.

All functions raise `FileNotFoundError` if the resource does not exist.


# Open Issues
## Support bytes-based paths?
Since import doesn't support bytes-based paths, maybe this API
shouldn't go down that route either? There's no guarantee for support
by loaders without extra effort and it's simply easier to not bother.

## Return a string rather than a `pathlib.Path` instance from `path`?
Using a higher-level API than strings is always better when it comes
to paths, but it will require depending on a third-party package on
PyPI for older versions of Python.

## Provide a read() function?
The function is equivalent to:
```python
def read(module_name, path):
    """Read the bytes of the resource."""
    with importlib.resources.open(...) as file:
        return file.read()
```

That's only 2 lines which everyone knows how to write (the `def` and
docstring equate to the same number of lines as the body of the
function). The only real reason to support a `read()` function is
potential performance benefits for loaders reading from e.g. a database.

But even if a `io.BytesIO` instance is returned by `open()`, it isn't
a costly operation to read all the bytes from the instance.
```python
data = b'...'
bytes_file = io.BytesIO(data)  # No copying; storing a reference.
with bytes_file as file:
    file.read()  # No copying; returning the reference.
```
So the only true overhead is the memory cost of a `BytesIO` instance,
which according to `sys.getsizeof(io.BytesIO())` is 88 bytes.
