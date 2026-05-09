.. SPDX-License-Identifier: GPL-2.0

=========================
Dry-run build scripts
=========================

The kernel build system can turn GNU make's dry-run output into a shell
script that rebuilds the configured kernel without invoking make for the
actual build steps::

    make defconfig
    make -n > build.sh
    bash build.sh

For an out-of-tree build, generate the script with the same ``O=`` value and
run the script from the object directory::

    make O=$objtree defconfig
    make O=$objtree -n > $objtree/build.sh
    ( cd $objtree && bash build.sh )

The generated script is intended to be an auditable, reproducible transcript
of a single full build.  It is not a replacement for kbuild during normal
development.

Motivation
==========

``make -n`` traditionally prints the commands that make would execute, but
the output is not directly usable as a shell script.  Recursive make commands
are printed, generated prerequisites may be missing, command wrappers include
dependency-tracking logic that is useful only to make, and some helper scripts
execute commands internally instead of exposing them to dry-run output.

The dry-run script mode reduces those differences.  The result is useful
when the build commands themselves are the artifact of interest:

* Make can be invoked in a restricted environment, for example with a mostly
  read-only source tree, while the generated script is executed separately.

* The script is an audit log of the commands used for the build.  It can be
  stored, reviewed, and compared over time.

* A straight-line shell script has fewer places for unexpected Makefile logic
  or environment-injected make syntax to hide.

* The script can rebuild an existing configuration so the resulting binaries
  can be compared with the output of the normal kbuild path.

This is related to reproducible builds, but it solves a different problem:
reproducible builds try to make equivalent builds produce identical output,
while dry-run scripts make the build recipe itself explicit.

Basic use
=========

Start from a configured tree.  The examples below use ``defconfig`` only to
make the setup self-contained::

    make defconfig
    make -n > build.sh
    bash -n build.sh
    bash build.sh

For an out-of-tree build::

    make O=$objtree defconfig
    make O=$objtree -n > $objtree/build.sh
    bash -n $objtree/build.sh
    ( cd $objtree && bash build.sh )

The script must be run with the same relevant environment as the dry-run
generation.  This includes variables such as ``ARCH``, ``CROSS_COMPILE``,
``LLVM``, compiler overrides, and other tool selections.  If an environment
wrapper such as ccache is part of the generated commands, the script will use
it too.

The generated script uses Bash-specific shell function export syntax to make
recursive ``make`` invocations no-ops during replay, so it should be run with
``bash`` rather than a generic POSIX ``sh``.

The script enables ``set -e`` so command failures stop the replay instead of
being hidden by later commands.

What the script does
====================

The generated script contains the command lines that kbuild would normally
execute for the requested target.  In dry-run mode, kbuild changes selected
rules so they print the underlying build commands instead of make-specific
bookkeeping:

* ``filechk`` rules print direct file generation commands instead of
  comparing temporary files with existing targets.

* ``if_changed`` and related wrappers print the command body directly instead
  of writing ``.cmd`` files or running ``fixdep``.

* Recursive ``$(MAKE)`` invocations are still printed by GNU make, but the
  generated script defines a shell function named ``make`` that usually
  returns success.  This keeps recursive kernel make lines visible in the log
  without executing make during replay.  Some tools makefiles are allowed to
  run normally because their dry-run output is tied to the tools source
  directory and is not a standalone command stream from the kernel object
  directory.

* Helper scripts that normally perform internal build steps, such as
  ``scripts/link-vmlinux.sh``, print the commands they would run when
  ``dry_run`` is set.

* Generated prerequisites that are normally produced by parent makefiles are
  documented with dry-run-only dependencies or ``PHONY`` entries so recursive
  dry-run make invocations can keep walking the build graph.

Some commands are still intentionally run while generating the script.  GNU
make itself executes recipe lines prefixed with ``+`` even in dry-run mode,
and kbuild uses this where it needs a recursive invocation to discover and
print commands from a lower-level makefile.  Those recursive make commands are
printed into the script too, but become no-ops when the script is replayed.

Scope and limitations
=====================

The dry-run script describes one build of one configured tree, target, and
environment.  It has important limitations:

