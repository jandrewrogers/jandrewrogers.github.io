---
layout: post
title: "MetroHash: Faster, Better Hash Functions"
---

MetroHash is a set of state-of-the-art hash functions for non-cryptographic use cases. **They are notable for being algorithmically generated in addition to their exceptional performance.** [Get the source here](https://github.com/jandrewrogers/MetroHash). The set of published hash functions may expand in the future.

* Fastest general-purpose algorithms for bulk hashing
* Fastest general-purpose algorithms for small, variable length keys
* Excellent statistical quality, similar to cryptographic hashes
* Supports both incremental and one-shot hashing
* Currently 64-bit, 128-bit, and 128-bit CRC instruction variants
* Unbounded set of statistically unique hash functions can be generated
* Elegant, compact, readable functions

In 2012, I started generating high-quality hash function families using a novel algorithm analysis and generation technique I developed. Thousands of excellent hash functions have been constructed this way. Function families are tunable for a mix of speed, statistical quality, and instruction set. Resistance to cryptanalysis was not a design criterion.

The set of functions included here were generated in an afternoon from related families. All easily pass Austin Appleby's [excellent SMHasher test suite](https://code.google.com/p/smhasher/). While these hash functions are optimized for modern Intel x86 with no attempt at portability, the underlying technique can be generalized to any microarchitecture. 

### Performance

MetroHash functions are much faster than comparable algorithms (metro128crc is memory bandwidth bound for large keys) while offering a statistical profile similar to MD5, a cryptographic hash. Functions generated in the same family, or using a different seed, have identical performance characteristics but are effectively statistically independent. 

Google's [CityHash](https://code.google.com/p/cityhash/) functions are used for speed comparison below since they are among the fastest high-quality functions and are available in the same variants. Relative performance varies a small amount across microarchitectures. The following table is typical for my Haswell test environment.


|              |Bulk Hashing                 |Small Keys                   |
|:-------------|:----------------------------|:----------------------------|
|**Metro64**     | 50% faster than City64      | 15% faster than City64      |
|**Metro128**    | 25% faster than City128     | 60% faster than City128     |
|**Metro128crc** | 30% faster than City128crc  | 80% faster than City128crc  |


The MetroHash algorithms included are designed to be general-purpose hash functions, much like CityHash. It is also possible to generate hash functions that are more highly optimized for a narrower set of use cases. 

### Background

A few years ago I had a number of use cases for hash functions that were not being adequately served by popular algorithms. The hash functions were generally suboptimal, giving up too much speed or too much statistical quality or both. I needed an expanded ecosystem of high-quality, high-performance hash functions that went beyond what something like Google's [CityHash](https://code.google.com/p/cityhash/) could deliver. MetroHash was the working name for my project to create it.

Designing fast, robust hash functions by hand is painful. My solution was to design algorithms and software that could iteratively generate, analyze, and optimize hash functions across an intractably large universe of potential hash function designs. The nature of the process is beyond the scope of this post, but conceptually it uses gradient descent to search for certain types of smoothable high-order, high-dimensionality functions, derivatives of which can be used to analytically construct a hash function with a relatively high probability of exhibiting high statistical quality. Over the nearly 100,000 compute hours spent on this analysis, the quality of the hash functions producible from the analysis data progressively improved. It was cool research and produced great results.

There is known room for improvement with additional analysis and likely with expanded use of instruction set extensions. However, this is not an active project for me. I had to dust off the code in order to generate the functions included here.

### Some Notes And Observations

The optimization process tends to produce tidy hash functions that conserve a small set of code motifs. This is presumably because they have properties that are nearly optimal for the target microarchitecture or improve predictability of function behavior from the software's perspective. Hash functions generated this way likely get close to the limits of hash performance for a given combination of microarchitecture, instruction set primitives, and statistical robustness. 

These hash functions were thoroughly tested for statistical weaknesses using standard tools and methods but are not cryptographic designs. Some key patterns may create an anomalous statistical bias in the hash functions. Fortunately, if such a pattern is found then it is straightforward to re-run the analysis and generation process to account for it.

AES-NI cryptographic instruction primitives available on modern Intel processors are typically too slow for the amount of usable randomness they generate per clock cycle. The instructions have a higher latency than CRC instructions and the output retains significant bias. That said, it would be interesting to force the analysis and optimization process to use that instruction set, given copious computing power, to see what it could come up with. 

