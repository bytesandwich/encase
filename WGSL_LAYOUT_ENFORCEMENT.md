# How encase Enforces WGSL Layout Correctness

A code-driven explanation of encase's WGSL memory layout validation mechanisms.

---

## 1. Core Invariants

encase enforces WGSL layout rules through compile-time const assertions and metadata computation. The key invariants are:

### 1.1 Alignment Rules

**WGSL Rule:** Every type has an alignment requirement. Offsets and sizes must respect these alignments.

**Enforcement:** `src/core/alignment_value.rs`
```rust
pub struct AlignmentValue(NonZeroU64);

impl AlignmentValue {
    pub const fn new(val: u64) -> Self {
        if !val.is_power_of_two() {
            panic!("Alignment must be a power of 2!");
        }
        // ... creates NonZeroU64
    }
}
```

Alignments are computed per-type:
- Scalars (f32, u32, i32): 4 bytes (`src/types/scalar.rs:18`)
- Vectors: Next power of 2 of size (`src/types/vector.rs:120`)
- Matrices: Next power of 2 of column size (`src/types/matrix.rs:137`)
- Arrays: Element alignment (`src/types/array.rs:26`)
- Structs: Maximum field alignment (`derive/impl/src/lib.rs:666`)

### 1.2 Size Rounding

**WGSL Rule:** Sizes must be rounded up to alignment boundaries.

**Enforcement:** `src/core/alignment_value.rs:75-77`
```rust
pub const fn round_up_size(&self, n: SizeValue) -> SizeValue {
    SizeValue::new(self.round_up(n.get()))
}
```

Used in:
- Array stride calculation (`src/types/array.rs:29`)
- Struct size calculation (`derive/impl/src/lib.rs:679`)

### 1.3 Array Stride Requirements

**WGSL Rule:** Array stride must be a multiple of element alignment. For uniform buffers, stride must be ≥16 bytes.

**Enforcement:** `src/types/array.rs:23-60`
```rust
impl<T: ShaderType + ShaderSize, const N: usize> ShaderType for [T; N] {
    const METADATA: Metadata<Self::ExtraMetadata> = {
        let alignment = T::METADATA.alignment();
        let el_size = SizeValue::from(T::SHADER_SIZE);
        
        // Stride = round_up(element_size, alignment)
        let stride = alignment.round_up_size(el_size);
        // ...
    };

    const UNIFORM_COMPAT_ASSERT: fn() = || {
        crate::utils::consume_zsts([
            <T as ShaderType>::UNIFORM_COMPAT_ASSERT(),
            if let Some(min_alignment) = Self::METADATA.uniform_min_alignment() {
                const_panic::concat_assert!(
                    min_alignment.is_aligned(Self::METADATA.stride().get()),
                    "array stride must be a multiple of ",
                    min_alignment.get(),
                    " (current stride: ",
                    Self::METADATA.stride().get(),
                    ")"
                );
            },
        ]);
    };
}
```

The uniform minimum alignment is 16 bytes (`src/core/traits.rs:5`):
```rust
const UNIFORM_MIN_ALIGNMENT: AlignmentValue = AlignmentValue::new(16);
```

### 1.4 Struct Layout Constraints

**WGSL Rule:** 
- Fields must be aligned to their type's alignment
- Struct size must be rounded up to struct alignment
- For uniform buffers, structs have additional spacing requirements

**Enforcement:** `derive/impl/src/lib.rs:665-689`

The derive macro generates:
```rust
const METADATA: Metadata<Self::ExtraMetadata> = {
    // 1. Struct alignment = max(field alignments)
    let struct_alignment = AlignmentValue::max([ /* field alignments */ ]);
    
    // 2. Compute field offsets and padding
    let extra = {
        let mut paddings = [0; N];
        let mut offsets = [0; N];
        let mut offset = 0;
        // For each field:
        //   offsets[i] = alignment.round_up(offset);
        //   offset += field_size;
        //   paddings[i-1] = alignment.padding_needed_for(offset);
        // ...
    };
    
    // 3. Struct size = round_up(last_offset + last_size, struct_alignment)
    let min_size = {
        let mut offset = extra.offsets[N - 1];
        offset += last_field_min_size;
        SizeValue::new(struct_alignment.round_up(offset))
    };
    
    Metadata {
        alignment: struct_alignment,
        has_uniform_min_alignment: true,
        min_size,
        // ...
    }
};
```