* It is not incremental.  The script does not decide what is out of date, and
  it does not update dependency metadata for a later incremental build.

* It is not parallel.  The script is a straight-line replay of the printed
  commands.

* It is not meant for production kernel builds.  Use normal kbuild for regular
  development and release builds.

* It is not a general command tracer.  It records the commands kbuild emits;
  it does not observe arbitrary process execution like ``strace`` would.

* It is not the same as ``compile_commands.json``.  Compilation databases are
  useful for tools that need compile commands for translation units, but they
  do not describe the full kernel build, including generated files, archives,
  vmlinux linking, post-processing, or boot image generation.

* The default kernel build for common configurations is the main supported
  use case.  Other targets may produce incomplete or meaningless output until
  their make rules are taught how to behave in dry-run mode.

* Install, packaging, documentation, tools, Rust, static analysis, and test
  targets are separate target families.  They may depend on files produced by
  a previous kernel build, host packaging tools, language-specific tooling, or
  external downloads.  Do not assume their dry-run output is complete unless
  that target has been validated separately.

* Configurations that rely on external tools or language ecosystems may need
  additional rule coverage before their dry-run scripts are complete.

* The script is tied to the configured source and object trees.  ``O=`` paths
  are emitted as absolute object-tree paths in many commands.  Moving the
  script to another tree, or changing generated headers and configuration
  between generation and replay, may invalidate it.  To compare two builds,
  keep the original dry-run object tree for script replay and use a snapshot
  copy for the normal make build.

Existing build state
====================

``make -n`` describes what make would do from the current state of the object
tree.  A dry-run script generated from a partial or complete build tree is
therefore a script for that state, not necessarily a complete script for
building the configured kernel from scratch.

This matters when the script is used as an audit artifact or when its output is
compared with a normal build:

* If the goal is a full build transcript, generate the script from a clean
  configured object tree.  A common workflow is to create a fresh ``O=`` tree,
  run the configurator, generate the script, and then replay it in that same
  object tree.

* If the goal is to capture an incremental rebuild, keep the existing object
  tree.  The resulting script is tied to the files, generated headers, command
  metadata, and timestamps present when ``make -n`` was run.

* If the tree has already been built, ``make -n`` may print a shorter or
  different script because make sees existing targets and dependency state.
  Some kbuild rules are forced and may still appear, but the output should not
  be treated as a clean-build transcript.

* If the tree is partially built, the generated script may rely on existing
  intermediate files that are not created by the script.  Use a clean object
  tree, or keep an exact snapshot of the object tree together with the script.

For replay-equivalence testing, generate the script first, snapshot the object
tree immediately afterwards, replay the script in the original object tree, and
run normal make in the snapshot copy.

When to prepare first
=====================

The preferred workflow is to generate the dry-run script from a configured
tree and let the script include the necessary build steps.  If a target still
requires state that is not represented in dry-run output, build that state
before generating the script.  For example::

    make O=$objtree defconfig
    make O=$objtree prepare
    make O=$objtree -n > $objtree/build.sh
    ( cd $objtree && bash build.sh )

Needing such a manual preparation step for the default build should be treated
as a kbuild bug or missing dry-run rule coverage.

Guidelines for kbuild rules
===========================

Kbuild rules should preserve normal make behavior unless ``dry_run`` is set.
When adding or changing rules, keep the following points in mind.

Expose real build commands
--------------------------

If a recipe calls a helper that performs more build steps internally, decide
whether the helper should print its internal commands in dry-run mode.  The
vmlinux link is the main example: ``scripts/link-vmlinux.sh`` uses a helper
that either runs a command normally or prints it when ``dry_run`` is set.

Do not hide error checking
--------------------------

The generated script should fail when the corresponding normal build command
would fail.  Avoid dry-run transformations that turn a command with important
error handling into an unconditional success.  If a helper prints a command in
dry-run mode, make sure the replayed command still carries the meaningful
redirections, quoting, and shell control flow.

Document generated prerequisites
--------------------------------

