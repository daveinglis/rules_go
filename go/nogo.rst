|logo| nogo build-time code analysis
====================================

.. _nogo: nogo.rst#nogo
.. _go_library: core.rst#go_library
.. _go_tool_library: core.rst#go_tool_library
.. _analysis: https://godoc.org/golang.org/x/tools/go/analysis
.. _Analyzer: https://godoc.org/golang.org/x/tools/go/analysis#Analyzer
.. _GoLibrary: providers.rst#GoLibrary
.. _GoSource: providers.rst#GoSource
.. _GoArchive: providers.rst#GoArchive
.. _vet: https://golang.org/cmd/vet/

.. role:: param(kbd)
.. role:: type(emphasis)
.. role:: value(code)
.. |mandatory| replace:: **mandatory value**
.. |logo| image:: nogo_logo.png
.. footer:: The ``nogo`` logo was derived from the Go gopher, which was designed by Renee French. (http://reneefrench.blogspot.com/) The design is licensed under the Creative Commons 3.0 Attributions license. Read this article for more details: http://blog.golang.org/gopher


**WARNING**: This functionality is experimental, so its API might change.
Please do not rely on it for production use, but feel free to use it and file
issues.

``nogo`` is a tool that analyzes the source code of Go programs. It runs
alongside the Go compiler in the Bazel Go rules and rejects programs that
contain disallowed coding patterns. In addition, ``nogo`` may report
compiler-like errors.

``nogo`` is a powerful tool for preventing bugs and code anti-patterns early
in the development process.

.. contents:: :depth: 2

-----

Overview
--------

Writing and registering analyzers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``nogo`` analyzers are Go packages that declare a variable named ``Analyzer``
of type `Analyzer`_ from package `analysis`_. Each analyzer is invoked once per
Go package, and is provided the abstract syntax trees (ASTs) and type
information for that package, as well as relevant results of analyzers that have
already been run. For example:

.. code:: go

    // package importunsafe checks whether a Go package imports package unsafe.
    package importunsafe

    import (
      "strconv"

      "github.com/bazelbuild/rules_go/go/tools/analysis"
    )

    var Analyzer = &analysis.Analyzer{
      Name: "importunsafe",
      Doc: "reports imports of package unsafe",
      Run: run,
    }

    func run(pass *analysis.Pass) (interface{}, error) {
      var findings []*analysis.Finding
      for _, f := range pass.Files {
        for _, imp := range f.Imports {
          path, err := strconv.Unquote(imp.Path.Value)
          if err == nil && path == "unsafe" {
            pass.Reportf(imp.Pos(), "package unsafe must not be imported")
          }
        }
      }
      return nil, nil
    }

Any diagnostics reported by the analyzer will stop the build. Do not emit
diagnostics unless they are severe enough to warrant stopping the build.

Each analyzer must be written as a `go_tool_library`_ rule and must import
`@org_golang_x_tools//go/analysis:go_tool_library`, the `go_tool_library`_
version of the package `analysis`_ target. `go_tool_library`_ is identical to
`go_library`_ but avoids a bootstrapping problem, which will be explained later.
For example:

.. code:: bzl

    load("@io_bazel_rules_go//go:def.bzl", "go_tool_library")

    go_tool_library(
        name = "importunsafe",
        srcs = ["importunsafe.go"],
        importpath = "importunsafe",
        deps = ["@org_golang_x_tools//go/analysis:go_tool_library"],
        visibility = ["//visibility:public"],
    )

    go_tool_library(
        name = "unsafedom",
        srcs = [
            "check_dom.go",
            "dom_utils.go",
        ],
        importpath = "unsafedom",
        deps = ["@org_golang_x_tools//go/analysis:go_tool_library"],
        visibility = ["//visibility:public"],
    )

The `nogo`_ rule generates a program that analyzes Go source code. This program
is run alongside the compiler. You must define a `nogo`_ target whose ``deps``
attribute contains all analyzer targets. These analyzers will be linked to the
generated ``nogo`` binary and executed at build-time.

.. code:: bzl

    load("@io_bazel_rules_go//go:def.bzl", "nogo")

    nogo(
        name = "nogo",
        deps = [
            ":importunsafe",
            ":unsafedom",
            "@analyzers//:loopclosure", # analyzers can be imported from a remote repo
        ],
        visibility = ["//visibility:public"], # must have public visibility
    )

**NOTE**: Writing each ``nogo`` analyzer as a `go_tool_library`_ rule instead of
a `go_library`_ rule avoids a circular dependency: `go_library`_ implicitly
depends on `nogo`_, which depends on analyzer libraries, which must not depend
on `nogo`_. `go_tool_library`_ does not have the same implicit dependency.

Finally, the `nogo`_ target must be passed to ``go_register_toolchains``
in your ``WORKSPACE`` file.

.. code:: bzl

    load("@io_bazel_rules_go//go:def.bzl", "go_rules_dependencies", "go_register_toolchains")
    go_rules_dependencies()
    go_register_toolchains(nogo="@//:nogo")

The generated ``nogo`` program will run alongside the compiler when building any
Go target (e.g. `go_library`_) within your workspace, even if the target is
imported from an external repository. However, ``nogo`` will not run when
targets from the current repository are imported into other workspaces and built
there.

Configuring analyzers
~~~~~~~~~~~~~~~~~~~~~

By default, ``nogo`` analyzers will emit diagnostics for all Go source files
built by Bazel. This behavior can be changed with a JSON configuration file.

The top-level JSON object in the file must be keyed by the name of the analyzer
being configured. These names must match the ``Analyzer.Name`` of the registered
analysis package. The JSON object's values are themselves objects which may
contain the following key-value pairs:

+----------------------------+---------------------------------------------------------------------+
| **Key**                    | **Type**                                                            |
+----------------------------+---------------------------------------------------------------------+
| ``"description"``          | :type:`string`                                                      |
+----------------------------+---------------------------------------------------------------------+
| Description of this analyzer configuration.                                                      |
+----------------------------+---------------------------------------------------------------------+
| ``"only_files"``           | :type:`dictionary, string to string`                                |
+----------------------------+---------------------------------------------------------------------+
| Specifies files that this analyzer will emit diagnostics for.                                    |
| Its keys are regular expression strings matching Go file names, and its values are strings       |
| containing a description of the entry.                                                           |
| If both ``only_files`` and ``exclude_files`` are empty, this analyzer will emit diagnostics for  |
| all Go files built by Bazel.                                                                     |
+----------------------------+---------------------------------------------------------------------+
| ``"exclude_files"``        | :type:`dictionary, string to string`                                |
+----------------------------+---------------------------------------------------------------------+
| Specifies files that this analyzer will not emit diagnostics for.                                |
| Its keys and values are strings that have the same semantics as those in ``only_files``.         |
| Keys in ``exclude_files`` override keys in ``only_files``. If a .go file matches a key present   |
| in both ``only_files`` and ``exclude_files``, the analyzer will not emit diagnostics for that    |
| file.                                                                                            |
+----------------------------+---------------------------------------------------------------------+

Example
^^^^^^^

The following configuration file configures the analyzers named ``importunsafe``
and ``unsafedom``. Since the ``loopclosure`` analyzer is not explicitly
configured, it will emit diagnostics for all Go files built by Bazel.

.. code:: json

    {
      "importunsafe": {
        "exclude_files": {
          "src/foo.go": "manually verified that behavior is working-as-intended",
          "src/bar.go": "see issue #1337"
        }
      },
      "unsafedom": {
        "only_files": {
          "src/js/*": ""
        },
        "exclude_files": {
          "src/(third_party|vendor)/*": "enforce DOM safety requirements only on first-party code"
        }
      }
    }

This label referencing this configuration file must be provided as the
``config`` attribute value of the ``nogo`` rule.

.. code:: bzl

    nogo(
        name = "nogo",
        deps = [
            ":importunsafe",
            ":unsafedom",
            "@analyzers//:loopclosure",
        ],
        config = "config.json",
        visibility = ["//visibility:public"],
    )

Running vet
~~~~~~~~~~~

`vet`_ is a tool that examines Go source code and reports correctness issues not
caught by Go compilers. It is included in the official Go distribution.

You can choose to run `vet`_ alongside the Go compiler by setting the ``vet``
attribute in your `nogo`_ target:

.. code:: bzl

    nogo(
        name = "nogo",
        vet = True,
        visibility = ["//visibility:public"],
    )

In the above example, the generated ``nogo`` program will only run `vet`_.
`vet`_ can also run alongside ``nogo`` analyzers given by the ``deps``
attribute.

`vet`_ will print error messages and stop the build if any correctness issues
are found in the source code being compiled. Only a subset of `vet`_ checks
which are 100% accurate will be executed. This is the same subset of `vet`_
checks that are run by the ``go`` tool during ``go test``.


API
---

nogo
~~~~

This generates a program that that analyzes the source code of Go programs. It
runs alongisde the Go compiler in the Bazel Go rules and rejects programs that
contain disallowed coding patterns.

Attributes
^^^^^^^^^^

+----------------------------+-----------------------------+---------------------------------------+
| **Name**                   | **Type**                    | **Default value**                     |
+----------------------------+-----------------------------+---------------------------------------+
| :param:`name`              | :type:`string`              | |mandatory|                           |
+----------------------------+-----------------------------+---------------------------------------+
| A unique name for this rule.                                                                     |
+----------------------------+-----------------------------+---------------------------------------+
| :param:`deps`              | :type:`label_list`          | :value:`None`                         |
+----------------------------+-----------------------------+---------------------------------------+
| List of Go libraries that will be linked to the generated nogo binary.                           |
|                                                                                                  |
| These libraries must declare an ``analysis.Analyzer`` variable named `Analyzer` to ensure that   |
| the analyzers they implement are called by nogo.                                                 |
|                                                                                                  |
| To avoid bootstrapping problems, these libraries must be `go_tool_library`_ targets, and must    |
| import `@org_golang_x_tools//go/analysis:go_tool_library`, the `go_tool_library`_ version of     |
| the package `analysis`_ target.                                                                  |
+----------------------------+-----------------------------+---------------------------------------+
| :param:`config`            | :type:`label`               | :value:`None`                         |
+----------------------------+-----------------------------+---------------------------------------+
| JSON configuration file that configures one or more of the analyzers in ``deps``.                |
+----------------------------+-----------------------------+---------------------------------------+
| :param:`vet`               | :type:`bool`                | :value:`False`                        |
+----------------------------+-----------------------------+---------------------------------------+
| Whether to run the `vet`_ tool.                                                                  |
+----------------------------+-----------------------------+---------------------------------------+

Example
^^^^^^^

.. code:: bzl

    nogo(
        name = "nogo",
        deps = [
            ":importunsafe",
            ":otheranalyzer",
            "@analyzers//:unsafedom",
        ],
        config = ":config.json",
        vet = True,
        visibility = ["//visibility:public"],
    )
