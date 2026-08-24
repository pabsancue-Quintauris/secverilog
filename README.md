# SecVerilog: build, install, and quick check

This repository has no setup or installation guide of its own. `verilog-0.9.6/README.txt`, `INSTALL`, and `QUICK_START.txt` all describe generic upstream Icarus Verilog usage and say nothing about the security-type extension this fork actually exists for. This file documents what those three do not: how to build the compiler, where it needs to be installed, and how to confirm it works.

## What this is

SecVerilog ([github.com/aferr/secverilog](https://github.com/aferr/secverilog)) is a research fork of Icarus Verilog 0.9.6 that adds security-label type annotations to Verilog declarations (`input {L} clk;`, `reg [15:0] {H} data;`) and checks information-flow policies at compile time. A design that violates its declared labels is reported as a specific failing assignment rather than accepted silently. The security-type logic lives in `verilog-0.9.6/sectypes.cc`/`.h`, `typecheck.cc`, `QuantExpr.cc`/`.h`, `QEVisitor.cc`, and `QESubVisitor.cc`; the grammar is `verilog-0.9.6/parse.y` (bison) plus `lexor.lex` (flex) and `lexor_keyword.gperf` (gperf).

Type-checking itself is delegated to the Z3 SMT solver: the compiler emits an SMT-LIB2 file, and Z3 decides satisfiability of each generated obligation.

## Prerequisites

- GNU Make, an ISO C++ compiler, bison, flex, gperf, GNU readline (with development headers), termcap/ncurses (with development headers), autoconf (at least 2.53, since no `configure` script is checked in).
- The Z3 SMT solver, available on `PATH` as `z3`, used at type-check time, not at build time.

## Building

No `configure` script is checked into `verilog-0.9.6/`, and rebuilding it hits five problems that are consequences of this being roughly 2013-era code, unrelated to the security-type extension itself. Apply all five before running `configure`.

```bash
cd verilog-0.9.6
```

**1. `autoheader` rejects two-argument `AC_DEFINE` calls.** In `configure.in`, add a third (description) argument to each of the five calls below:

```
AC_DEFINE([HAVE_LROUND], [1])                 ->  AC_DEFINE([HAVE_LROUND], [1], [Define if lround is available])
AC_DEFINE([HAVE_LLROUND], [1])                ->  AC_DEFINE([HAVE_LLROUND], [1], [Define if llround is available])
AC_DEFINE([HAVE_NAN], [1])                    ->  AC_DEFINE([HAVE_NAN], [1], [Define if nan is available])
AC_DEFINE([HAVE_FMIN], [1])                   ->  AC_DEFINE([HAVE_FMIN], [1], [Define if fmin is available])
AC_DEFINE([HAVE_FMAX], [1])                   ->  AC_DEFINE([HAVE_FMAX], [1], [Define if fmax is available])
AC_DEFINE([UINT64_T_AND_ULONG_SAME], [1])     ->  AC_DEFINE([UINT64_T_AND_ULONG_SAME], [1], [Define if uint64_t and unsigned long are the same type])
```

**2. `verinum.h` uses `uint64_t` without including it.** Add, near its other includes:

```cpp
#include <cstdint>
```

**3. The generated parser expects a header name modern bison does not produce.** `parse.cc` (generated from `parse.y`) contains `#include "parse.hh"`, but a current bison writes the header as `parse.h`. In `Makefile.in` (and `Makefile`, if `configure` has already run), change the rule:

```make
parse.h: parse.cc
	mv parse.cc.h parse.h 2>/dev/null || mv parse.hh parse.h 2>/dev/null || true
	cp -f parse.h parse.hh
```

**4. `libstdc++`'s `std::set` now requires a `const`-callable comparator.** Add `const` to both comparator classes' `operator()`:

```cpp
// PExpr.h
class ExprComparator {
public:
    bool operator()(const PExpr* e1, const PExpr* e2) const { ... }   // add const
};

// sectypes.h
class HypothesisComparator {
public:
    bool operator()(const Hypothesis* h1, const Hypothesis* h2) const { ... }   // add const
};
```

**5. GCC 10+ defaults to `-fno-common`, and `driver/cfparse_misc.h` declares a global without `extern`.** Add `-fcommon` to `driver/Makefile.in`'s `CFLAGS`:

```make
CFLAGS = -Wall -Wshadow -g -O2 -fcommon
```

With all five applied:

```bash
autoreconf -fi
./configure --prefix=/path/to/install
```

**Only build the core compiler and its driver, not the simulation runtime.** `vvp/` (the VVP runtime) has its own, unrelated `parse.y` for the VVP assembly language, and fails to build under a current bison/GCC for the same class of union/YYSTYPE reasons as the main grammar, but is not needed for type-checking at all:

```bash
make ivl
make -C ivlpp
make -C driver
```

`make -j` (parallel) is unsafe specifically for the `parse.cc`/`parse.h` pair: this Makefile's own comment already flags it as a known hazard (`pr3462585`). If a parallel build produces errors like `'YYSTYPE' has no member named 'numb'`, force a clean, serial regeneration of just that pair first, then rebuild:

```bash
rm -f parse.cc parse.h parse.hh parse.o
make parse.h
make ivl
```

## Installing

`driver/iverilog` (the wrapper users normally invoke) looks for `ivl` and `ivlpp` at the `--prefix` path given to `configure`, specifically `<prefix>/lib/ivl/`, not in the build directory. A full `make install` will fail, since it depends on `all`, which depends on the broken `vvp` build. Install just the two binaries actually needed by hand:

```bash
mkdir -p /path/to/install/lib/ivl
cp ivl /path/to/install/lib/ivl/ivl
cp ivlpp/ivlpp /path/to/install/lib/ivl/ivlpp
```

## Quick check

**Do not use `driver/iverilog` for a first check.** It requires an installed `vvp.conf`/default-target-module configuration that a partial install does not provide, even just to reach the `-z` (type-check-only) flag. Invoke the core compiler directly instead; `-z` is one of its own options (see `ivl -h`), not something the driver adds:

```bash
cat > /tmp/probe.v <<'EOF'
module probe();
    reg {L} lo;
    reg {H} hi;

    always @(*) begin
        lo = 0;      // safe: L assigned from a constant
    end

    always @(*) begin
        lo = hi;     // unsafe: H flowing into L
    end
endmodule
EOF

./ivl -z /tmp/probe.v      # writes /tmp/probe.z3 next to the input file
z3 -smt2 /tmp/probe.z3
```

Expected output: `unsat` (the safe assignment, no violation) followed by `sat` (the unsafe assignment, a real violation exists). If both lines print, the build is working.

The `.z3` output path is derived from the compiled module's own recorded source file name, truncated at its first `.`, not from the current directory. It lands next to the input file, which matters if that surprises you when it is not in your working directory.

To identify exactly which source line a `sat` result corresponds to when checking a real design with more than one obligation, pair z3's output (one line per `(check-sat)`, in file order) against the `; <expression> @<file>:<line>` comment immediately preceding each one in the `.z3` file.

## A note on the bundled `test/` regression suite

None of `test/*.v` exercises the plain, default two-point (`L`/`H`) lattice by itself. `sound.v` and `deptype.v` need a `Par`-named custom lattice; `diamond.v` needs a `Domain`/four-point diamond lattice; the `quant_*.v` files need quantified dependent labels. All of these require a `-l latticefile` that is not included in this repository, so none of them will run as-is; the probe above is the fastest way to confirm a build without one. The project's own `test.rb` (a thin wrapper running exactly `iverilog -z file.v && z3 -smt2 file.z3` over every `test/*.v` and diffing against its `.expected`) requires Ruby, not installed by default on every system; if available, it is otherwise a faithful automation of the same check described above, once a lattice file is supplied.

## Language subset

This is Verilog-2001 plus security labels in `{...}` immediately after a type/range and before the identifier on a declaration (`input {L} clk;`, `output reg [15:0] {L} out;`, `reg[1:0] {Par x} x;`). There is no struct support, no `always_ff`, no `foreach`, no `` `clog2 ``, and no SystemVerilog fill/cast literals (`'0`, sized casts like `($clog2(N))'(expr)`). A SystemVerilog design has to be rewritten into this subset before any labels can be added; it is not accepted as-is.