### 1.5 Uniform vs Storage Buffer Rules

**Key Difference:** Uniform buffers enforce the 16-byte minimum alignment constraint.

**Storage buffers:**
- No minimum alignment requirement beyond type alignment
- Can use arrays with stride < 16

**Uniform buffers:**
- `has_uniform_min_alignment: true` is set for arrays and structs
- Array stride must be ≥16 bytes
- Struct fields containing types with `has_uniform_min_alignment` must respect 16-byte boundaries

**Code locations:**
- Arrays set `has_uniform_min_alignment: true` (`src/types/array.rs:39`)
- Structs set `has_uniform_min_alignment: true` (`derive/impl/src/lib.rs:684`)
- Vectors set `has_uniform_min_alignment: false` (`src/types/vector.rs:124`)
- Matrices set `has_uniform_min_alignment: false` (`src/types/matrix.rs:143`)
- Runtime-sized arrays set `has_uniform_min_alignment: true` (`src/types/runtime_sized_array.rs:138`)

**Note on matrices:** Matrices are laid out as arrays of column vectors but don't require 16-byte stride validation. This is because WGSL treats matrix columns as vectors, not as array elements subject to uniform array stride rules.

---

## 2. ShaderType Metadata

### 2.1 Metadata Structure

`src/core/traits.rs:7-13`
```rust
pub struct Metadata<E> {
    pub alignment: AlignmentValue,
    pub has_uniform_min_alignment: bool,
    pub min_size: SizeValue,
    pub is_pod: bool,
    pub extra: E,
}
```

Every type implementing `ShaderType` must define:
```rust
trait ShaderType {
    type ExtraMetadata;
    const METADATA: Metadata<Self::ExtraMetadata>;
    // ...
}
```

### 2.2 Metadata Computation for Primitive Types

**Scalars** (`src/types/scalar.rs:16-18`):
```rust
const METADATA: Metadata<()> = 
    Metadata::from_alignment_and_size(4, 4).pod();
```
- Alignment: 4
- Size: 4
- No extra metadata
- Marked as POD (plain old data)

**Vectors** (`src/types/vector.rs:118-129`):
```rust
const METADATA: Metadata<()> = {
    let size = SizeValue::from(T::SHADER_SIZE).mul(N);
    let alignment = AlignmentValue::from_next_power_of_two_size(size);
    
    Metadata {
        alignment,
        has_uniform_min_alignment: false,
        min_size: size,
        is_pod: <[T; N] as ShaderType>::METADATA.is_pod(),
        extra: ()
    }
};
```
- vec2: size=8, alignment=8
- vec3: size=12, alignment=16
- vec4: size=16, alignment=16

**Matrices** (`src/types/matrix.rs:135-150`):
```rust
pub struct MatrixMetadata {
    pub col_padding: u64,
}

const METADATA: Metadata<MatrixMetadata> = {
    // Column is a vector of R elements
    let col_size = SizeValue::from(T::SHADER_SIZE).mul(R);
    let alignment = AlignmentValue::from_next_power_of_two_size(col_size);
    
    // Total size = C columns, each rounded to alignment
    let size = alignment.round_up_size(col_size).mul(C);
    let col_padding = alignment.padding_needed_for(col_size.get());
    
    Metadata {
        alignment,
        has_uniform_min_alignment: false,
        min_size: size,
        is_pod: <[T; R] as ShaderType>::METADATA.is_pod() && col_padding == 0,
        extra: MatrixMetadata { col_padding },
    }
};
```
- mat2x2: size=16, alignment=8 (2 columns of vec2)
- mat3x3: size=48, alignment=16 (3 columns of vec3)
- mat4x4: size=64, alignment=16 (4 columns of vec4)

### 2.4 Metadata Computation for Arrays

