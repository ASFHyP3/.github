[Mypy](https://mypy.readthedocs.io/en/stable/) is a static type checker for Python and is used in all of our Python projects as a [reusable action](https://github.com/ASFHyP3/actions/blob/main/.github/workflows/reusable-mypy.yml).

You can find our currently recommended config options [here](https://github.com/ASFHyP3/actions/#reusable-mypyyml).

This article documents how we use mypy, including some non-obvious configuration options and error cases that we've encountered. Please see https://mypy.readthedocs.io/en/stable/ for the official mypy documentation.

## Running mypy locally

*Last updated 2025-01-23*

To run mypy locally, activate the conda/mamba environment for the project (`mypy` should be installed via conda-forge or pip in `environment.yml`) and run:

```
mypy .
```

## mypy cache

*Last updated 2025-01-23*

If you're using the `install_types = true` config option and you see `error: --install-types failed (no mypy cache directory)`, then you should run `mkdir .mypy_cache` before running mypy. See https://github.com/python/mypy/issues/10768

If at any point mypy is failing to recognize config changes or otherwise behaving strangely, the problem might be solved by deleting and re-creating the `.mypy_cache` directory.

## Ignoring specific error codes

*Last updated 2025-01-23*

Mypy includes an error code such as `[attr-defined]` or `[arg-type]` with each error message.

If you have carefully reviewed a particular error, you understand why it's being reported, and you decide that ignoring the error is the best resolution, then you can ignore the error code for just that line by adding a `# type: ignore[error-code]` comment to the offending line of code, e.g. `# type: ignore[arg-type]`.

You can also ignore multiple error codes for that line by including multiple codes as a comma-delimited list, e.g. `# type: ignore[arg-type,attr-defined]`.

You should avoid using a blanket `# type: ignore` comment.

Also see [Error codes](https://mypy.readthedocs.io/en/stable/error_codes.html).

## Configuration via `pyproject.toml`

*Last updated 2025-01-23*

We recommend using `pyproject.toml` for config options, as documented [here](https://mypy.readthedocs.io/en/stable/config_file.html#using-a-pyproject-toml-file). Note that you can have multiple `[[tool.mypy.overrides]]` sections as shown [here](https://mypy.readthedocs.io/en/stable/config_file.html#example-pyproject-toml).

## Missing imports

*Last updated 2025-01-23*

See [Missing imports](https://mypy.readthedocs.io/en/stable/running_mypy.html#missing-imports) for the full documentation.

Mypy flags three types of missing imports:

**module is installed, but missing library stubs or py.typed marker:** This means that mypy found the imported module in the current environment but was not able to find type annotations for it. There are various solutions for this, but our recommended solution is to simply disable the error code via the `disable_error_code = ["import-untyped"]` config option, which means that mypy will assume dynamic typing for that module (via the `Any`) type. This option is already included in our [recommended config options](https://github.com/ASFHyP3/actions/#reusable-mypyyml).

**Library stubs not installed:** mypy flags this error for third-party libraries when type stubs are available but not installed. Our recommended config already includes the `install_types = true` option, which tells mypy to automatically install missing type stubs.

**Cannot find implementation or library stub:** This means that mypy was not able to find the imported module in the current environment. You can review the [troubleshooting tips](https://mypy.readthedocs.io/en/stable/running_mypy.html#cannot-find-implementation-or-library-stub) for this error, but we've also seen unavoidable instances of this error for certain modules, for example ISCE-related modules as discussed [here](https://github.com/ASFHyP3/hyp3-autorift/pull/307), and when a module is installed as a compiled `.so` file, as shown [here](https://github.com/ASFHyP3/hyp3-testing/blob/059a1a1262e421fefbd1c28ba3d623177bd4d354/hyp3_testing/compare.py#L10-L11). You can add a `# type: ignore[import-not-found]` comment to the `import` lines to suppress this error, in which case mypy assumes dynamic typing for those modules.

## Failure to automatically install type stubs

*Last updated 2025-01-23*

Using the `install_types = true` config option may cause problems for pip's dependency resolver. For example, we encountered the following error when adding mypy to hyp3-autorift (see <https://github.com/ASFHyP3/hyp3-autorift/issues/301>):

```
Installing collected packages: urllib3, types-requests
  Attempting uninstall: urllib3
    Found existing installation: urllib3 1.26.19
    Uninstalling urllib3-1.26.19:
      Successfully uninstalled urllib3-1.26.19
ERROR: pip's dependency resolver does not currently take into account all the packages that are installed. This behaviour is the source of the following dependency conflicts.
botocore 1.35.94 requires urllib3<1.27,>=1.25.4; python_version < "3.10", but you have urllib3 2.3.0 which is incompatible.
```

The solution is to add the type stub packages (in this case, just `types-requests`) to the project requirements so that they get installed when building the environment.

## Excluding files

*Last updated 2025-01-23*

You may need to exclude certain files from mypy's type checking, commonly `build/` and `setup.py`, which often result in `Duplicate module` errors. For example, when running mypy in our [hyp3](https://github.com/ASFHyP3/hyp3) repo:

```
lib/dynamo/dynamo/__init__.py: error: Duplicate module named "dynamo" (also at "./lib/dynamo/build/lib/dynamo/__init__.py")
```

```
lib/lambda_logging/setup.py: error: Duplicate module named "setup" (also at "./lib/dynamo/setup.py")
```

These can be solved by adding the following to the `[tool.mypy]` section in `pyproject.toml` (also see [`pyproject.toml` in hyp3](https://github.com/ASFHyP3/hyp3/blob/v9.2.0/pyproject.toml#L42-L45)):

```toml
[tool.mypy]
exclude = [
    '/build/',
    '/setup\.py$',
]
```

Note that using single-quoted strings avoids having to escape special characters.

Also see:
- https://mypy.readthedocs.io/en/stable/command_line.html#cmdoption-mypy-exclude
- https://mypy.readthedocs.io/en/stable/config_file.html#example-pyproject-toml

## Skipping imports for excluded modules

*Last updated 2025-01-23*

If you exclude a certain module that is imported by other modules in the project, then you will also need to tell mypy not to follow those imports (otherwise, you will still get errors for the excluded files). For example, [hyp3-autorift](https://github.com/ASFHyP3/hyp3-autorift/blob/v0.21.2/pyproject.toml#L102-L106) uses the following option to exclude any directory named `vend/`:

```toml
[tool.mypy]
exclude = ["/vend/"]
```

and also uses the following options to skip following imports for `hyp3_autorift.vend` and its sub-modules:

```toml
[[tool.mypy.overrides]]
module = "hyp3_autorift.vend.*"
follow_imports = "skip"
```

Also see:
- https://mypy.readthedocs.io/en/stable/config_file.html#confval-follow_imports
- https://mypy.readthedocs.io/en/stable/config_file.html#using-a-pyproject-toml-file

## Duplicate `conftest.py` files

*Last updated 2025-01-23*

If your tests make use of multiple `conftest.py` files and mypy complains, the problem can likely be solved by adding a `tests/__init__.py` file, which is also [recommended by pytest](https://docs.pytest.org/en/stable/explanation/pythonpath.html). See https://github.com/ASFHyP3/asf-tools/pull/257 for an example.