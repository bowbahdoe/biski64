# LoopMix128: Fast and Robust 2^128 Period PRNG

This repository contains `LoopMix128`, an extremely fast pseudo-random number generator (PRNG) with a guaranteed period of 2^128, proven injectivity, and clean passes in both BigCrush and PractRand (32TB). It is designed for non-cryptographic applications where speed and statistical quality are important.

## Features

* **High Performance:** Significantly faster than standard library generators and competitive with or faster than other modern high-speed PRNGs like wyrand and xoroshiro128++.
* **Good Statistical Quality:** Has passed TestU01's BigCrush suite and PractRand (up to 32TB) with zero anomalies.
* ~**Guaranteed Period:** Minimum period length of 2^128 through its 128 bit low/high counter looping.~
* **Proven Injectivity:** Z3 Prover proven injectivity across its 192 bit state. ([z3 script](check_injective.z3)) ([results](check_injective_out.txt))
* **Parallel Streams:** The injective 192 bit state facilitates parallel streams as outlined below.

*Note: I found a counterexample using Z3 Solver in which fast_loop maps to itself (0x5050a1e1d03b6432). A new version will be released with fast_loop and slow_loop as simple Weyl Sequences.*

## Performance

* **Speed:** 8.75x Java random, 21% faster than Java xoroshiro128++, 98% faster than C xoroshiro128++ and PCG64 ([benchmark](benchmark_out.txt))
* Passed 256M to 32TB PractRand with zero anomalies ([results](test_practrand_out.txt))
* Passed BigCrush with these lowest p-values: ([results](test_bigcrush_out.txt))
    * `0.01` sknuth_MaxOft (N=20, n=10000000, r=0, d=100000, t=32), Sum ChiSqr
    * `0.02` sknuth_CouponCollector (N=1, n=200000000, r=27, d=8), ChiSqr
    * `0.02` svaria_WeightDistrib (N=1, n=20000000, r=28, k=256, Alpha=0, Beta=0.25), ChiSqr

## Algorithm Details

```
// Golden ratio fractional part * 2^64
const uint64_t GR = 0x9e3779b97f4a7c15ULL;

// Initialized to non-zero with SplitMix64 (or equivalent)
uint64_t slow_loop, fast_loop, mix; 

// Helper for rotation
static inline uint64_t rotateLeft(const uint64_t x, int k) {
    return (x << k) | (x >> (64 - k));
}

// --- LoopMix128 ---
uint64_t loopMix128() {
  uint64_t output = GR * (mix + fast_loop);

  // slow_loop acts as a looping high counter (updating once per 2^64 calls) to ensure a 2^128 period
  if ( fast_loop == 0 ) {
    slow_loop += GR;
    mix ^= slow_loop;
    }

  // A persistent non-linear mix that does not affect the period of fast_loop and slow_loop
  mix = rotateLeft(mix, 59) + fast_loop;

  // fast_loop loops over a period of 2^64
  fast_loop = rotateLeft(fast_loop, 47) + GR;

  return output;
  }
```

*(Note: The code above shows the core logic. See implementation files for full seeding and usage examples.)*


## Parallel Streams

Thanks to the proven injectivity of the 192 bit state of LoopMix128, parallel streams can be implemented as follows:
* Simply randomly seed fast_loop, slow_loop, and mix and take advantage of the large 2^192 injective state which makes collisions incredibly improbable.
* Or, if absolutely critical - randomly seed fast_loop and mix for each stream and manually assign slow_loop uniquely to each stream taking into account that slow_loop normally increases by GR at the end of every 2^64 cycle (spacing slow_loop evenly across the streams).


## PractRand Testing (With Varied Seeds)

As running BigCrush and PractRand can behave differently depending on initial seeded states, PractRand was also run multiple times from 256M to 8GB using varied initial seeds (seeding with SplitMix64). Below are the counts of total suspicious results when running PractRand 1000 times for LoopMix128 and some reference PRNGs:

```
LoopMix128          0 failures, 24 suspicious
xoroshiro256++      0 failures, 27 suspicious
xoroshiro128++      0 failures, 28 suspicious
wyrand              0 failures, 32 suspicious
/dev/urandom        0 failures, 37 suspicious
```

## Creation
Created by Daniel Cota after he fell down the PRNG rabbit-hole by circumstance.