`src/types/array.rs:25-44`
```rust
pub struct ArrayMetadata {
    pub stride: SizeValue,
    pub el_padding: u64,
}

impl<T: ShaderType + ShaderSize, const N: usize> ShaderType for [T; N] {
    type ExtraMetadata = ArrayMetadata;
    const METADATA: Metadata<Self::ExtraMetadata> = {
        let alignment = T::METADATA.alignment();
        let el_size = SizeValue::from(T::SHADER_SIZE);
        
        // Stride must be multiple of element alignment
        let stride = alignment.round_up_size(el_size);
        let el_padding = alignment.padding_needed_for(el_size.get());
        
        // Total array size
        let size = stride.mul(N as u64);
        
        Metadata {
            alignment,
            has_uniform_min_alignment: true,
            min_size: size,
            is_pod: T::METADATA.is_pod() && el_padding == 0,
            extra: ArrayMetadata { stride, el_padding },
        }
    };
}
```

**Example:** `[f32; 10]`
- Element size: 4
- Element alignment: 4
- Stride: 4 (round_up(4, 4) = 4)
- Total size: 40
- **This fails uniform buffer validation** because stride (4) < 16

### 2.5 Metadata Computation for Structs

`derive/impl/src/lib.rs:481-528`

The derive macro computes struct metadata in several passes:

**Pass 1 - Collect field alignments:**
```rust
let alignments = field_data.iter().map(|data| data.alignment(root));
```

**Pass 2 - Compute offsets and paddings:**
```rust
let paddings = field_data.iter().enumerate().map(|(i, current)| {
    if !is_first {
        let alignment = current.alignment(root);
        
        // Round up current offset to field alignment
        out.extend(quote! {
            offsets[i] = #alignment.round_up(offset);
            
            // Padding before this field
            let padding = #alignment.padding_needed_for(offset);
            offset += padding;
            paddings[i-1] = padding;
        });
    }
    
    // Add field size to offset
    let size = current.size(root);
    out.extend(quote! {
        offset += #size;
    });
    
    if is_last {
        // Final padding to struct alignment
        out.extend(quote! {
            paddings[i] = struct_alignment.padding_needed_for(offset);
        });
    }
    
    out
});
```

**Pass 3 - Generate metadata:**
```rust
const METADATA: Metadata<StructMetadata<N>> = {
    let struct_alignment = AlignmentValue::max([field_alignments]);
    
    let extra = {
        // Compute offsets and paddings
        StructMetadata { offsets, paddings }
    };
    
    let min_size = {
        let mut offset = extra.offsets[N - 1];
        offset += last_field_min_size;
        SizeValue::new(struct_alignment.round_up(offset))
    };
    
    Metadata {
        alignment: struct_alignment,
        has_uniform_min_alignment: true,
        min_size,
        is_pod: false,
        extra,
    }
};
```

### 2.6 Field Order and Padding

**Field order matters** because offsets are computed sequentially:

```rust
#[derive(ShaderType)]
struct Example1 {
    a: f32,      // offset: 0, size: 4
    b: Vec4f,    // offset: 16 (rounded up from 4), size: 16
}
// Total size: 32

#[derive(ShaderType)]
struct Example2 {
    a: Vec4f,    // offset: 0, size: 16
    b: f32,      // offset: 16, size: 4
}
// Total size: 32 (rounded up from 20 to alignment 16)
```

Both have the same size but different internal layouts.

### 2.7 repr(C) and repr(align(N))

**encase ignores Rust repr attributes** because:

1. WGSL layout ≠ C layout
2. Metadata is computed from `ShaderType` implementations, not Rust type layout
3. The `#[shader(align(N))]` attribute controls WGSL alignment

**From `derive/impl/src/lib.rs:48-59`:**
```rust
fn alignment(&self, root: &Path) -> TokenStream {
    if let Some((alignment, _)) = self.align {
        let alignment = Literal::u64_suffixed(alignment as u64);
        quote! {
            #root::AlignmentValue::new(#alignment)
        }
    } else {
        let ty = &self.field.ty;
        quote! {
            <#ty as #root::ShaderType>::METADATA.alignment()
        }
    }
}
```

