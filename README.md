# ECMA-402 Proposal: Intl.NumberFormat V3

**Status: Stage 3 (as of [July 2021](https://github.com/tc39/notes/blob/master/meetings/2021-07/july-15.md))**

- Spec: Intl.NumberFormat ([diff](https://tc39.es/proposal-intl-numberformat-v3/out/numberformat/diff.html), [proposed](https://tc39.es/proposal-intl-numberformat-v3/out/numberformat/proposed.html))
- Spec: Intl.PluralRules ([diff](https://tc39.es/proposal-intl-numberformat-v3/out/pluralrules/diff.html), [proposed](https://tc39.es/proposal-intl-numberformat-v3/out/pluralrules/proposed.html))
- Spec: Parameter Resolution ([diff](https://tc39.es/proposal-intl-numberformat-v3/out/negotiation/diff.html), [proposed](https://tc39.es/proposal-intl-numberformat-v3/out/negotiation/proposed.html))
- Spec: Annexes ([diff](https://tc39.es/proposal-intl-numberformat-v3/out/annexes/diff.html), [proposed](https://tc39.es/proposal-intl-numberformat-v3/out/annexes/proposed.html))

Intl.NumberFormat was first added in the initial Intl specification.  More recently, the ECMA-402 Proposal [Unified Intl.NumberFormat](https://github.com/tc39/proposal-unified-intl-numberformat) added several new key features.  This proposal, which I'm calling "Intl.NumberFormat V3", is one more batch of features that have been shown to be important to clients of this API.

## Motivation

In ECMA-402, we receive dozens of feature requests each year.  When forming this proposal, the author ([sffc](https://github.com/sffc/)) considered every feature request relating to Intl.NumberFormat and put them up against the following criteria:

1. The feature must have multiple stakeholders.
2. The feature must have robust prior art, e.g., in CLDR, ICU, or Unicode.
3. The feature must be difficult to implement in user land (such as a locale data dependency).

All parts of this proposal meet that bar, and furthermore, the author's intent is that all Intl.NumberFormat feature requests meeting that bar are part of this proposal.

## Features

### formatRange ([ECMA-402 #393](https://github.com/tc39/ecma402/issues/393))

This piece is modeled off of the [Intl.DateTimeFormat.prototype.formatRange
](https://github.com/tc39/proposal-intl-DateTimeFormat-formatRange) proposal.  It involves adding a new function `.formatRange()` to the Intl.NumberFormat prototype, in large part following the semantics introduced by `.formatRange()` in Intl.DateTimeFormat.

```javascript
const nf = new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "CHF",
  maximumFractionDigits: 0,
});
nf.formatRange(3, 5);  // "CHF 3–5"
```

Peer methods will also be added:

- `Intl.NumberFormat.prototype.formatRangeToParts`
- `Intl.PluralRules.prototype.selectRange` ([#16](https://github.com/tc39/proposal-intl-numberformat-v3/issues/16))

For example:

```javascript
const pl = new Intl.PluralRules("sl");
pl.selectRange(102, 201);  // "few"
```

The formatToParts semantics from Intl.DateTimeFormat will be adopted here: parts will gain a `source` property that will be either `"shared"`, `"startRange"`, or `"endRange"`.

```javascript
const nf = new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "GBP",
  currencyDisplay: "code",
  maximumFractionDigits: 0,
});
nf.formatRangeToParts(3, 5);
/*
[
  {type: "currency", value: "GBP ", source: "shared"}
  {type: "integer", value: "3", source: "startRange"}
  {type: "literal", value: "–", source: "shared"}
  {type: "integer", value: "5", source: "endRange"}
]
*/
```

When both sides of the range resolve to the same value after rounding, an approximately sign will be added.  The approximately sign is appended to the number in addition to a minus or plus sign, as applicable. ([#10](https://github.com/tc39/proposal-intl-numberformat-v3/issues/10), [#11](https://github.com/tc39/proposal-intl-numberformat-v3/issues/11), [#13](https://github.com/tc39/proposal-intl-numberformat-v3/issues/13))

```javascript
const nf = new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "EUR",
  maximumFractionDigits: 0,
});
nf.formatRange(2.9, 3.1);  // "~€3"

const nf = new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "EUR",
  signDisplay: "always",
});
nf.formatRange(2.999, 3.001);  // "~+€3.00"
```

If the second argument is less than the first argument, or if either is NaN, an error is thrown. ([#12](https://github.com/tc39/proposal-intl-numberformat-v3/issues/12))

```javascript
const nf = new Intl.NumberFormat("en-US");
nf.formatRange(500, 1/0);  // "500–∞"
nf.formatRange(500, 0/0);  // RangeError
nf.formatRange(500, 0);  // RangeError
```

#### Feature Detection

```javascript
if (Intl.NumberFormat.prototype.formatRange) {
  // feature is available
}
```

### Grouping Enum ([ECMA-402 #367](https://github.com/tc39/ecma402/issues/367))

Main Issue: [#3](https://github.com/tc39/proposal-intl-numberformat-v3/issues/3)

Currently, Intl.NumberFormat accepts a `{ useGrouping }` option, which accepts a boolean value.  However, as reported in the bug thread, there are several options users may want when specifying grouping.  This proposal is to make the following be valid inputs to `{ useGrouping }`:

- `false`: do not display grouping separators
- `"min2"`: display grouping separators when there are at least 2 digits in a group; for example, "1000" (first group too small) and "10,000" (now there are at least 2 digits in that group). (Bikeshed: [#23](https://github.com/tc39/proposal-intl-numberformat-v3/issues/23))
- `"auto"` (default): display grouping separators based on the locale preference, which may also be dependent on the currency.  Most locales prefer to use grouping separators.
- `"always"`: display grouping separators even if the locale prefers otherwise.
- `true`: alias for `"always"`
- `undefined` (default): alias for `"auto"`

In `resolvedOptions`, either `false` or one of the three strings will be returned.  This is an observable behavior change, because currently only the booleans `true` and `false` are returned.

#### Feature Detection

```javascript
if (new Intl.NumberFormat("und").resolvedOptions().useGrouping === "auto") {
  // feature is available
}
```

### New Rounding/Precision Options ([ECMA-402 #286](https://github.com/tc39/ecma402/issues/286))

Main Issue: [#8](https://github.com/tc39/proposal-intl-numberformat-v3/issues/8)

Additional Context: [Unified NumberFormat #9](https://github.com/tc39/proposal-unified-intl-numberformat/issues/9)

The following additional options are proposed to the Intl.NumberFormat options bag to control rounding behavior:

- `roundingPriority` = a string set to either `"auto"`, `"morePrecision"`, or `"lessPrecision"` (details below)
- `roundingIncrement` = a Number in the following list: « 1, 2, 5, 10, 20, 25, 50, 100, 200, 250, 500, 1000, 2000, 2500, 5000 »
  - Nickel rounding: `{ minimumFractionDigits: 2, maximumFractionDigits: 2, roundingIncrement: 5 }`
  - Dime rounding: `{ minimumFractionDigits: 2, maximumFractionDigits: 2, roundingIncrement: 10 }`
- `trailingZeroDisplay` = a string expressing the strategy for displaying trailing zeros on whole numbers:
  - `"auto"` = current behavior. Keep trailing zeros according to minimumFractionDigits and minimumSignificantDigits.
  - `"stripIfInteger"` = same as `"auto"`, but remove the fraction digits if they are all zero.

`roundingIncrement` cannot be mixed with significant digits rounding or any setting of `roundingPriority` other than `"auto"`.

#### Rounding Priority

Currently, Intl.NumberFormat allows for two rounding strategies: min/max fraction digits, or min/max significant digits.  Currently, if both min/max fraction digits and min/max significant digits are both specified, significant digit settings take priority and fraction digit settings are ignored.

The new option `roundingPriority` specifies two new strategies to resolve mixed fraction digits and significant digits settings.  To best express the new strategies, consider the following option bag:

```
{
    maximumFractionDigits: 2,
    maximumSignificantDigits: 2
}
```

The above options should be interpreted to mean:

1. Round the number at the hundredths place
2. Round the number after the second significant digit

Now, consider the number "4.321". `maximumFractionDigits` wants to round at the hundredths place, producing "4.32". However, `maximumSignificantDigits` wants to round after two significant digits, producing "4.3". We therefore have a conflict.

The new setting `roundingPriority` offers a hint on how to resolve this conflict. There are three options:

1. `roundingPriority: "auto"` means that significant digits always win a conflict.
2. `roundingPriority: "morePrecision"` means that the result with more precision wins a conflict.
3. `roundingPriority: "lessPrecision"` means that the result with less precision wins a conflict.

The whole result is taken atomically, including the trailing zeros. For example:

```
{
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
    minimumSignificantDigits: 2,
    maximumSignificantDigits: 6
}
```

Consider the input number "1".  `minimumFractionDigits` wants to retain trailing zeros up to the hundredths place, producing "1.00", whereas `minimumSignificantDigits` wants to retain only as many as are required to render two significant digits, producing "1.0". However, since `maximumSignificantDigits` has more precision (rounding to 10^-5, rather than 10^-2 for fraction precision), the resolved answer is "1.0".

#### Feature Detection

```javascript
if (new Intl.NumberFormat("und").resolvedOptions().roundingPriority) {
  // feature is available
}
```

### Interpret Strings as Decimals ([ECMA-402 #334](https://github.com/tc39/ecma402/issues/334))

The `format()` method currently accepts a Number or a BigInt, and strings are interpreted as Numbers.  This part proposes redefining strings to be represented as decimals instead of Numbers.

```javascript
const nf = new Intl.NumberFormat("en-US");
const string = "987654321987654321";
nf.format(string);
// Current:  "987,654,321,987,654,300"
// Proposed: "987,654,321,987,654,321"
```

A common use case is to store currency amounts as a BigInt of a small unit, like cents. This API can be used to retain full precision of the BigInt:

```javascript
const nf = new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: "EUR",
});
const bi = 1000000000000000110000n;
nf.format(bi + "E-6");
// Current:  "€1,000,000,000,000,000.10"
// Proposed: "€1,000,000,000,000,000.11"
```

The general syntax for decimal strings, essentially `#.#E#`, is a widely understood format in computing. The specific version we use is the ECMA-262 `StringNumericLiteral` grammar, which also allows non-base-10 numbers like hexadecimal and binary.

Arbitrary-precision decimal strings are intended to be used as pass-through for formatting. The champions do not intend for strings to become the de-facto standard for numeric computations in ECMAScript. For a general-purpose arbitrary-precision decimal type, see the [Decimal](https://github.com/tc39/proposal-decimal) proposal.

#### Feature Detection

```javascript
if (new Intl.NumberFormat("und").format("11111111111111111112").indexOf("2") !== -1) {
  // feature is available
}
```

### Rounding Modes ([ECMA-402 #419](https://github.com/tc39/ecma402/issues/419))

Main Issue: [#7](https://github.com/tc39/proposal-intl-numberformat-v3/issues/7)

Intl.NumberFormat always performs "half-up" rounding (for example, if you have 2.5, it gets rounded to 3).  However, we understand that there are users and use cases that would benefit from exposing more options for rounding modes.

The list of rounding modes is proposed to be:

1. ceil (toward +∞)
2. floor (toward -∞)
3. expand (away from 0)
4. trunc (toward 0)
5. halfCeil (ties toward +∞)
6. halfFloor (ties toward -∞)
7. halfExpand (ties away from 0; current behavior; default)
8. halfTrunc (ties toward 0)
9. halfEven (ties toward the value with even cardinality)

The behavior of these modes will reflect the [ICU user guide](https://unicode-org.github.io/icu/userguide/format_parse/numbers/rounding-modes.html), where "expand" maps to ICU "UP" and "trunc" maps to ICU "DOWN".

Rounding does not look at or change the sign bit of the number. Therefore, -0 and 0 are equivalent for the purposes of rounding. ([#21](https://github.com/tc39/proposal-intl-numberformat-v3/issues/21))

#### Feature Detection

```javascript
if (new Intl.NumberFormat("und").resolvedOptions().roundingMode) {
  // feature is available
}
```

### Sign Display Negative

Main Issue: [#17](https://github.com/tc39/proposal-intl-numberformat-v3/issues/17)

A new option `signDisplay: "negative"` is proposed, according to feedback from clients.  The new option will behave like `"auto"` except that the sign will not be shown on negative zero.

```javascript
var nf = new Intl.NumberFormat("en", {
  signDisplay: "negative"
});
nf.format(-1.0);  // -1
nf.format(-0.0);  // 0  (note: "auto" produces "-0" here)
nf.format(0.0);   // 0
nf.format(1.0);   // 1  (note: "exceptZero" produces "+1" here)
```

#### Feature Detection

```javascript
try {
  new Intl.NumberFormat("und", { signDisplay: "negative" });
  // feature is available
} catch(e) {
  // feature is NOT available
}
```

## Implementation Status

Shipped in:

- Safari 15.4
- Chrome 106
- Firefox 93

<details>
A V8 prototype of the latest spec text can be found in https://chromium-review.googlesource.com/c/v8/v8/+/2336146 w/ the flag --harmony_intl_number_format_v3
</details>
