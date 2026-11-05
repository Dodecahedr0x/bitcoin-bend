# Agents

This repository is a Bitcoin node written in Bend 2. Several agents share it. Read this file before editing.

The language rules below are the Bend project snippet from <https://bend2.dev/agents/bend.md>. The later sections say how agents divide the work in this repository.

## Bend

- Before starting a new session writing Bend, run `bend version` and read all of `bend guide`; read `bend guide shaders` before parallel or graphical work.
- State each requirement as a law in `LAWS.bend` before writing its code, and put the law diff up for review before proving it.
- Never delete or weaken an accepted law to pass a check; prove every law in `PROOF.bend` and require `bend PROOF.bend` to pass.
- Lay out `main.bend`, `LAWS.bend`, and `PROOF.bend` as Bend's demos do: a header comment with purpose, expected output, and run command; laws through an import alias; one proof def per law.
- Keep decisions in pure functions over `Data` with the IO in a thin shell; match every `Result`, close every handle on every branch, bound loops with `Nat` fuel, and keep `@unsafe` out of anything a law depends on.
- Define a def above its first caller, put the shrinking argument first, match only parameters and pattern-bound variables, and annotate every operator and every `<-` bind; check with `--check-only` after each def.
- Fork with `a b = f(x) g(y)` over halves of equal work, splitting on a `Nat` depth with the midpoint at `i + (1 << p)`; leaves run a tail loop over 8 to 64 items, and a head-and-tail recursion runs on one core.
- Use `Array` for data that is indexed or updated in place, a tree for data a fork splits, and a list for data a leaf walks once.
- Write `+` only for a second use and `Array.clone` only for a second array; count `term_keep`, `ctr_take`, and `rfc_seal` inside your defs in the emitted C, and treat a new one as a regression.
- Build the deployment target; run it with `+RTS --threads 1 --gpu off` and with all cores, confirm equal output, and report the speedup.
- Run `bend PROOF.bend --verdict` last; report commands, outputs, and open holes, since a passing proof covers only its stated laws.

If `bend` is missing, install it from <https://bend2.dev/notes/install-bend2-from-source/>. Use <https://bend2.dev/llms.txt> to find a current example, and prefer that example's Markdown over memory. The installed compiler wins when it disagrees with an example: record `bend version` and check locally.

A runnable program is `main.bend`. A pure subsystem is `job.bend`. Both sit beside `LAWS.bend` and `PROOF.bend`. The root `PROOF.bend` imports every subsystem proof. `bend PROOF.bend` refuses a proof file that sits beside `LAWS.bend` without importing it.

## What this repository builds

Bitcoin Core 32.1, tag `v32.1`, is the parity target. [Issue 1](https://github.com/Dodecahedr0x/bitcoin-bend/issues/1) records the target and the boundaries: bitcoind, bitcoin-cli, bitcoin-tx, bitcoin-wallet, bitcoin-util, REST, ZMQ, and the mining IPC schema. bitcoin-qt, the libbitcoinkernel C ABI, and Erlay sketch messages are outside that boundary. `sendtxrcncl` is inside it.

[Issue 88](https://github.com/Dodecahedr0x/bitcoin-bend/issues/88) is the completion list. The node is finished when every issue linked there is closed, the differential test against bitcoind 32.1 passes, and the block-connection median is at or below bitcoind on the same machine.

Consensus and policy numbers come from the `v32.1` sources or from a bitcoind 32.1 run. A limit you remember is not a law until that source confirms it.

## Sharing the work

Take one open issue. Read its Task and Done When with `gh issue view N --repo Dodecahedr0x/bitcoin-bend`. Skip an issue another agent has already claimed in a comment or an open pull request.

Before the first edit, comment on the issue: `Taking this.` Name the branch you will use, `issue-N-short-name`. One issue, one branch, one pull request. Link the pull request to the issue.

Stay inside that issue's files. Add a new directory rather than rewriting a module another open pull request owns. The shared files are the root `PROOF.bend`, the CI workflow, and the chain-parameter module. Touch those only to add your import or your chain case, and keep both sides when you rebase.

Publish the law diff before the proofs. The first commit on the branch is `LAWS.bend` (and the pure types those laws mention). Comment on the issue with the counterexample each new law rejects. A law that is still true of a no-op implementation is too weak; strengthen it before proving. After that comment is on the issue, later commits may add `job.bend` and `PROOF.bend`.

An accepted law is one already on the default branch, or one whose issue is closed. Extend a law by naming a stronger property. Leave an unfinished proof as a hole and report it. A green `bend PROOF.bend` after a deleted law is a failed task.

When your change breaks another module's proof, repair your implementation or add a new law that states the extra requirement. Leave their law text in place.

## Handing off

Comment on the issue before you stop, including when you finish. Include:

- `bend version`
- the issue number and the files you touched
- the commands you ran, and their output: `bend PROOF.bend`, `bend PROOF.bend --verdict`, `--check-only`, and the native runs
- one-thread and all-core results, which must match, and the speedup when the issue is on the hot path
- open holes, named by law
- anything the next agent must not redo

Close the issue only when every Done When bullet is true and the comment shows the command output. Leave [issue 88](https://github.com/Dodecahedr0x/bitcoin-bend/issues/88) open until every other issue is closed.

If the Done When disagrees with bitcoind 32.1, stop and comment the evidence. Change the implementation to match the binary, or ask for a spec change. Do not edit the issue so a weaker program looks finished.

## Checks

Run these from the repository root before you say a requirement holds:

```sh
bend version
bend PROOF.bend
bend PROOF.bend --verdict
bend path/to/file.bend --check-only
bend path/to/main.bend -o main
./main +RTS --threads 1 --gpu off
./main +RTS --threads <all-cores> --gpu off
```

Put runtime flags inside `+RTS` … `-RTS`, and end that block before program arguments. Keep `--gpu off` on both CPU measurements. A GPU call is `f!(x)`. When the machine has a GPU and the issue is on the signature or script path, also run with `--gpu 1GB` and record that the outputs still match.

For a performance claim, warm each mode, take five runs, and report the median and the spread beside the bitcoind 32.1 median on the same machine. The proof checker stays out of the timed region.

Runtime behavior that a law cannot see — sockets, files, process failure — needs a test against bitcoind 32.1. Close every handle on every branch, including `Fail`. Bound an IO loop with `Nat` fuel.

After `bend file.bend -o file.c`, count `term_keep`, `ctr_take`, and `rfc_seal` inside your defs, not in the runtime's. A new count is a regression to remove or to explain in the issue comment.

## Checker habits

These are the mistakes that waste a shared CI run:

- A def calls something written below it. Helpers come first. Mutual recursion becomes one def with a selector.
- The recursive argument is not first, or the recursive call does not receive a smaller piece of that argument.
- `match` inspects a call result. Pass that value to a helper and match the helper's parameter. There is no `if`. Branch on `True{}` and `False{}`.
- An operator lacks its type, as in `(a + b : U32)`. A `<-` binding lacks its type. A type argument is implicit.
- A plain binder is used twice. `+` marks a second use of `Data`. Functions, arrays, and handles cannot take `+`. `List<&2, U32>` is the reusable list of bytes.
- A fork is one element against the rest of a list. Split equal work on a `Nat` depth. A leaf is a tail loop over 8 to 64 items.
- `@unsafe` appears in a function a law reaches. The accept loop may be `@unsafe`. Anything a law imports may not.