The `#[shader(align(N))]` attribute overrides the type's natural alignment.

### 2.8 Implicit vs Explicit Padding

**Implicit padding:** Computed automatically by encase
- Between struct fields (`derive/impl/src/lib.rs:502`)
- At end of struct (`derive/impl/src/lib.rs:523`)
- Between array elements (`src/types/array.rs:30`)

**Explicit padding:** Using `#[shader(size(N))]` attribute
- Increases field size without changing its underlying data
- Creates extra padding after the field (`derive/impl/src/lib.rs:90-96`)

Example:
```rust
#[derive(ShaderType)]
struct Padded {
    #[shader(size(16))]
    a: f32,      // Takes 16 bytes (12 bytes padding after)
    b: f32,      // offset: 16
}
```

---

## 3. Array Stride Validation (The Panic Source)

### 3.1 Where Array Stride is Computed

`src/types/array.rs:26-30`
```rust
const METADATA: Metadata<Self::ExtraMetadata> = {
    let alignment = T::METADATA.alignment();
    let el_size = SizeValue::from(T::SHADER_SIZE);
    
    let stride = alignment.round_up_size(el_size);
    // ...
}
```

**Stride formula:** `stride = round_up(element_size, element_alignment)`

### 3.2 The Panic Logic

`src/types/array.rs:46-60`
```rust
const UNIFORM_COMPAT_ASSERT: fn() = || {
    crate::utils::consume_zsts([
        <T as ShaderType>::UNIFORM_COMPAT_ASSERT(),
        if let Some(min_alignment) = Self::METADATA.uniform_min_alignment() {
            const_panic::concat_assert!(
                min_alignment.is_aligned(Self::METADATA.stride().get()),
                "array stride must be a multiple of ",
                min_alignment.get(),
                " (current stride: ",
                Self::METADATA.stride().get(),
                ")"
            );
        },
    ]);
};
```

**The panic occurs when:**
1. `Self::METADATA.uniform_min_alignment()` returns `Some(16)`
2. `Self::METADATA.stride().get()` is not a multiple of 16

**For arrays, this is always set:**
```rust
has_uniform_min_alignment: true,  // line 39
```

### 3.3 Why Validate During Single Value Serialization?

**The validation is NOT during serialization.** It's a **compile-time check** triggered when:

`src/core/buffers.rs:129`
```rust
impl<B: BufferMut> UniformBuffer<B> {
    pub fn write<T>(&mut self, value: &T) -> Result<()>
    where
        T: ?Sized + ShaderType + WriteInto,
    {
        T::assert_uniform_compat();  // <-- This triggers the panic
        self.inner.write(value)
    }
}
```

The `assert_uniform_compat()` call executes the const function `UNIFORM_COMPAT_ASSERT`, which panics if the type is invalid for uniform buffers.

**Why check single values?** Because in WGSL, **any type can be an array element**. Even when writing a single struct instance, if that struct could theoretically be used in an array within a uniform buffer, the layout must be compatible.

### 3.4 Example: Struct with Size 32 and Alignment 4

```rust
#[derive(ShaderType)]
struct MyStruct {
    a: f32,      // offset: 0, size: 4
    b: f32,      // offset: 4, size: 4
    c: f32,      // offset: 8, size: 4
    d: f32,      // offset: 12, size: 4
    e: f32,      // offset: 16, size: 4
    f: f32,      // offset: 20, size: 4
    g: f32,      // offset: 24, size: 4
    h: f32,      // offset: 28, size: 4
}
// Size: 32, Alignment: 4

// This FAILS in uniform buffer:
#[derive(ShaderType)]
struct Container {
    arr: [MyStruct; 2],
}

Container::assert_uniform_compat();  // PANIC!
```

**Why it fails:**
- Array element (MyStruct): size=32, alignment=4
- Array stride: round_up(32, 4) = 32
- Uniform min alignment: 16
- **32 is multiple of 16** ✓ (this would actually PASS)

Let me correct the example:

```rust
#[derive(ShaderType)]
struct SmallStruct {
    a: u32,      // offset: 0, size: 4
}
// Size: 4, Alignment: 4

// This FAILS:
[SmallStruct; 2]::assert_uniform_compat();  // PANIC!
```