Dry-run recursive make can fail before printing useful output if it sees a
prerequisite that does not exist and has no rule in the current makefile.  If
the file is generated by a parent makefile or an earlier part of the script,
add a dry-run-only rule or mark it phony in the makefile that needs it.

Examples include generated linker scripts, ``modules.order``, vDSO image
sources, ``scripts/mod/modpost``, and host tools used only as prerequisites.

Avoid make-only bookkeeping
---------------------------

Dependency extraction, ``.cmd`` file updates, temporary comparison files, and
similar bookkeeping are useful to make but usually do not belong in the
replay script.  In dry-run mode, prefer printing the command that creates the
target directly.

Keep output shell-safe
----------------------

The dry-run output is parsed by the shell.  Avoid emitting informational make
messages, kbuild status lines, or partial shell fragments as standalone lines.
Quote paths and variables in commands that may contain shell metacharacters.

Testing
========

At minimum, changes that affect dry-run script generation should test both
script generation and replay for a simple configuration::

    make O=$objtree defconfig
    make O=$objtree -n > $objtree/build.sh
    bash -n $objtree/build.sh
    ( cd $objtree && bash build.sh )

For changes that affect generated files, final linking, module handling, or
architecture boot images, also test a configuration that exercises the
modified rule.  Regression tests are important because unrelated kbuild
changes can easily reintroduce missing prerequisites or shell fragments that
are harmless to normal make but fatal to the generated script.

Replay equivalence
------------------

The strongest validation is to compare a normal build with a replayed script
build from the same dry-run snapshot.  Use fixed build metadata so expected
timestamps and build version strings do not cause false differences::

    export CCACHE_DISABLE=1
    export KBUILD_BUILD_VERSION=1
    export KBUILD_BUILD_TIMESTAMP='Mon Jan 1 00:00:00 UTC 2024'

    rm -rf $base $plain
    mkdir -p $base

    make O=$base defconfig
    make O=$base -n > $base/make.sh
    chmod +x $base/make.sh
    bash -n $base/make.sh

    cp -a $base $plain

    make -j$(nproc) O=$plain
    ( cd $base && ./make.sh )

    for f in \
        vmlinux \
        vmlinux.unstripped \
        System.map \
        modules.builtin \
        modules.builtin.modinfo \
        arch/x86/boot/bzImage
    do
        cmp "$plain/$f" "$base/$f"
    done

The fixed metadata environment must be used for configuration, dry-run
generation, script replay, and the normal make build.  Some generated headers,
including ``include/generated/utsversion.h``, may be created while generating
the script.  If ``make -n`` uses different metadata from the two real builds,
the comparison can fail even when the replayed commands are correct.

For configurations that generate debug information, the two object trees may
also be embedded in DWARF or BTF data.  Add debug-prefix maps for both object
trees when comparing full binaries from different directories::

    mapflags="-fdebug-prefix-map=$base=. -fdebug-prefix-map=$plain=."
    export KCFLAGS="$mapflags"
    export KAFLAGS="$mapflags"

The script is replayed in ``$base`` because the generated commands refer to the
object tree used during dry-run generation.  The normal make build uses
``$plain``, which is a snapshot copy of that same tree immediately after script
generation.  ``bash -x`` is useful for replay validation because the script is
otherwise quiet for many commands.

Validation matrix
-----------------

Use a small matrix so both cheap smoke tests and broader coverage run
regularly:

* ``allnoconfig`` replay equivalence.  This is fast and covers the final
  vmlinux link, ``System.map``, built-in module metadata, and boot image
  post-processing with a small configuration.

* ``defconfig`` replay equivalence.  This exercises modules, objtool, ASN.1
  generated sources, certificate generation, x86 boot compression, and more
  cross-directory prerequisites.

