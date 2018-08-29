# Pseudorandom number generators

[PRNGs](https://gist.github.com/bryc/38a86416e8fa940e009adb15821f36d7) appear to have a lot in common with non-cryptographic hashes. They both try to achieve _random-looking output_. And in some cases employ similar concepts and borrow from each other. And some people have made both PRNGs and hash functions.

Like hashes, PRNGs often predominantly utilize 64-bit arithmetic, thus making it hard to find good JavaScript PRNGs. So this article documents my own implementations of PRNGs.

_Note that PRNGs typically need a separate algorithm to generate a random seed. This is best achieved by a hashing algorithm_.


| Algorithm | State size | Speed | Notes |
| --------- | ---------- | ----- | ----- |
| xoroshiro64+ | 64-bit | 8,026,741 | lfsr issues. |
| xoroshiro64* | 64-bit | 7,985,287 | lfsr issues. |
| xoroshiro64** | 64-bit | 7,982,769 | preferred version. lfsr issues, but fast. |
| xoshiro128+ | 128-bit | 6,927,766 | lfsr issues. |
| xoshiro128** | 128-bit | 6,914,233 | preferred version. has lfsr issues, but better than xorshift. |
| sfc32 | 128-bit | 7,178,353 | chaotic. best 2^128 state JS PRNG. passes practrand. |
| gjrand32 | 128-bit | 5,878,910 | chaotic.  |
| jsf32 | 128-bit | 4,838,098 | chaotic. sadly quite slow in JS. |
| jsf32b | 128-bit | 4,819,955 | jsf32 with another rotate. better randomness, no perf cost in JS. |
| xorshift128 | 128-bit | 6,068,160 |  |
| xorshift32 | 32-bit | 5,745,952 | no idea why it's slow. |
| xorshift32a | 32-bit | 5,702,701 |  |
| xorshift32b | 32-bit | 5,667,568 |  |
| lcg | 31-bit | 2,525,211 | park-miller lcg. super slow and only 31 bits. |
| mulberry32 | 32-bit | 7,988,488 | best 2^32 state JS PRNG. passes gjrand.  |

Out of the table, the best JS PRNGS seem to be: `sfc32`, and `mulberry32`. runnerups `jsf32b` and `gjrand32` should be good but seems to lag behind. `xoshiro128**` is good if you don't mind that it fails some statistical tests. 


## Alea

Alea is based on MWC (Multiply-with-Carry). It includes its own string hash function: Mash. It's speed is pretty good, but I cannot accurately measure its seed/state size (yet). 

```js
function Alea(seed) {
    if(seed === undefined) {seed = +new Date() + Math.random();}
    function Mash() {
        var n = 4022871197;
        return function(r) {
            for(var t, s, u = 0, e = 0.02519603282416938; u < r.length; u++)
            s = r.charCodeAt(u), f = (e * (n += s) - (n*e|0)),
            n = 4294967296 * ((t = f * (e*n|0)) - (t|0)) + (t|0);
            return (n|0) * 2.3283064365386963e-10;
        }
    }
    return function() {
        var m = Mash(), a = m(" "), b = m(" "), c = m(" "), x = 1, y;
        seed = seed.toString(), a -= m(seed), b -= m(seed), c -= m(seed);
        a < 0 && a++, b < 0 && b++, c < 0 && c++;
        return function() {
            var y = x * 2.3283064365386963e-10 + a * 2091639; a = b, b = c;
            return c = y - (x = y|0);
        };
    }();
}
```

## LCG (Lehmer MCG)

Has a state/period of 2^31-1. Is not that ideal, but it works. It's a bit slower than others due to the modulo operator. 

```js
function lcg(a) {
    return function() {
      a = Math.imul(48271, a) % 2147483647;
      return (a >>> 0) / 2147483647;
    }
}
```

## Xorshift

The original vanilla xorshift introduced in 2003. Comes in 128 and 32-bit flavors. All of the xo* PRNGs have some LFSR characteristics, which may have issues in linearity/binary rank tests. But they're fast and cute.

```js
function xorshift128(a, b, c, d) {
    return function() {
        var t = d ^ d << 11; t ^= t >>> 8;
        t ^= a; t ^= a >>> 19; 
        d = c | 0; c = b | 0; b = a | 0;
        a = t;
        return (t >>> 0) / 4294967296;
    }
}

function xorshift32(a) {
    return function() {
        a ^= a << 13;
        a ^= a >>> 17;
        a ^= a << 5;
        return (a >>> 0) / 4294967296;
    }
}

// potentially better variants of xorshift32:
function xorshift32a(a) {
    return function() {
        a ^= a << 25;
        a ^= a >>> 7;
        a ^= a << 2;
        return (Math.imul(a, 1597334677) >>> 0) / 4294967296;
    }
}

function xorshift32b(a) {
    return function() {
        var t = Math.imul(a, 1597334677);
        t = t>>>24 | t>>>8&65280 | t<<8&16711680 | t<<24;
        a ^= a << 25;
        a ^= a >>> 7;
        a ^= a << 2;
        return ((a + t) >>> 0) / 4294967296;
    }
}

```

## Xoroshiro

The xoroshiro family included two 32-bit compatible entries, `xoroshiro64**` and `xoroshiro64*`. It has a state size of 64-bit. I've included an unoffical implementation of `xoroshiro64+`. 

```js
function xoroshiro64ss(a, b) {
    return function() {
        var r = Math.imul(a, 0x9E3779BB); r = (r<<5 | r>>>27) * 5;
        b ^= a;
        a = (a << 26 | a >>> 6) ^ b ^ b << 9;
        b = b << 13 | b >>> 19;
        return (r >>> 0) / 4294967296;
    }
}

function xoroshiro64s(a, b) {
    return function() {
        var r = Math.imul(a, 0x9E3779BB);
        b ^= a;
        a = (a << 26 | a >>> 6) ^ b ^ b << 9;
        b = b << 13 | b >>> 19;
        return (r >>> 0) / 4294967296;
    }
}

// unofficial xoroshiro64+ for completeness
function xoroshiro64p(a, b) {
    return function() {
        var r = a + b;
        b ^= a;
        a = (a << 26 | a >>> 6) ^ b ^ b << 9;
        b = b << 13 | b >>> 19;
        return (r >>> 0) / 4294967296;
    }
}

```

## Xoshiro

The latest (as of May 2018) in the Xorshift-derivative series, xoshiro family now offers 128-bit state generators in 32-bit just like the original xorshift. Comes in two variants: `xoshiro128**` and `xoshiro128+`

```js
function xoshiro128ss(a, b, c, d) {
    return function() {
        var t = b << 9, r = a * 5; r = (r << 7 | r >>> 25) * 9;
        c ^= a; d ^= b;
        b ^= c; a ^= d; c ^= t;
        d = d << 11 | d >>> 21;
        return (r >>> 0) / 4294967296;
    }
}

function xoshiro128p(a, b, c, d) {
    return function() {
        var t = b << 9, r = a + d;
        c ^= a; d ^= b;
        b ^= c; a ^= d; c ^= t;
        d = d << 11 | d >>> 21;
        return (r >>> 0) / 4294967296;
    }
}
```

## JSF / smallprng

Bob Jenkin's PRNG from 2007. Unfortunately only has a 32-bit seed despite having a 128-bit state. But perhaps the entire state can be seeded using a different algorithm. Seems pretty decent. Bob Jenkins also made some nice hash functions. It is described as a "chaotic PRNG".

```js
function jsf32(a, b, c, d) {
    return function() {
        a >>>= 0; b >>>= 0; c >>>= 0; d >>>= 0;
        var t = a - (b << 27 | b >> 5) | 0;
        a = b ^ (c << 17 | c >> 15) | 0;
        b = c + d | 0;
        c = d + t | 0;
        d = a + t | 0;
        return (d >>> 0) / 4294967296;
    }
}

// 3-rotate version, improves randomness.
function jsf32b(a, b, c, d) {
    return function() {
        a >>>= 0; b >>>= 0; c >>>= 0; d >>>= 0;
        var t = a - (b << 23 | b >> 9) | 0;
        a = b ^ (c << 16 | c >> 16) | 0;
        b = c + (d << 11 | d >> 21) | 0;
        b = c + d | 0;
        c = d + t | 0;
        d = a + t | 0;
        return (d >>> 0) / 4294967296;
    }
}

// https://gist.github.com/imneme/85cff47d4bad8de6bdeb671f9c76c814
```

## gjrand32

Another "chaotic PRNG" designed by David Blackman. It is from the GJrand suite of random number tests. The original version is 64-bit and well-tested, this one hasn't been extensively tested but seems to be good.

```js
function gjrand32(a, b, c, d) {
    return function() {
      a >>>= 0; b >>>= 0; c >>>= 0; d >>>= 0;
      a = a << 16 | a >>> 16;
      b = b + c | 0;
      a = a + b | 0;
      c ^= b;
      c = c << 11 | c >>> 21;
      b ^= a;
      a = a + c | 0;
      b =  c << 19 | c >>> 13;
      c = c + a | 0;
      d = d + 0x96a5 | 0;
      b = b + d | 0;
      return (a >>> 0) / 4294967296;
    }
}
```

## sfc32

Yet another chaotic PRNG, the sfc stands for "Small Fast Counter".

```js
function sfc32(a, b, c, d) {
    return function() {
      a >>>= 0; b >>>= 0; c >>>= 0; d >>>= 0; 
      var t = (a + b) | 0;
      a = b ^ b >>> 9;
      b = c + (c << 3) | 0;
      c = (c << 21 | c >>> 11);
      d = d + 1 | 0;
      t = t + d | 0;
      c = c + t | 0;
      return (t >>> 0) / 4294967296;
    }
}
```

## Mulberry32 and SplitMix32

These PRNGs use a 32-bit state, similar to xorshift32. Quite useful for embedded systems. And potentially better than the Lehmer MCG. But due to the small state size, they aren't as good as the 128-bit state PRNGs.


```c
function mulberry32(a) {
    return function() {
      var t = a += 0x6D2B79F5;
      t = Math.imul(t ^ t >>> 15, t | 1);
      t ^= t + Math.imul(t ^ t >>> 7, t | 61);
      return ((t ^ t >>> 14) >>> 0) / 4294967296;
    }
}

function splitmix32(a) {
    return function() {
      var t = a += 0x9e3779b9;
      t ^= t >>> 15; t = Math.imul(t, 0x85ebca6b);
      t ^= t >>> 13; t = Math.imul(t, 0xc2b2ae3d);
      return ((t ^= t >>> 16) >>> 0) / 4294967296;
    }
}
```


****

# Seed generating functions

Here are various functions that can be used to efficiently generate seeds for PRNGs. I'll include integer and string hashing functions as well here.

Note: Certain generators have their own seed procedure, such as jsf32, sfc32 and potentially gjrand32. I'm assuming that you can seed these generators with any hashing algorithm such as MurmurHash3, which is what I use.

`initseed` is an experimental 128-bit seed generator. It takes an array of 4 32-bit numbers and mixes them using the mixing function from Murmur3/xxHash.

```js
function initseed(s) {
    function mix(h) {
        h ^= h>>>15; h = Math.imul(h,2246822507);
        h ^= h>>>13; h = Math.imul(h,3266489917); h ^= h>>>16;
        return h;
    }
    var e = s[0] + s[1] + s[2] + s[3];
    s[0] += e ^ 597399067; s[1] ^= e ^ 668265263; s[2] += e ^ 951274213; s[3] ^= e ^ 374761393;
    s[0] = mix(s[0]) + s[1]; s[1] = mix(s[1]);
    s[2] = mix(s[2]) + s[3]; s[3] = mix(s[3]);
    e = s[0] + s[1] + s[2] + s[3];
    s[0] += e; s[1] ^= e; s[2] += e; s[3] ^= e;
    return s;
}