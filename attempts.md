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

### Attempt 2: default MatMulNBits N-tile swizzle in groups of 8 ✅

- **Change**: Increased the N-tile swizzle group from 4 to 8 in the default `MatMulNBits` path.
- **Prefill**: 615.281 token/sec (baseline: 613.955) — +0.22%
- **Decode**: 50.8542 token/sec (baseline: 48.9975) — +3.79%; +0.20% vs attempt 1
- **Result**: Slight further decode improvement with no added complexity. Kept and committed.
- **Hypothesis**: Larger N-tile rotation improves locality/scheduling a little more for the recurring 1024/2048/6144 decode projection shapes.

### Attempt 3: default MatMulNBits N-tile swizzle in groups of 16 ✅

- **Change**: Increased the N-tile swizzle group from 8 to 16 in the default `MatMulNBits` path.
- **Prefill**: 615.455 token/sec (baseline: 613.955) — +0.24%
- **Decode**: 51.0543 token/sec (baseline: 48.9975) — +4.20%; +0.39% vs attempt 2
- **Result**: Further decode improvement with the same simple implementation. Kept and committed.
- **Hypothesis**: The decode workload benefits from a wider rotated N-tile issue order, likely because adjacent projection tiles share enough access patterns for better scheduling/cache behavior.