**Why it fails:**
- Element size: 4, alignment: 4
- Stride: 4
- 4 % 16 ≠ 0 ✗

### 3.5 All ShaderTypes as Potential Array Elements

**Yes, encase treats all types as potential array elements** for uniform buffer validation.

From `src/types/array.rs:39`:
```rust
has_uniform_min_alignment: true,
```

This flag propagates up through the type hierarchy. Any struct containing an array (or type with this flag) must also set it.

### 3.6 Uniform vs Storage Context

**Uniform buffer context** (`src/core/buffers.rs:129`):
```rust
T::assert_uniform_compat();
```

**Storage buffer context** (`src/core/buffers.rs:50-57`):
```rust
pub fn write<T>(&mut self, value: &T) -> Result<()>
where
    T: ?Sized + ShaderType + WriteInto,
{
    // NO uniform compatibility check
    let mut writer = Writer::new(value, &mut self.inner, 0)?;
    value.write_into(&mut writer);
    Ok(())
}
```

Storage buffers don't call `assert_uniform_compat()`, so the stride check doesn't apply.

---

## 4. Buffer-Context-Dependent Validation

### 4.1 Checks Requiring Buffer Wrappers

**Uniform-specific checks:**
- Array stride ≥16 validation
- Struct field offset alignment to 16 bytes
- Runtime-sized array prohibition

**These checks require:**
```rust
UniformBuffer<B> or DynamicUniformBuffer<B>
```

**Code:** `src/core/buffers.rs:129, 316, 139, 326, 146, 333`

### 4.2 What Buffer Wrappers Supply

**UniformBuffer/StorageBuffer:**
- Type selection (uniform vs storage)
- Offset (always 0 for non-dynamic)

**DynamicUniformBuffer/DynamicStorageBuffer:**
- Type selection
- Dynamic offset management
- Alignment enforcement (min 32 bytes for WebGPU)

From `src/core/buffers.rs:171-179`:
```rust
pub const fn new_with_alignment(buffer: B, alignment: u64) -> Self {
    if alignment < 32 {
        panic!("Alignment must be at least 32!");
    }
    Self {
        inner: buffer,
        alignment: AlignmentValue::new(alignment),
        offset: 0,
    }
}
```

### 4.3 Information from ShaderType Alone

**ShaderType provides:**
- Size (min and runtime)
- Alignment
- Metadata (offsets, paddings, stride)

**ShaderType CANNOT provide:**
- Whether the type will be used in uniform or storage buffer
- Dynamic offset requirements
- Runtime array lengths (for reading)

### 4.4 Why Serializing Outside Context Changes Behavior

**Without buffer wrapper:**
```rust
let data: MyStruct = /* ... */;
// Can compute size, alignment, but can't validate uniform compatibility
```

**With buffer wrapper:**
```rust
let mut buffer = UniformBuffer::new(Vec::new());
buffer.write(&data).unwrap();  // Triggers assert_uniform_compat()
```

The wrapper provides the **usage context** that determines which validation rules apply.

**Key insight:** The validation is eager (at write time) rather than lazy because:
1. WGSL layout rules are context-dependent (uniform vs storage)
2. Catching errors early prevents GPU validation failures
3. Type safety ensures correct usage patterns

---

## 5. What encase Explicitly Supports vs Implicitly Discourages

### 5.1 Explicitly Supported Patterns

**From code structure and assertions:**

1. **Direct buffer wrapper usage:**
   ```rust
   let mut buffer = UniformBuffer::new(Vec::new());
   buffer.write(&my_data).unwrap();
   ```
   Evidence: `src/core/buffers.rs:50-150`

2. **Struct composition:**
   ```rust
   #[derive(ShaderType)]
   struct Inner { x: f32 }
   
   #[derive(ShaderType)]
   struct Outer { inner: Inner }
   ```
   Evidence: Derive macro supports nested structs (`derive/impl/src/lib.rs`)

3. **Array usage (with uniform constraints):**
   ```rust
   [Vec4f; 100]  // OK: stride = 16
   ```
   Evidence: `src/types/array.rs:23-61`

