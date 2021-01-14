# ECMA-402 Proposal: Intl.NumberFormat V3

**Status: Stage 2 (June 2020)**

- Spec: Intl.NumberFormat ([diff](https://tc39.es/proposal-intl-numberformat-v3/out/numberformat/diff.html), [proposed](https://tc39.es/proposal-intl-numberformat-v3/out/numberformat/proposed.html))
- Spec: Parameter Resolution ([diff](https://tc39.es/proposal-intl-numberformat-v3/out/negotiation/diff.html), [proposed](https://tc39.es/proposal-intl-numberformat-v3/out/negotiation/proposed.html))

Intl.NumberFormat was first added in the initial Intl specification.  More recently, the ECMA-402 Proposal [Unified Intl.NumberFormat](https://github.com/tc39/proposal-unified-intl-numberformat) added several new key features.  This proposal, which I'm calling "Intl.NumberFormat V3", is one more batch of features that have been shown to be important to clients of this API.

## Motivation

In ECMA-402, we receive dozens of feature requests each year.  When forming this proposal, the author ([sffc](https://github.com/sffc/)) considered every feature request relating to Intl.NumberFormat and put them up against the following criteria:

1. The feature must have multiple stakeholders.
2. The feature must have robust prior art, e.g., in CLDR, ICU, or Unicode.
3. The feature must be difficult to implement in user land (such as a locale data dependency).

All parts of this proposal meet that bar, and furthermore, the author's intent is that all Intl.NumberFormat feature requests meeting that bar are part of this proposal.

## formatRange ([ECMA-402 #393](https://github.com/tc39/ecma402/issues/393))

This piece is modeled off of the [Intl.DateTimeFormat.prototype.formatRange
](https://github.com/tc39/proposal-intl-DateTimeFormat-formatRange) proposal.  It involves adding a new function `.formatRange()` to the Intl.NumberFormat prototype, in large part following the semantics introduced by `.formatRange()` in Intl.DateTimeFormat.

```javascript
const nf = new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "EUR",
  maximumFractionDigits: 0,
});
nf.formatRange(3, 5);  // "€3–5"
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
  currency: "EUR",
  maximumFractionDigits: 0,
});
nf.formatRangeToParts(3, 5);
/*
[
  {type: "currency", value: "€", source: "startRange"}
  {type: "integer", value: "3", source: "startRange"}
  {type: "literal", value: "–", source: "shared"}
  {type: "integer", value: "5", source: "endRange"}
]
*/
```

Negative numbers will be allowed.

```javascript
const nf = new Intl.NumberFormat("en-US", {
  style: "percent",
});
nf.formatRange(-0.25, 0.50);  // "-25% – 50%"
```

When both sides of the range resolve to the same value after rounding, the display will fall back to the approximately sign (see below).

```javascript
const nf = new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "EUR",
  maximumFractionDigits: 0,
});
nf.formatRange(2.9, 3.1);  // "~€3"
```

## Grouping Enum ([ECMA-402 #367](https://github.com/tc39/ecma402/issues/367))

Currently, Intl.NumberFormat accepts a `{ useGrouping }` option, which accepts a boolean value.  However, as reported in the bug thread, there are several options users may want when speficying grouping.  This proposal is to add the following strings as valid inputs to `{ useGrouping }`:

- `"min2"`: display grouping separators when there are at least 2 digits in a group; for example, "1000" (first group too small) and "10,000" (now there are at least 2 digits in that group).
- `"auto"` (default): display grouping separators based on the locale preference, which may also be dependent on the currency.  Most locales prefer to use grouping separators.
- `"always"` (`true`): display grouping separators even if the locale prefers otherwise.

Previously considered was an option `"never"` corresponding to the current value `false`.  The current proposal does not add that option, because `false` will continue to work, and since non-empty strings are truthy, `if(nf.resolvedOptions().useGrouping)` will continue to work as expected.

## New Rounding/Precision Options ([ECMA-402 #286](https://github.com/tc39/ecma402/issues/286))

Additional Context: [Unified NumberFormat #9](https://github.com/tc39/proposal-unified-intl-numberformat/issues/9)

Currently, Intl.NumberFormat allows for two rounding strategies: min/max fraction digits, or min/max significant digits.  Those strategies cannot be combined.

I propose adding the following options to control rounding behavior:

- `roundingIncrement` = an integer, either 1 or 5 with any number of zeros.
  - Example values: 1 (default), 5, 10, 50, 100
  - Nickel rounding: `{ maximumFractionDigits: 2, roundingIncrement: 5 }`
  - Dime rounding: `{ maximumFractionDigits: 2, roundingIncrement: 10 }`
- `trailingZeros` = an enum expressing the strategy for resolving trailing zeros when combining min/max fraction and significant digits.
  - `"auto"` = obey mininumFractionDigits or minimumSignificantDigits (default behavior).
  - `"strip"` = always remove trailing zeros.
  - `"stripIfInteger"` = remove them only when the entire fraction is zero.
  - *optional:* `"keep"` = always keep trailing zeros according to the rounding magnitude.

The exact semantics of how to allow fraction digits and significant digits to interoperate are being tracked by [#8](https://github.com/tc39/proposal-intl-numberformat-v3/issues/8).

## Interpret Strings as Decimals ([ECMA-402 #334](https://github.com/tc39/ecma402/issues/334))

The `format()` method currently accepts a Number or a BigInt, and strings are interpreted as Numbers.  This part proposes redefining strings to be represented as decimals instead of Numbers.

```javascript
const nf = new Intl.NumberFormat("en-US");
const string = "987654321987654321";
nf.format(string);
// Current:  "987,654,321,987,654,300"
// Proposed: "987,654,321,987,654,321"
```

We will reference existing standards for interpreting decimal number strings where possible.

## Rounding Modes ([ECMA-402 #419](https://github.com/tc39/ecma402/issues/419))

Intl.NumberFormat always performs "half-up" rounding (for example, if you have 2.5, it gets rounded to 3).  However, we understand that there are users and use cases that would benefit from exposing more options for rounding modes.

The list of rounding modes could be:

1. `"halfUp"` (default)
2. `"halfEven"`
3. `"halfDown"`
4. `"ceiling"`
5. `"floor"`
6. `"up"`
7. `"down"`

## Approximately Sign

A new option will be added to signDisplay: `"approximately"`.  Using data newly available in CLDR 38, the output will be:

```javascript
const nf = new Intl.NumberFormat("en-US", {
  style: "percent",
});
nf.format(0.25);  // "~25%"
```

The approximately sign is currently not supported on negative numbers; it will be displayed only on positive numbers and zero.  In other words, `signDisplay: "approximately"` has the same behavior as `signDisplay: "always"`, but an approximately sign is used instead of a plus sign.  This limitation may change in the future.