* Config-toggle replay equivalence for options that change the build graph or
  generated metadata.  Useful combinations include:

  * ``CONFIG_MODVERSIONS=y``, ``CONFIG_MODULE_SIG=y``,
    ``CONFIG_MODULE_SIG_ALL=y``, and ``CONFIG_FUNCTION_TRACER=y``.  This
    covers genksyms ``.cmd`` metadata, ``modpost`` CRC handling, module
    signing, objtool, and ftrace command paths.  Use a deterministic
    ``CONFIG_MODULE_SIG_KEY`` for binary comparisons; the default
    ``certs/signing_key.pem`` is generated during the build and intentionally
    contains fresh key material.

  * ``CONFIG_DEBUG_INFO_BTF=y`` and ``CONFIG_DEBUG_INFO_BTF_MODULES=y`` when
    ``pahole`` is available.  This covers BTF generation and module BTF
    post-processing.  Use debug-prefix maps for replay-equivalence tests so
    path differences between the two object trees do not change DWARF or BTF
    output.

  * Alternate kernel and initramfs compression choices such as
    ``CONFIG_KERNEL_XZ=y`` and ``CONFIG_INITRAMFS_COMPRESSION_XZ=y``.  This
    covers boot image compression command paths and initramfs generation.

  * Instrumentation options such as ``CONFIG_KASAN=y``, ``CONFIG_UBSAN=y``,
    and ``CONFIG_KCOV=y``.  These options alter compiler flags and generated
    object coverage without necessarily changing the final target list.

* Generation-only tests for larger configurations such as ``allyesconfig`` or
  architecture-specific debug configurations.  At least check that ``make -n``
  succeeds and ``bash -n`` accepts the generated script.

* Generation-only tests for several deterministic ``randconfig`` seeds.  These
  are cheap because they stop after script generation, but they still cover
  build-graph combinations that fixed configurations may miss.

* Selected target tests for areas touched by a change, for example
  ``vmlinux``, ``modules``, ``arch/x86/boot/bzImage``, ``tools/objtool``,
  or an external module build with ``M=``.

* Toolchain variants where available: ``LLVM=1``, cross builds with
  ``ARCH=`` and ``CROSS_COMPILE=``, and builds with or without ccache.

* Documentation builds for this file after documentation changes::

      make SPHINXDIRS=kbuild htmldocs

Failure triage
--------------

Common failure modes point to different rule problems:

* ``make -n`` fails with ``No rule to make target``: a generated prerequisite
  is needed while walking the dry-run graph.  Add dry-run-only prerequisite
  materialization or mark the cross-directory input phony in the makefile
  that consumes it.

* ``bash -n`` fails: the dry-run output is not valid shell.  Look for
  unquoted newlines, incomplete conditionals, or make output mixed into a
  recipe.

* Replay fails with ``command not found`` for a status line such as
  ``DESCEND``: a recipe prefixed with ``+`` executed in dry-run mode and wrote
  human-readable output into the script.  Suppress that output or turn it into
  a shell comment for dry-run mode.

* Replay fails only for tools builds: many tools makefiles assume their source
  directory as the current working directory.  Either keep the tools recursive
  make command as a real replay command, or make the emitted commands
  standalone by using absolute source paths.

* Replay succeeds but binary comparison differs: first check build metadata
  such as ``KBUILD_BUILD_TIMESTAMP``, ``KBUILD_BUILD_VERSION``,
  ``KBUILD_BUILD_USER``, and ``KBUILD_BUILD_HOST``.  If those are fixed,
  compare generated headers, ``System.map``, and link command order.

Troubleshooting
===============

``No rule to make target ...``
-------------------------------

The current makefile is seeing a missing prerequisite during dry-run graph
walk.  If that file is generated elsewhere in the full build, add a dry-run
rule or ``PHONY`` entry in the makefile that needs it.

``command not found`` for a kbuild status line
----------------------------------------------

The generated script contains text that was meant for humans, not the shell.
Common causes are GNU make directory messages or kbuild quiet output.  Suppress
that output in dry-run mode.

The script succeeds despite errors
----------------------------------

Check whether a helper script prints commands but drops the shell control flow
that tested their exit status.  The replay script should preserve the failing
command or an equivalent failure check.

The script fails only for ``O=...`` builds
------------------------------------------

The command may be using a source-tree path relative to the object tree, or
vice versa.  Generate with ``O=$objtree`` and run the script from ``$objtree``.
Use ``$(srctree)`` for source files and object-tree-relative paths for
generated outputs.
