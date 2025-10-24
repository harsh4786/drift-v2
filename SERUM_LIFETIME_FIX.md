# Fix for Lifetime Issue in Pubkey Conversion

## The Problem
```rust
Pubkey::new_from_array(*array_ref![cast_slice::<u64, u8>(&identity(self.own_address)), 0, 32])
```

**Error:** Temporary value dropped while borrowed

The `identity(self.own_address)` creates a temporary value that gets dropped before we can use it.

## The Solution

We need to bind the temporary value to a variable so it lives long enough:

```rust
{
    let own_address = identity(self.own_address);
    Pubkey::new_from_array(*array_ref![cast_slice::<u64, u8>(&own_address), 0, 32])
}
```

## Alternative: Direct Conversion (SIMPLEST)

Since `self.own_address` is `[u64; 4]` and we need `[u8; 32]`, we can use `bytemuck::cast`:

```rust
Pubkey::new_from_array(bytemuck::cast(identity(self.own_address)))
```

This is the cleanest solution because:
- `bytemuck::cast` converts `[u64; 4]` directly to `[u8; 32]`
- No temporary lifetime issues
- No need for `cast_slice` or `array_ref`
- One line, zero allocations

## Recommended Fix

In your serum_dex fork at `dex/src/state.rs` line 580:

**Replace:**
```rust
Pubkey::new(cast_slice(&identity(self.own_address) as &[_]))
```

**With:**
```rust
Pubkey::new_from_array(bytemuck::cast(identity(self.own_address)))
```

This should work perfectly since:
- `self.own_address` is type `[u64; 4]` (32 bytes)
- `identity()` just returns the value unchanged
- `bytemuck::cast` safely converts `[u64; 4]` → `[u8; 32]`
- `Pubkey::new_from_array` takes `[u8; 32]`

And `bytemuck` is already a dependency of serum_dex!
