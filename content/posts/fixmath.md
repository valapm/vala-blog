---
title: "Advanced Arithmetic in Bitcoin Script"
date: 2022-10-16T11:33:40+03:00
draft: false
tags: ["sCrypt"]
cover:
  image: "/spiral.jpeg"
  alt: "logarithmic spiral"
  relative: false
  hidden: false
---

In Bitcoin, we are limited to basic arithmetic. Everything available to us in Script is addition, subtraction, multiplication, division, and modulus.

If we want to challenge Ethereum and build Defi applications using automated market makers, compound interest or liquidity mechanisms, this is not enough. Fortunately, there is a way to use what we have available and implement everything we need. We built a [library](https://github.com/valapm/bsv-fixmath) for Vala implementing `log`, `exp`, `root` and `pow`.

One challenge we face is Bitcoin's lack of floating points. We'll have to implement everything using [fixed-point arithmetic](https://en.wikipedia.org/wiki/Fixed-point_arithmetic). The next challenge is keeping our script code concise. Even on BSV we don't want giant transactions for simple mathematics. We will have to make a trade-off between precision and size.

All examples will be using [sCrypt](https://scrypt.io/). We will work with 64-bit fixed-point numbers:

```js
static int precision = 64;
static int scale = 18446744073709551616; // 2 ^ precision

int fixedPointInt = num << precision;
```

The key is to approximate the logarithm and exponentiation. From there, we can derive everything else we need. Let's start with the binary logarithm. This is an approximation for numbers greater than 1.

```js
/**
* Finds the zero-based index of the first one in the binary representation of x.
* Accepts numbers scaled by 2**64 (64-bit fixed-point number).
* Adapted from https://github.com/paulrberg/prb-math
*/
static function mostSignificantBit(int x) : int {
  int msb = 0;

  if (x >= 340282366920938463463374607431768211456) {
    // 2^128
    x = x >> 128;
    msb += 128;
  }
  if (x >= 18446744073709551616) {
    // 2^64
    x = x >> 64;
    msb += 64;
  }
  //
  // Several similar lines removed for this example....
  //
  if (x >= 2) {
    // 2^1
    // No need to shift x any more.
    msb += 1;
  }

  return msb;
}

/**
* Calculates the binary logarithm of x.
* Accepts and returns scaled by 2**64 (64-bit fixed-point number).
* Only works for x greater than 1 << 64 (log2(1)).
* Adapted from https://github.com/paulrberg/prb-math
*/
static function log2(int x) : int {
  if (x < scale) {
    require(false);
  }

  // Calculate the integer part of the logarithm and add it to the result and finally calculate y = x * 2^(-n).
  int n = mostSignificantBit(x / scale);

  // The integer part of the logarithm as a signed 59.18-decimal fixed-point number. The operation can't overflow
  // because n is maximum 255, scale is 1e18 and sign is either 1 or -1.
  int result = n * scale;

  // This is y = x * 2^(-n).
  int y = x >> n;

  // If y = 1, the fractional part is zero.
  if (y != scale) {

    // Calculate the fractional part via the iterative approximation.
    // The "delta >>= 1" part is equivalent to "delta /= 2", but shifting bits is faster.
    loop (64) : i {
      y = (y * y) / scale;

      // Is y^2 > 2 and so in the range [2,4)?
      if (y >= scale << 1) {
        // Add the 2^(-m) factor to the logarithm.
        int delta = scale >> (i + 1);
        result += delta;

        // Corresponds to z/2 on Wikipedia.
        y = y >> 1;
      }
    }
  }
  return result;
}
```

From here, we can easily calculate the logarithm of other bases.

```js
static int ln2 = 12786308645202657280; // log(x) * scale / log2(x)
static int ln10 = 5553023288523357184; // log10(x) * scale / log2(x)

static function log(int x) : int {
	return FixMath.log2(x) * FixMath.ln2 / FixMath.scale;
}

static function log10(int x) : int {
	return FixMath.log2(x) * FixMath.ln10 / FixMath.scale;
}
```

Next up is the exponentiation of base 2. This approximation works between -60 and 192.

```js
/**
* Calculates the binary exponent of x using the binary fraction method.
* Accepts and returns scaled by 2**64 (64-bit fixed-point number).
* Adapted from https://github.com/paulrberg/prb-math
*/
static function exp2(int x) : int {
  if (x > 3541774862152233910272 || x < -1103017633157748883456) {
    // 192 max value
    // -59.794705707972522261 min value
    require(false);
  }

  // Start from 0.5 in the 192.64-bit fixed-point format.
  int result = 0x800000000000000000000000000000000000000000000000;

  // Multiply the result by root(2, 2^-i) when the bit at position i is 1. None of the intermediary results overflows
  // because the initial result is 2^191 and all magic factors are less than 2^65.
  if ((x & 0x8000000000000000) > 0) {
    result = (result * 0x16a09e667f3bcc909) >> 64;
  }
  if ((x & 0x4000000000000000) > 0) {
    result = (result * 0x1306fe0a31b7152df) >> 64;
  }
  if ((x & 0x2000000000000000) > 0) {
    result = (result * 0x1172b83c7d517adce) >> 64;
  }
  //
  // Several similar lines removed for this example....
  //
  if ((x & 0x1) > 0) {
    result = (result * 0x10000000000000001) >> 64;
  }

  // We're doing two things at the same time:
  //
  //   1. Multiply the result by 2^n + 1, where "2^n" is the integer part and the one is added to account for
  //      the fact that we initially set the result to 0.5. This is accomplished by subtracting from 191
  //      rather than 192.
  //   2. Convert the result to the unsigned 60.18-decimal fixed-point format.
  //
  // This works because 2^(191-ip) = 2^ip / 2^191, where "ip" is the integer part "2^n".
  result = result << 64;
  result = result >> (191 - (x >> 64));
  return result;
}
```

Now, we can easily derive the exponential function.

```js
static int log2e = 26613026195688644608; // Math.floor(Math.log2(Math.E) * 2**64)

/**
* Accepts and returns scaled by 2**64 (64-bit fixed-point number).
*/
static function exp(int x) : int {
  if (x > 2454971259878909673472 || x < -764553562531197616128) {
    // Max value is 133.084258667509499441
    // Min value is -41.446531673892822322
    require(false);
  }

  return FixMath.exp2((x * log2e) >> 64);
}
```

Using exp and log, we can derive `root` and `pow`.

```js
static function sqrt(int x) : int {
  return exp2(log2(x) / 2);
}

static function root(int x, int base) : int {
  return exp2((log2(x) << 64) / base);
}

static function pow(int base, int exp) : int {
  return exp2((exp * log2(base)) >> 64);
}
```

## How big are the Scripts?

The exponential function is smallest with only **8kb**. The logarithm comes in with **17kb** while root and pow require **25kb** each as they are using `exp` and `log`. With 10 sats per kilobyte as currently recommended by Gorillapool (they accept 1 sat / kilobyte), running the root or pow functions would set you back about 1/100 of a cent. For comparison, calculating the natural logarithm on Ethereum requires [at least 585 gas](https://xn--2-umb.com/22/exp-ln/index.html), coming up to 1 cent at current gas costs. 100x cheaper then Ethereum!

## Vala

Vala uses `exp` and `log` in its automated market maker, namely the [Logarithmic Market Scoring Rule](https://www.cultivatelabs.com/crowdsourced-forecasting-guide/how-does-logarithmic-market-scoring-rule-lmsr-work). Without this, there could be no liquidity mechanism and the app would be unusable.

## The library

The full library is open sourced on Github and contains a Typescript implementation of the same functions in addition to the sCrypt code so that you can actually generate valid transactions.

https://github.com/valapm/bsv-fixmath
