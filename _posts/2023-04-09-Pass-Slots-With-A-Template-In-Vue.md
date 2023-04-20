---
layout: post
title:  "Pass slots with a template in Vue"
date:   2023-04-09 15:45:02
categories: post
---
{% newthought "Using a template to pass a slot to a grandchild" %} in Vue.<!--more--> I was working on some features in a Vue frontend that had reused components in the following fashion: {% fullwidth "assets/img/slots.drawio.svg" "The Components" %}


Intended to help with language-sensitive comparisions it can also be used for natural sort. For example:

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