# Optimization Notes

Current branch: `codex/interpreter-loop-fusions`.

The sieve microbenchmark is mostly sensitive to interpreter dispatch, branch
handling, and simple RAM load/store loops. The current fused handlers for
`ADDI + ADDI + BNE` and `ADDI + branch` improve the benchmark by about 1.29x
against `origin/master` on this machine.

## Playbook

1. Add store-loop fusions first. Target `mark_loop` shapes such as
   `sb zero, 0(ptr); add ptr, ptr, step; sub tmp, ptr, base; blt tmp, limit`.
2. Add `lbu/lw + beq/bne` test-branch fusions for sieve filter checks.
3. Add indexed memory fusions such as `add addr, base, index; lbu/lw/sb/sw`.
4. Generalize branch fusions only after the longer hot loop shapes are covered.
5. Keep new fusions interpreter-only until measured; add JIT/T2C support only
   for proven broadly useful patterns.
6. Benchmark each patch separately with the fib/sieve ELF, `hyperfine`, and
   preferably `perf stat`.

Avoid mixing this optimization work into the ACT test-suite branch or PR.

## Benchmark Log

Benchmark command shape:

`hyperfine --warmup 1 --runs 10 '<rv32emu> -q /tmp/rv32emu-bench/fibsieve.elf'`

All runs use the same fib/sieve ELF with `OUTER=16000`.

| Step | Mean time | Relative to master | Notes |
| --- | ---: | ---: | --- |
| `origin/master` at `7b4e872` | `2.520 s +/- 0.089` | `1.00x` | Local baseline |
| `fuse13/fuse14` branch point | `2.010 s +/- 0.096` | `1.25x` | `ADDI+ADDI+BNE`, `ADDI+branch` |
| `fuse15` store-loop fusion | `1.362 s +/- 0.060` | `1.85x` | Adds `SB+ADD+SUB+BLT`; checksum preserved |
| `fuse16` load-test fusion | `1.190 s +/- 0.070` | `2.12x` | Adds `LBU+BEQ/BNE`; checksum preserved |
| `fuse17` indexed load-test fusion | `1.110 s +/- 0.039` | `2.27x` | Adds `ADD+LBU+BEQ/BNE`; checksum preserved |
| `fuse18` Fibonacci loop fusion | `1.029 s +/- 0.050` | `2.45x` | Adds exact `ADD+SW+mv+mv+ADDI+ADDI+BNE`; checksum preserved |
| `fuse19` byte-fill countdown fusion | `998.6 ms +/- 53.1` | `2.52x` | Adds `LI+SB+ADDI+ADDI+BNE`; checksum preserved |

`fuse15` is worth keeping for the sieve workload. It targets `mark_loop`
directly and also gives a large incremental gain over `fuse13/fuse14`
(`1.48x` faster in the three-way benchmark).

`fuse16` is also worth keeping. In an incremental run it improved from
`1.411 s +/- 0.066` for the `fuse15` binary to `1.190 s +/- 0.070`, a
`1.19x` gain.

`fuse17` is correct but lower impact. In an incremental run it improved from
`1.155 s +/- 0.056` for the `fuse16` binary to `1.110 s +/- 0.039`, a `1.04x`
gain. Keep it if code complexity remains low.

`fuse18` is benchmark-specific but still measurable. In an incremental run it
improved from `1.117 s +/- 0.041` for the `fuse17` binary to
`1.029 s +/- 0.050`, a `1.09x` gain.

`fuse19` is correct but marginal. In an incremental run it improved from
`1.025 s +/- 0.057` for the `fuse18` binary to `998.6 ms +/- 53.1`, a `1.03x`
gain. This is likely below the threshold for a broad upstream change unless the
pattern can be generalized cleanly.

Final cross-check after `fuse19`: current branch measured `995.4 ms +/- 47.9`
against `2.468 s +/- 0.085` for `origin/master` at `7b4e872`, or `2.48x`
faster overall.
