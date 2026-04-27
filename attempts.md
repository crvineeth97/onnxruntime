# WebGPU swizzle attempts

## Baseline

- **Commit**: f5689c8043
- **Prefill**: 613.955 token/sec
- **Decode**: 48.9975 token/sec
- **Profile note**: Decode is dominated by `MatMulNBits`; the top five `MatMulNBits` shader shapes account for about 80.9% of profiled decode shader time.

### Attempt 1: default MatMulNBits N-tile swizzle in groups of 4 ✅

- **Change**: Reordered default `MatMulNBits` N-tile traversal within groups of 4 while preserving output indices.
- **Prefill**: 614.876 token/sec (baseline: 613.955) — +0.15%
- **Decode**: 50.7529 token/sec (baseline: 48.9975) — +3.58%
- **Result**: Meaningful decode improvement. Kept and committed.
- **Hypothesis**: Rotating neighboring N tiles slightly improves decode scheduling/cache behavior for the dominant small-M `MatMulNBits` shader shapes without changing per-workgroup math.
