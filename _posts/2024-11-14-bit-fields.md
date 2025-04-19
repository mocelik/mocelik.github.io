---
categories: [cpp]
date: '2024-11-14T22:23:42-05:00'
layout: post
permalink: /c/bit-fields/
tags: [c, cpp]
title: Bit-fields
---

Bit-fields are a feature of the C and C++ languages to optimize the space taken by an integral member variable of a struct or class. It specifies the exact number of bits that a data member will hold, so that multiple adjacent bit-fields can be packed (condensed) into one field.

Using bit-fields can reduce the amount of memory used, which can improve performance. On the other hand, bit-fields can increase the CPU overhead to interact with the data, which can impede performance. In the worst case, a bit-field may be set up in such a way that it doesn’t reduce the memory usage, but it still adds CPU overhead. This article goes into detail on how that can happen and how to avoid it.

Most of the details of bit-fields are implementation-defined, meaning that their behaviour is only partially specified by the C++ (and C) standards. The case studies below attempt to explicitly state whether a behaviour is standardized or implementation defined (platform-dependent), but this page has not been peer-reviewed and is susceptible to errors. The examples given here were validated using GCC 14.2 on both x86 and ARM64 platforms. Differences are expected when using GCC for different hardware platforms, or when using another compiler such as MSVC.

The next section will discuss some of the portability concerns. Following that, there is a brief introduction into the basics of bit-fields. Finally, a few case studies show good and bad uses of bit-fields.

## Portability Concerns

