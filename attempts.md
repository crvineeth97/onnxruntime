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

### Attempt 4: default MatMulNBits N-tile swizzle in groups of 32 ❌

- **Change**: Increased the N-tile swizzle group from 16 to 32 in the default `MatMulNBits` path.
- **Prefill**: 614.004 token/sec (baseline: 613.955) — unchanged
- **Decode**: 49.6973 token/sec (baseline: 48.9975) — +1.43%; -2.66% vs attempt 3
- **Result**: Regressed from the current best. Reverted with `git stash`.
- **Hypothesis**: A 32-tile rotation is too wide and disrupts the locality/scheduling effect that helped at group size 16.

### Attempt 5: default MatMulNBits group16 rotate by 2 ❌

- **Change**: Kept the N-tile swizzle group at 16 but changed the rotation from +1 tile to +2 tiles.
- **Prefill**: 613.787 token/sec (baseline: 613.955) — unchanged
- **Decode**: 49.7473 token/sec (baseline: 48.9975) — +1.53%; -2.56% vs attempt 3
- **Result**: Regressed from the current best. Reverted with `git stash`.
- **Hypothesis**: The +1 rotation is important; skipping an extra neighboring tile loses the beneficial locality/scheduling behavior.

### Attempt 6: default MatMulNBits group16 reverse rotation ❌

- **Change**: Kept the N-tile swizzle group at 16 but rotated by -1 tile instead of +1.
- **Prefill**: 615.382 token/sec (baseline: 613.955) — +0.23%
- **Decode**: 50.8033 token/sec (baseline: 48.9975) — +3.69%; -0.49% vs attempt 3
- **Result**: Worse than the current best. Reverted with `git stash`.
- **Hypothesis**: Direction matters; the forward +1 issue order is better for this workload than wrapping from the end of each group.

### Attempt 7: default MatMulNBits group16 rotate by 8 ❌

- **Change**: Kept the N-tile swizzle group at 16 but rotated by half a group (+8 tiles).
- **Prefill**: 615.499 token/sec (baseline: 613.955) — +0.25%
- **Decode**: 50.4529 token/sec (baseline: 48.9975) — +2.97%; -1.18% vs attempt 3
- **Result**: Worse than the current best. Reverted with `git stash`.
- **Hypothesis**: Half-group rotation separates neighboring tiles too much, reducing the locality/scheduling benefit from the +1 rotation.
