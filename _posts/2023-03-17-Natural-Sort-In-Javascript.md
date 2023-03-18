---
layout: post
title:  "Natural sort in javascript"
date:   2023-03-17 20:44:01
categories: post
---
{% newthought "Use the 'standard' natural sort" %} in javascript.<!--more--> Have you ever implemented a naive alphabetical sort only to notice that the order isn't right due to different cases or numeric values? Using natural or 'human readable' sort {% sidenote 'naturalSort', 'See [Natural sort order](https://en.wikipedia.org/wiki/Natural_sort_order)' %} for some list or other is often a requirement, even if it's not explicitly stated.

{% newthought "Don't waste time" %} trying to implement your own comparator. Or import yet another npm package. Javascript has a 'standard' way of doing this with the `Intl.Collator`{% sidenote 'collator', 'See [Intl.Collator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator)' %}.

{% newthought "Intended to help" %} with language-sensitive comparisions it can also be used for natural sort. For example:

```javascript
const items = ["z11", "z2", "Z11", "Z2"];
const comparator = new Intl.Collator(undefined, {
  numeric: true,
  caseFirst: "lower",
}).compare;
const sorted = [...items].sort(comparator);
console.log(sorted);
// output: [ 'z2', 'Z2', 'z11', 'Z11' ]
```
The first parameter for the `Collator` is the language, undefined uses the default, which seems to be English. Then the options object allows you to add `numeric` to treat numbers within strings as numbers, i.e. order 2 before 11. The `caseFirst` property allows you to decide whether upper or lower. [Check out the full api at MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator/Collator).