4. **Runtime-sized arrays in storage:**
   ```rust
   #[derive(ShaderType)]
   struct Data {
       #[shader(size(runtime))]
       items: Vec<f32>,
   }
   
   let mut buffer = StorageBuffer::new(Vec::new());
   buffer.write(&data).unwrap();
   ```
   Evidence: `src/types/runtime_sized_array.rs:145-146`

5. **Dynamic offsets:**
   ```rust
   let mut buffer = DynamicStorageBuffer::new(data);
   let offset1 = buffer.write(&item1).unwrap();
   let offset2 = buffer.write(&item2).unwrap();
   ```
   Evidence: `src/core/buffers.rs:218-230`

### 5.2 Technically Possible but Fragile Patterns

**From absence of direct support:**

1. **Manual offset calculation:**
   ```rust
   // Fragile: must manually track offsets
   let offset = my_struct.size().get();
   ```
   Evidence: No API for manual layout management

2. **Partial buffer writes:**
   ```rust
   // Fragile: must ensure alignment
   writer.write_slice(&bytes);
   ```
   Evidence: `WriteInto` trait but no structured partial write API

3. **Type reinterpretation:**
   ```rust
   // Fragile: no safe transmute
   let bytes = buffer.as_ref();
   ```
   Evidence: No safe type casting APIs

### 5.3 Why Wrapping Breaks Assumptions

**Breaking pattern:**
```rust
struct GpuUniform<T> {
    buffer: wgpu::Buffer,
    _marker: PhantomData<T>,
}

impl<T: ShaderType + WriteInto> GpuUniform<T> {
    fn update(&self, data: &T) {
        // Problem: No uniform compatibility check!
        let mut bytes = Vec::new();
        let mut buffer = StorageBuffer::new(&mut bytes);
        buffer.write(data).unwrap();
        self.buffer.write(bytes);
    }
}
```

**Why it breaks:**
1. Uses `StorageBuffer` (no uniform validation)
2. Data might be incompatible with uniform buffers
3. GPU validation fails at runtime (too late!)

**encase expects:**
```rust
impl<T: ShaderType + WriteInto> GpuUniform<T> {
    fn update(&self, data: &T) {
        // Correct: Use UniformBuffer wrapper
        let mut buffer = UniformBuffer::new(Vec::new());
        buffer.write(data).unwrap();  // Validates!
        self.buffer.write(buffer.into_inner());
    }
}
```

**Evidence:** The buffer wrapper types are the **primary API** (`src/core/buffers.rs`). They're not optional convenience wrappers—they're the validation layer.

---

## 6. Extractable Subset for Custom GpuUniform<T>

### 6.1 Minimal WGSL Rules for Single Uniform Struct

**Assumptions:**
- No arrays
- No dynamic offsets
- Single uniform binding
- Fixed-footprint types only

**Required rules:**

#### Rule 1: Field Alignment
- ✅ **Required**
- Each field must be aligned to its type's alignment
- Prevents misaligned memory access

**Enforcement:** `derive/impl/src/lib.rs:500`
```rust
offsets[i] = alignment.round_up(offset);
```

#### Rule 2: Struct Size Rounding
- ✅ **Required**
- Total size must be multiple of struct alignment
- Ensures proper padding at end

**Enforcement:** `derive/impl/src/lib.rs:679`
```rust
SizeValue::new(struct_alignment.round_up(offset))
```

#### Rule 3: Field Type Alignment
- ✅ **Required**
- Scalars: 4 bytes
- Vectors: Next power of 2 of size
- Nested structs: Max field alignment

**Enforcement:** Built into `ShaderType::METADATA`

#### Rule 4: Minimum Binding Size
- ✅ **Required**
- Buffer must be at least `min_size()` bytes
- Prevents GPU validation errors

**Enforcement:** `src/core/rw.rs:60-67` (Writer creation)

### 6.2 WGSL Rules That Can Be Dropped

**For the restricted model (no arrays, single uniform):**

#### Droppable 1: Array Stride = 16
- ❌ **Not needed** (no arrays)
- Would check: `src/types/array.rs:46-60`

