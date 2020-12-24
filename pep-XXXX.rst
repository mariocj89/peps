PEP: XXXX
Title: Extensible customizations of the interpreter at startup
Author: Mario Corchero <mariocj89@gmail.com>
Sponsor: Pablo Galindo
BDFL-Delegate: XXXX
Discussions-To: <email address>
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 25-12-2020
Python-Version: 3.11
Post-History: python-ideas: 16th Dec. python-dev: 18th Dec.

Abstract
========

This post proposes supporting extensible customization of the interpreter, by
allowing users to install scripts that will be executed at startup.

Motivation
==========

System administrators, tools that repackage the interpreter and some
libraries need to customize aspects of the interpreter at startup time.

This is usually achieved via `sitecustomize.py` for system administrators
whilst libraries rely on exploiting `pth` files. This post proposes a way of
achieving the same in a more user-friendly and structured way.

Limitations of `pth` files
--------------------------

If a library needs to perform any customization before an import or that
relates to the general working of the interpreter, they often rely on the
fact that `pth` files, which are loaded at startup, can include Python code
that will be executed when the `pth` file is evaluated.

Note that `pth` files were originally developed to just add additional
directories to `sys.path`, but it also allowed to contain lines that started
with "import", which will be \`exec\`ed. Users have exploited this feature to
allow the customizations that they needed. See setuptools [#setuptools]_ or
betterexceptions [#betterexceptions]_ as examples.

Using `pth` files for this purpose is far from ideal for library developers,
as they need to inject code into a single line preceded by an import, making
it rather unreadable. Library developers following that practice will usually
create a module that performs all actions on import, as done by
betterexceptions [#betterexceptions]_, but the approach is still not really really user
friendly.

Aditionally, it is also non-ideal for users of the interpreter as if they
want to inspect what is being executed at Python startup they need to review
all the `pth` files for potential code execution which can be spread across
all site paths. Most of those pth will be "legit" pth files that just modify
the path, answering the question of "what is changing my interpreter at
startup" a rather complex one.

Lastly, there have been multiple suggestions for removing code execution from
`pth` files, see [#bpo-24534]_ and [#bpo-33944]_.

Limitations of `sitecustomize.py`
---------------------------------

Whilst sitecustomize is an acceptable solution, it assumes a single person is
in charge of the system and the interpreter. If both the system administrator
and the responsibility of provisioning the interpreter want to add
customizations at the interpreter startup they need to agree on the contents
of the file and combine all the changes. This is not a major limitation
though, and it is not the main driver of this change, but should the change
happen, it will also improve the situation for this users, as rather than
having a `sitecustomize.py` which performs all those actions, they can have
custom isolated files named after the features they want to enhance. As an
example, ubuntu could change their current `sitecustomze.py` to just be
`ubuntu_apport_python_hook`. This not only better represents its intent but
also gives users of the interpreter a better understanding of the
modifications happening on their interpreter.

Benefits of `__sitecustomize__`
-------------------------------

Having a structured way of injecting custom startup scripts, will allow
supporting in a better way the cases presented above. It will result in both
maintainers and users better experience as detailed, and allow CPython to
deprecate and eventually remove code execution from `pth` files, as desired
in the previously mentioned bpos.
Additionally, these solutions provide a unique way to support all use-cases
that before have been fulfilled via the misuse of `pth` files,
`sitecustomize.py` and `usercustomize.py`. The use of a `__sitecustomize__`
will allow for packages, tools and system admins to inject scripts that will
be loaded at startups through an easy to understand mechanism.

Rationale
=========

This post proposes supporting extensible customization of the interpreter at
startup by allowing users to install scripts into a namespace package named
`__sitecustomze__` that will be executed a startup. The implementation will
attempt to import `__sitecustomize__` and fail silently if not present.
Should the namespace package be present, Python will import all scripts
within it.

The `site` module will expose an option on its main function that allows
listing all scripts found in that namespace package, which will allow users
to quickly see all customizations that affect an interpreter.

We will also work with build backends on facilitating the installation of
these files.

Why `__sitecustomize__`
-----------------------

The name aims to follow the already existing concept of `sitecustomize.py`.
To prevent conflict with the existing package and to make explicit it is not
a module that is intended to be imported by the user, it is surrounded with
double `_`.

Namespace package
-----------------

Having `__sitecustomize__` be a namespace package allows developers to
install customizations in any path present in their Python path. This allows
for easy installation in usersite, at the interpreter site or any other path
the user chooses to add. We choose to use a standard namespace package rather
than just looking for the folder name in the different site paths to support
tools that might modify the path in any way and to make it easier to reason
about. Python will just "import" `__sitecustomize__` and execute all scripts
in that namespace package.

The risk of using a namespace package for it is that a tool might by mistake
choose to install an `__init__.py` in the folder, breaking the mechanism to
resolve the namespace package. Given that we are documenting this as a
"folder to drop scripts" and the fact that it is a namespace package is an
implementation detail, we do not expect this to be an issue.

Disabling start scripts
-----------------------

In some scenarios, like when the startup time is key, it might be desired to
disable this option altogether. Whilst we could have added a new flag to do
so, we think that the already existing flag `-S` [#s-flag]_ is already good enough,
as it disables all `site` related manipulation. If the flag is passed in,
`__sitecustomze__` will not be used.

Order of execution
------------------

The scripts in `__sitecustomize__` will be executed in alphabetic order after
all other site-dependent manipulations have been executed. This means after
the evaluation of all `pth` files and the execution of `sitecutzomize.py` and
`usercustomize.py`. We considered executing them in random order, but that
could result in different results depending on how the interpreter chooses to
pick up those files. So even if it won't be a good practice to rely on other
files being executed, we think that is better than having randomly different
results on interpreter startup.

Additionally, if needed, packages can declare dependencies between their
startup scripts by importing them from `__sitecustomize__`. As all these
files are just Python scripts that are executed by importing them, if one of
them imports another (thanks to `__sitecustomize__` being a namespace
package), it will make sure that it is executed before the other.

Impact on startup time
----------------------

If an interpreter is not using the tool, the impact on performance is
expected to be that of an import that fails and the exception being ignored.
This impact will be reduced in the future as we will remove two other
imports: "sitecustomize.py" and "usercustomize.py".

If the user has custom scripts, we think that the impact on the performance
of importing the namespace package and walking it's acceptable, as the user
wants to use this feature. If they need to run a time-sensitive application,
they can always use `-S` to disable this entirely.

Running "./python -c pass" with perf on 50 iterations, repeating 50 times the
command on each and getting the geometric mean on a commodity laptop did not
reveal any substantial raise on CPU time beyond nanoseconds with this
implementation, which is expected given the additional import.

Failure handling
----------------

Any error on any of the scripts will not be logged unless the interpreter is
run in verbose mode and it should not stop the evaluation of other scripts.
The user will just receive a message saying that the script failed to be
executed, that verbose mode can be used to get more information. This
behaviour follows the one already existing for `sitecustomize.py`.

Scripts naming convention
-------------------------

Packages will be encouraged to include the name of the package within the
name of the script to avoid collisions between packages, even if they might
likely.

Relationship with sitecustomize and usercustomize
-------------------------------------------------

The existing logic for `sitecustomize.py` and `usercustomize.py` will be left
and later deprecated and scheduled for removal. Once `__sitecustomize__` is
supported, it will provide better integration for all existing users, and
even if it will indeed require a migration for System administrators, we
expect the effort required to be minimal, it will just require moving and
renaming the current `sitecustomize.py` into the new provided folder.

Identifying all installed scripts
---------------------------------

To facilitate debugging of the Python startup, a new option will be added to
the main of the site module to list all scripts that will be executed as part
of the `__sitecustomze__` initialization.

How to teach this
=================

This can be documented and taught as simple as saying that the interpreter
will try to import the `__sitecustomize__` package at startup and it if finds
any modules within it, it will then execute all of them.

For system administrators and tools that package the interpreter, we can now
recommend placing files in `__sitecustomze__` as they used to place
`sitecustomize.py`. Being more comfortable on that their content won't be
overridden by the next person, as they can provide with specific files to
handle the logic they want to customize.

Library developers should be able to specify a new argument on tools like
setuptools that will inject those new files. Something like
`sitecustomize_scripts=["scripts/betterexceptions.py"]`, which allows them to
add those. Should the build backend not support that, they can manually
install them as they used to do with `pth` files. We will recommend them to
include the name of the package as part of the scripts name.

Backward compatibility
======================

We propose to add support for `__sitecustomize__` in the next release of
Python, add a warning on the three next releases on the deprecation and
future removal of `sitecustomize.py`, `usercustomize.py` and code execution
in `pth` files, and remove it after maintainers have had 4 releases to
migrate. Ignoring those lines in pth files.

Reference Implementation
========================

An initial implementation that passes the CPython test suite is available for
evaluation [#reference-implementation]_.

This implementation is just for the reviewer to play with and check potential
issues that this PEP could generate.

Rejected Ideas
==============

Do nothing
----------

Whilst the current status "works" it presents the issues listed in the
motivation. After analysing the impact of this change, we believe it is worth
given the enhanced experience it brings.

Formalize using `pth` files
---------------------------

Another option would be to just glorify and document the usage of `pth` files
to inject code at startup code, but that is a suboptimal experience for users
as listed in the motivation.

Searching files within a folder rather than a namespace package
---------------------------------------------------------------

Similarly to how `pth` files are looked up, we could have implemented the
`__sitecustomize__` logic. We preferred to use a namespace package as it
brings other benefits like being able to declare dependencies easily and we
consider it is easier to teach.

Support for shutdown custom scripts
-----------------------------------

`init.d` users might be tempted to implement this feature in a way that users
could also add code at shutdown, but extra support for that is not needed, as
Python users can already do that via `atexit`.

.. [#bpo-24534]
   https://bugs.python.org/issue24534

.. [#bpo-33944]
   https://bugs.python.org/issue33944

.. [#s-flag]
   https://docs.python.org/3/using/cmdline.html#id3

.. [#setuptools]
   https://github.com/pypa/setuptools/blob/b6bbe236ed0689f50b5148f1172510b975687e62/setup.py#L100

.. [#betterexceptions]
   https://github.com/Qix-/better-exceptions/blob/7b417527757d555faedc354c86d3b6fe449200c2/better_exceptions_hook.pth#L1

.. [#reference-implementation]
   https://github.com/mariocj89/cpython/tree/pu/__sitecustomize__