The [C++ standard](https://eel.is/c++draft/class.bit) mentions the following:

> \[…\] Allocation of bit-fields within a class object is implementation-defined. Alignment of bit-fields is implementation-defined. Bit-fields are packed into some addressable allocation unit.
> 
> \[*Note [1](https://eel.is/c++draft/class.bit#note-1)*: Bit-fields straddle allocation units on some machines and not on others. Bit-fields are assigned right-to-left on some machines, left-to-right on others. — *end note*\]

This behaviour is specified by the Application Binary Interface (ABI). On most (non-Microsoft) systems, this will be the industry-standard [Itanium ABI](https://itanium-cxx-abi.github.io/cxx-abi/abi.html), as [mentioned by GCC](https://gcc.gnu.org/onlinedocs/libstdc++/manual/abi.html). However, the Itanium ABI does not fully define how bit-fields will be laid out either (emphasis mine):

> If `sizeof(T)*8 >= n`, **the bit-field is allocated as required by the base C ABI** \[…\].

The *base C ABI* is usually the System V (SysV) ABI, again [mentioned by GCC](https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html#index-mabi-7). The SysV ABI is split into two parts: a generic ABI (**gABI**) and a processor supplement ABI (**psABI**).

As the name suggestions, the [SysV gABI](https://refspecs.linuxfoundation.org/elf/gabi41.pdf) contains generic information that is platform-independent, such as the ELF format. It does not contain hardware-specific information: that is left to the psABI corresponding to that processor (e.g. the psABI for [x86-64](https://gitlab.com/x86-psABIs/x86-64-ABI), for [ARM](https://github.com/ARM-software/abi-aa/tree/main), and so on). The gABI explicitly states which parts are left for the psABI to define.

Regarding bit-fields, the implementation details are specified in the psABI of the specific platform. For x86-64, [a prebuilt PDF can be found here](https://raw.githubusercontent.com/wiki/hjl-tools/x86-psABI/x86-64-psABI-1.0.pdf). For ARM, the specific section of [the psABI that discusses bit-fields is here](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst#Bit-fields).

## Overview: Size and Runtime Access Cost

The BitfieldedStruct below is an example of a struct with three bit-fields, each with a different width.

```cpp
struct BitfieldedStruct {
    uint32_t value1 : 20;
    uint32_t value2 : 10;
    uint32_t value3 : 2;
};
```

The bit-field widths (20, 10, 2) add up to 32 bits, or 4 bytes, so those three members take up only 4 bytes of space instead of the usual 12.

With that said, although using the bit-field saves some memory, reading from or writing to the fields takes some additional work at runtime. An example of what a compiler might generate for x86-64 and ARM64 is shown below. Some common load and store instructions were removed for brevity. The source code can be found here: [godbolt](https://godbolt.org/z/Kv41z6cYK).

```asm
// x86-64

// read value1
and  eax, 1048575
// write value1
and  eax, 1048575
and  esi, -1048576
or   esi, eax

// read value2
shr  eax, 20
and  eax, 1023
// write value2
and  ax, 1023
sal  eax, 4
and  si, -16369
or   esi, eax

// read value3
shr  eax, 30
// write value3
sal  eax, 6
and  esi, 63
or   esi, eax
```

For x86-64:

The `and` instructions clears the excess bits (since ANDing anything with 0 is 0)

1048575 corresponds to 20 bits set and the remaining bits cleared

1023 corresponds to 10 bits set and the remaining bits cleared

The `or` instructions sets some bits without affecting existing bits outside of the set range

The `shr` instruction **sh**ifts the data (to the **r**ight), by 20 for value2, and by 30 for value3 to shift them back to 0 from their respective offsets

The `sal` instruction is a **s**hift **a**rithmetic **l**eft, so the left counterpart of `shr`.

```asm
// ARM64

// read value1
and  w0, w0, 1048575
// write value1
bfi  w2, w1, 0, 20

// read value2
ubfx x0, x0, 20, 10
// write value2
bfi  w2, w1, 20, 10

// read value3
lsr  w0, w0, 30
// write value3
bfi  w2, w1, 30, 2
```

For ARM64:

The `and` instruction is similar to the x86-64 version.

The `bfi` is a **b**it-**f**ield **i**nsert instruction to write to a bit-field; used in all of the writes.

The `ubfx` is an **u**nsigned **b**it-**f**ield e**x**tract instruction, for 10 bits at offset 20

The `lsr` is a **l**ogical **s**hift **r**ight to shift the data by 30 bits, similar to the `shr` instruction from x86-64

The additional instructions seen above will add an extra runtime workload to the CPU; but using bit-fields is not necessarily slower. Since bit-fields reduce the memory footprint of data, less overall RAM is required, and more data can fit into a cache. This can improve performance for large datasets. The next section goes into detail on how bit-fields may (or may not) be laid out to save space.

## Bit-field Case Studies

In the introductory example, the size of a struct was reduced from 12 to 4 bytes using bit-fields. However, using a bit-field may not always reduce the size of a struct. The following case studies show various bit-field configurations and explain how or why they behave the way they do.

### Case 1: Straddling – Spanning multiple units

Consider the following struct:

```cpp
struct NonPortableBitfield {
   uint8_t first  : 7;
   uint8_t second : 7;
   uint8_t third  : 2;
};
```

The sum of the three widths is 16 bits or 2 bytes. However, neither the C nor the C++ standards specify whether a bit-field member that doesn’t fit into the previous unit can straddle (span) two units or not. The above struct usually takes up 3 bytes of space, even though there are only 2 bytes worth of data. That’s not all; even though there were no memory savings, there is still a runtime overhead because the CPU still strips out the padding bits. This is a bad use of bit-fields; it would have been more efficient to declare this same struct without any bit-fields at all.

### Case 2: Avoiding Straddling

In the case above with 16-bits spread over 3 bytes, the standard defers to the implementation to define whether the member fields will straddle two units or not. However, there is a simple portable way to reduce the total size: use a larger type so that everything fits in the same unit. Specifically, each `uint8_t` in the above example can be replaced with a `uint16_t` as follows:

```cpp
struct PortableBitfield {
    uint16_t first  : 7;
    uint16_t second : 7;
    uint16_t third  : 2;
};
```

Now, all three members fit into a 16-bit integer, so there is no need to straddle the members. The following guarantee from the C-standard takes effect:

> If enough space remains, a bit-field that immediately follows another bit-field in a structure shall be packed into adjacent bits of the same unit.

And thus, space is guaranteed to be saved, and one can expect the size of the struct to be consistent from one platform to the next. However, this guaranteed space-saving behaviour has a cost: there is a difference in the alignment requirements of a uint8_t and a uint16_t. While there is no portable way to remove or change alignment requirements, there are some common ways that compilers may provide this functionality. This is discussed in the next section.

Note that on some platforms it may be sufficient to only increase the size of the first member; the following members may be allocated within the first.

### Case 3: Packed Bit-fields

Having studied the case where using a uint16_t can prevent straddling, there may be a temptation to always use bigger types. However, there is a stricter alignment requirement whenever a bigger type is used: although an array of two uint8_t’s will take up the same amount of space as a single uint16_t, the uint8_t array may allocated at any address, whereas the uint16_t must be at an address divisible by 2. This may be a concern if the struct is used as a data member of another struct or class due to padding.

The alignment and padding of a struct can sometimes be controlled through attributes provided by the compiler. Specifically, the `__attribute__((packed))` or `#pragma pack` attributes can be used as follows:

```cpp
struct __attribute__((packed)) PackedBitfield {
    uint16_t first  : 7;
    uint16_t second : 7;
    uint16_t third  : 2;
};
```

```cpp
#pragma pack(push, 1)
struct PackedBitfield {
    uint16_t first  : 7;
    uint16_t second : 7;
    uint16_t third  : 2;
};
#pragma pack(pop)
```

In fact, using this attribute may even allow straddling on some platforms, so the type might be changed back to a `uint8_t`, too.

However, the `packed` attribute doesn’t come for free. Just like how bit-fields themselves come at a runtime cost, packing and removing alignment also comes at an extra cost. Below is a comparison of the assembly to set the middle (`second`) bit-field member on x86 on a standard vs packed structure using GCC.

```asm
// x86 write to PortableBitfield.second
mov     eax, esi
movzx   esi, WORD PTR [rdi]
and     eax, 127
sal     eax, 7
and     si, -16257
or      esi, eax
mov     WORD PTR [rdi], si
```

```asm
// x86 write to PackedBitfield.second
movzx   eax, BYTE PTR [rdi]
mov     edx, esi
shr     si
sal     edx, 7
and     esi, 63
and     eax, 127
or      eax, edx
mov     BYTE PTR [rdi], al     
movzx   eax, BYTE PTR [rdi+1]
and     eax, -64
or      eax, esi
mov     BYTE PTR [rdi+1], al
```

There are more instructions to write to the packed `second` than to the standard (unpacked) version. Although it is nice to visualize the overhead of packing by seeing the extra instructions like this, there are cases where the packed version does not generate any additional assembly. For example, the same compiler that generated the above code for the `second` field did not generate any different code for the `first` and `third` fields between the standard and packed versions.

However, even in cases where identical assembly is generated, the standard and packed versions may not have the same performance: there are hardware operations from the cache and memory controllers that may introduce extra latency. For example, if a uint16_t that is unaligned (due to packing) happens to straddle a cache line boundary, then both of those lines need to have been cached. The odds of that happening for 2-bytes on a 64-byte cache line is 1/64 – though this may be further halved on some platforms due to the [spatial prefetcher fetching cache lines in pairs](https://stackoverflow.com/questions/29199779/false-sharing-and-128-byte-alignment-padding).

The NonPortable, Portable, and Packed Bitfield case studies can be seen on [godbolt](https://godbolt.org/z/T6W1jvzK1).

### Case 4: Overlapped Bit-fields and Non-bit-fields

On a different vein, consider the following struct:

```cpp
struct MixedBitfield_1 {
    uint16_t first : 8;
    uint8_t  second;
};
```

Although it is nonsensical to have a struct laid out like this, it serves an educational purpose before the more complicated cases afterwards. The uint8_t member, even if it is not a bit-field, may be allocated within the 8 bits of padding from the first bit-field member. This is implementation-defined behaviour, and is referred to as “overlapping” in some psABIs.

What would the size of the following struct be?

```cpp
struct MixedBitfield_2 {
    uint32_t first : 20;
    uint32_t second;
    uint32_t third : 12;
};
```

We know that the `first` and `third` members have 32 bits in total, with another 32 bits from `second`, for a total of 64 bits or 8 bytes. However, the size of the above struct, as-is, is 12 bytes.

Since the second field is not a bit-field, it has to be aligned and accessed like a standard `uint32_t`. It cannot be implicitly made into a bit-field to complement the `first` and `third` fields adjacent to it – even if straddling is allowed.

The reason MixedBitfield_1 was able to allocate `uint8_t second` into the bit-field padding of the previous member is because there were no alignment requirements preventing it. However, for MixedBitfield_2, the `uint32_t second` member has to be allocated on a 4-byte boundary, so there must be some padding to allow that to happen.

To reduce padding as much as possible, the fields should be re-ordered so that the first and third members share the same `uint32_t`. No, the compiler cannot change the order for you.

```cpp
struct ReorderedBitfield {
    uint32_t first : 20;
    uint32_t third : 12;
    uint32_t second;
};
static_assert(sizeof(ReorderedBitfield) == 2*sizeof(uint32_t));
```

Interleaved normal and bit-field members may be useful in scenarios where the limited space leftover by the bit-field may allow normal members to fit, such as the following:

```cpp
struct AmazingStruct {
    uint64_t first : 40;
    uint8_t  second;
    uint16_t third;
};
```

Impressively, `second` and `third` can be allocated within the unused parts of the `first` member, so the size of this struct is only 8 bytes. Importantly, the order of `second` and `third` is important due to alignment. The `uint16_t` could not have been placed after the 40th bit, since that address is not on a 2-byte boundary. Attempting to reorder the last two members would double the size of the struct due to tail-padding optimization, and would make this another bad use of bit-fields.

These structs can be seen on [godbolt](https://godbolt.org/z/MYxcb65T8).

### Case 5: Packed Interleaved Bit-fields and Standard Fields

Consider MixedBitfield_2 from the previous case, except with the compiler-specific `packed` attribute. In that case, the alignment requirement is lifted, and the only padding will be to round the bit-field widths to a multiple of 8:

```cpp
struct __attribute__((packed)) PackedMixedBitfield {
    uint32_t first : 20;
    uint32_t second;
    uint32_t third : 12;
};
static_assert(sizeof(PackedMixedBitfield) == 9);
```

20 will be rounded up to 24, and 12 will be rounded up to 16. This will result in `first` using 3 bytes, `third` using 2 bytes, and the expected `second` using 4 bytes, for a total of 9 bytes of space. While this is smaller than the original 12-byte version, it is still larger than the ReorderedBitfield, and also incurs more runtime overhead due to unaligned access.

Again, on [godbolt](https://godbolt.org/z/7Erv9bae4).

### Case 6: Inter-Bit-field Struct Interactions

If two bit-fielded structs are allocated next to each other, such as in a struct or array, can they share bits?

```cpp
struct SingleBit {
    uint8_t value : 1;
};

struct HalfByte {
    uint8_t value : 4;
};

struct BitfieldArray {
    SingleBit elems[8];
};
static_assert(sizeof(BitfieldArray) == 8);

struct CompositeBitfield {
    SingleBit first;
    SingleBit second;
    SingleBit third;
    SingleBit fourth;
    HalfByte fifth;
};
static_assert(sizeof(CompositeBitfield) == 5);
```

In short, no. An easy way to build intuition on why this can’t happen is to consider whether `SingleBit` or `HalfByte` can be stored by reference (or pointer) – do they have unique addresses?

Yes: `SingleBit` and `HalfByte` must be allowed to be passed as individual arguments to a function call, so they must have unique addresses. Otherwise, if they had been further packed together, then a compiler would not be able to compile a function that took a `SingleBit` by reference. If it could, then the reference would have to encode more than just the byte-address; it would need the bit offset, too.

On [godbolt](https://godbolt.org/z/71hz8bnnv).

## A Time and Place for Everything

We have demonstrated that there are no universal laws on the “best” ways to lay out bit-fields. The layout and size of a bit-field depends entirely on the ordering and widths of the various bit-field members, as well as the ABI, so each bit-field candidate is a unique case. Furthermore, each environment is unique: some environments may have their performance limited by memory, in which case bit-fields may help, but others may have their performance limited by CPU usage, which case bit-fields may limit the performance even more.

While the only way to determine if a bit-field will improve performance is to run a benchmark in the target environment, the cases above demonstrate bit-field layouts that do not reduce memory while still increasing CPU overhead – a definite performance inhibitor. Those cases should be avoided at all costs.

Using a bit-field is an opportunity for optimization that might end up being a pessimization. Tread carefully.
