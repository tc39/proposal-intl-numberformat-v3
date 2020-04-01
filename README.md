# ECMA-402 Proposal: Intl.NumberFormat V3

**Status: Stage 1 (March 2020)**

Intl.NumberFormat was first added in the initial Intl specification.  More recently, the ECMA-402 Proposal [Unified Intl.NumberFormat](https://github.com/tc39/proposal-unified-intl-numberformat) added several new key features.  This proposal, which I'm calling "Intl.NumberFormat V3", is one more batch of features that have been shown to be important to clients of this API.

Champion: Shane F. Carr (sffc)

## Part 1: formatRange ([ECMA-402 #393](https://github.com/tc39/ecma402/issues/393))

This piece is modeled off of the [Intl.DateTimeFormat.prototype.formatRange
](https://github.com/tc39/proposal-intl-DateTimeFormat-formatRange) proposal.  It involves adding a new function `.formatRange()` to the Intl.NumberFormat prototype, in large part following the semantics introduced by `.formatRange()` in Intl.DateTimeFormat.

```javascript
const nf = new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "EUR",
});
nf.formatRange(3, 5);  // "€3–5"
```

Peer methods will also be added:

- `formatRangeToParts`

## Part 2: Better interop between PluralRules and NumberFormat ([ECMA-402 #397](https://github.com/tc39/ecma402/issues/397))

It is common to want to perform both plural rules selection and number formatting together.  Currently, you need to create two separate objects with the same set of options, and we have had multiple bug reports come in about this process being unintuitive.  Further, the Intl.NumberFormat implementation requires plural rules under the hood anyway, so users are not getting optimal performance by having to create another object for the plural rules selection.

One possibility, to be hammered down in Stage 1, is adding a method `formatSelect` that performs formatting and plural selection in one step.

```javascript
const fmt = new Intl.NumberFormat("fr-FR", {
    notation: "compact"
});
const { string, pluralForm } = fmt.formatSelect(2.5e6);
console.log(string, pluralForm);
// "2.5 M" many
```

Peer methods would also be added:

- `formatToPartsSelect`
- `formatRangeSelect`
- `formatRangeToPartsSelect`

## Part 3: Grouping Enum ([ECMA-402 #367](https://github.com/tc39/ecma402/issues/367))

Currently, Intl.NumberFormat accepts a `{ useGrouping }` option, which accepts a boolean value.  However, as reported in the bug thread, there are several options users may want when speficying grouping.  This proposal is to add the following strings as valid inputs to `{ useGrouping }`:

- `"never"` (`false`): do not display grouping separators.
- `"min2"`: display grouping separators when there are at least 2 digits in a group; for example, "1000" (first group too small) and "10,000" (now there are at least 2 digits in that group).
- `"auto"` (default): display grouping separators based on the locale preference, which may also be dependent on the currency.  Most locales prefer to use grouping separators.
- `"always"` (`true`): display grouping separators even if the locale prefers otherwise.

## Part 4: New Rounding/Precision Options ([ECMA-402 #286](https://github.com/tc39/ecma402/issues/286))

Additional Context: [Unified NumberFormat #9](https://github.com/tc39/proposal-unified-intl-numberformat/issues/9)

Currently, Intl.NumberFormat allows for two rounding strategies: min/max fraction digits, or min/max significant digits.  Those strategies cannot be combined.

I propose adding the following options to control rounding behavior:

- `precision`: `"one"` (default) or `"nickel"`, to allow for nickel rounding, like 1.03 formatted to 1.05.  This is a popular feature in certain cash currencies, and it is easy to implement.
- `fractionDigits`: Set both `minimumFractionDigits` and `maximumFractionDigits` at the same time, and also enable interoperability with significant digits and hiding fraction digits when near an integer.  Exact details to follow.

## Part 5: Scale Option ([ECMA-402 #334](https://github.com/tc39/ecma402/issues/334))

Currently, only Number and BigInt are the two types that Intl.NumberFormat is able to format.  However, we have had users requesting the ability to format numbers that can't be represented by those two types.

This proposal is to add a `{ scale }` option to Intl.NumberFormat that multiplies the input by a power of 10 before formatting.  When `{ style: "unit" }`, `scale` defaults to `-2`; otherwise, it defaults to `0`.

```javascript
const nf = new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "EUR",
  scale: -6,
});
const micros = 1_000_000n;
nf.format(micros);  // "€1.00"
```

## Part 6: Interpret Strings as Decimals ([ECMA-402 #334](https://github.com/tc39/ecma402/issues/334))

The `format()` method currently accepts a Number or a BigInt, and strings are interpreted as Numbers.  This part proposes redefining strings to be represented as decimals instead of Numbers.

```javascript
const nf = new Intl.NumberFormat("en-US");
const string = "987654321987654321";
nf.format(string);
// Current:  "987,654,321,987,654,300"
// Proposed: "987,654,321,987,654,321"
```

We will reference existing standards for interpreting decimal number strings where possible.

## Part 7: Rounding Modes ([ECMA-402 #419](https://github.com/tc39/ecma402/issues/419))

Intl.NumberFormat always performs "half-up" rounding (for example, if you have 2.5, it gets rounded to 3).  However, we understand that there are users and use cases that would benefit from exposing more options for rounding modes.

The list of rounding modes could be:

1. `"halfUp"` (default)
2. `"halfEven"`
3. `"halfDown"`
4. `"ceiling"`
5. `"floor"`
6. `"up"`
7. `"down"`