#### Droppable 2: Runtime-Sized Array Prohibition
- ❌ **Not needed** (no arrays)
- Would check: `src/types/runtime_sized_array.rs:145-146`

#### Droppable 3: Dynamic Offset Alignment
- ❌ **Not needed** (no dynamic offsets)
- Would check: `src/core/buffers.rs:182-191`

#### Droppable 4: Inter-Field 16-Byte Spacing
- ❌ **Not needed** (without arrays as fields)
- Would check: `derive/impl/src/lib.rs:449-479`
- **CAVEAT:** If struct fields can contain arrays, this is required

### 6.3 Absolutely Required Checks

**To avoid UB or GPU validation errors:**

1. **Buffer size ≥ data size**
   ```rust
   if buffer.len() < T::min_size().get() as usize {
       panic!("Buffer too small");
   }
   ```
   From: `src/core/rw.rs:60-67`

2. **Type has valid metadata**
   ```rust
   const _: () = {
       let _ = T::METADATA;  // Compile-time check
   };
   ```
   Ensures all const assertions passed

3. **Alignment is power of 2**
   ```rust
   // Built into AlignmentValue::new()
   ```
   From: `src/core/alignment_value.rs:9-15`

4. **Size is non-zero**
   ```rust
   // Built into SizeValue::new()
   ```
   From: `src/core/size_value.rs:9-17`

### 6.4 Implementation Checklist

For implementing a custom `GpuUniform<T>` with restricted usage:

**Compile-time requirements:**
- [ ] `T: ShaderType + ShaderSize` (fixed-footprint)
- [ ] `T: WriteInto` (serializable)
- [ ] Validate `T::METADATA` computes without panic

**Runtime requirements:**
- [ ] Allocate buffer with size ≥ `T::min_size().get()`
- [ ] Align buffer offset to `T::METADATA.alignment().get()`
- [ ] Use `Writer::new()` for bounds checking
- [ ] Use `T::write_into()` for serialization

**Optional uniform-specific validation:**
- [ ] If supporting arrays: Check stride ≥ 16
- [ ] If supporting nested structs with arrays: Check field offsets

**Minimal safe implementation:**
```rust
pub struct GpuUniform<T> {
    buffer: wgpu::Buffer,
    _marker: PhantomData<T>,
}

impl<T: ShaderType + ShaderSize + WriteInto> GpuUniform<T> {
    pub fn new(device: &wgpu::Device) -> Self {
        let size = T::min_size().get();
        let buffer = device.create_buffer(&wgpu::BufferDescriptor {
            size,
            usage: wgpu::BufferUsages::UNIFORM | wgpu::BufferUsages::COPY_DST,
            mapped_at_creation: false,
            label: None,
        });
        Self {
            buffer,
            _marker: PhantomData,
        }
    }
    
    pub fn write(&self, queue: &wgpu::Queue, data: &T) {
        // Safety check (optional but recommended)
        const _: () = {
            let _ = T::METADATA;  // Validates type at compile time
        };
        
        // Serialize to temporary buffer
        let mut bytes = Vec::new();
        let mut writer = Writer::new(data, &mut bytes, 0)
            .expect("Failed to create writer");
        data.write_into(&mut writer);
        
        // Upload to GPU
        queue.write_buffer(&self.buffer, 0, &bytes);
    }
}
```

**Key insight:** For the restricted case, **most complexity comes from arrays and dynamic offsets**. A single-struct uniform can use encase's serialization without full validation.

---

## Summary

encase enforces WGSL layout correctness through:

1. **Compile-time const assertions** for alignment, stride, and offset rules
2. **Type-level metadata** capturing layout information
3. **Buffer wrapper types** providing usage context
4. **Trait-based validation** ensuring correctness before GPU upload

The design makes invalid layouts **impossible to compile**, pushing errors to compile time rather than runtime.

For custom implementations, the minimal required subset is:
- Type metadata validation
- Buffer size checking
- Proper serialization via `WriteInto`

The array stride validation is **aggressive by design**: encase assumes any type might be used in an array, so it validates eagerly to prevent subtle GPU errors.
