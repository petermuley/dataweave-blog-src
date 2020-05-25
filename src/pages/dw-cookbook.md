---
author: "Michael"
date: "2020-05-22"
path: "/dw-cookbook"
title: "Data-weave Cookbook"
---

## Recipes:
* [Appending XML namespaces](#recursively-appending-xml-namespaces)
* [Bitwise math](#bitwise-math)
* [Email validation](#rfc822-compliant-email-validation)
* [Power sets](#powerset)

Everything below is mainted in my Gist: https://gist.github.com/thisisverytricky

---

## Recursively appending XML namespaces

```data-weave
%dw 2.0

fun appendNamespace(data, nsSelector: (k: Key) -> Namespace | Null) =
  data match {
    case is Array -> data map appendNamespace($, nsSelector)
    case is Object -> data mapObject do {
      var ns0 = nsSelector($$)
      ---
      if (ns0 != null) ns0#"$($$)": appendNamespace($, nsSelector)
      else ($$): appendNamespace($, nsSelector)
    }
    else -> data
}
```

Example payload:
```JSON
{
  "GetOrderById": {
    "OrderId": 656734
  }
}
```
Simple use, mapping all keys to the same namespace.
```data-weave
%dw 2.0
import dw::modules::Namespaces //import the function from /src/main/resources/dw/modules/Namespaces.dwl

ns soapenv http://schemas.xmlsoap.org/soap/envelope/
ns tem http://tempuri.org/

output application/xml
---
{
    soapenv#Envelope: {
        soapenv#Header:null,
        soapenv#Body : payload appendNamespace tem
    }
}
```
Of course, our namespace selector is a function, so we could do something like call a system API via `lookup` to dynamically pull in a map:
```data-weave
%dw 2.0
import dw::modules::Namespaces //import the function from /src/main/resources/dw/modules/Namespaces.dwl

ns soapenv http://schemas.xmlsoap.org/soap/envelope/

var nsMap = lookup('get-namespace-map')
---
{
    soapenv#Envelope: {
        soapenv#Header:null,
        soapenv#Body : payload appendNamespace do {
          var ns0 = nsMap[$]
          ---
          if (ns0 != null) ns0 as Namespace //coerce our JSON structure to a Namespace type
          else null
        }
    }
}
```

---

## Bitwise Math
Data-weave doesn't have built in operators for bitwise math. We could implement this in java.. but lets take a look at a simple implementation in data-weave that we can then reuse as a module later.

```data-weave
%dw 2.0
import dw::core::Numbers
import dw::core::Strings

fun AND(lo: Number, ro: Number) = do {
    var binary = getBinary(lo, ro)
    ---
    Numbers::fromBinary(binary.left map ($ as Number * binary.right[$$] as Number) reduce ($$++$))
}

fun OR(lo: Number, ro: Number) = do {
    var binary = getBinary(lo, ro)
    ---
    Numbers::fromBinary(binary.left map (if ($ == "1" or binary.right[$$] == "1") "1" else "0") reduce ($$++$))
}

fun XOR(lo: Number, ro: Number) = do {
    var binary = getBinary(lo, ro)
    ---
    Numbers::fromBinary(binary.left map (if ($ == binary.right[$$]) "0" else "1") reduce ($$++$))
}

fun getBinary(lo: Number, ro: Number) = do {
    var loB = Numbers::toBinary(lo)
    var roB = Numbers::toBinary(ro)
    var size = max([sizeOf(loB), sizeOf(roB)]) default 0
    ---
    { 
        left: Strings::leftPad(loB, size, '0') splitBy '',
        right: Strings::leftPad(roB, size, '0') splitBy ''
    }
}
```

A lot easier to implement than you might think! All we have to do is convert it to binary and then map. Probably not the most efficient method for doing this, but a fun thing to play with. We can then use it to do something like implement the `powerSet` function below.

---

## Powerset

Need to get the power set? Well here you go! Making use of our previous Bitwise module along with some basic set theory we all learned at some point, we can do this:

```data-weave
%dw 2.0

import Bitwise

fun powerSet(set: Array) = do {
    var iterable = (0 to pow(2,sizeOf(set))-1) as Array
    ---
    iterable map (item) -> set filter (Bitwise::AND(item,pow(2,$$)) != 0)
}
```

Easy peasy!

---

## RFC822 Compliant Email Validation

If the regex is valid in java, its valid in data-weave!

```data-weave
%dw 2.0
output application/json
var isValidEmail = (e) -> e matches /(?:(?:\r\n)?[ \t])*(?:(?:(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*))*@(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*))*|(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*)*\<(?:(?:\r\n)?[ \t])*(?:@(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*))*(?:,@(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*))*)*:(?:(?:\r\n)?[ \t])*)?(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*))*@(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*))*\>(?:(?:\r\n)?[ \t])*)|(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*)*:(?:(?:\r\n)?[ \t])*(?:(?:(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*))*@(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*))*|(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*)*\<(?:(?:\r\n)?[ \t])*(?:@(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*))*(?:,@(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*))*)*:(?:(?:\r\n)?[ \t])*)?(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*))*@(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*))*\>(?:(?:\r\n)?[ \t])*)(?:,\s*(?:(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*))*@(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*))*|(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*)*\<(?:(?:\r\n)?[ \t])*(?:@(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*))*(?:,@(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*))*)*:(?:(?:\r\n)?[ \t])*)?(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\"(?:[^\\"\r\\]|\\.|(?:(?:\r\n)?[ \t]))*\"(?:(?:\r\n)?[ \t])*))*@(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*)(?:\.(?:(?:\r\n)?[ \t])*(?:[^()<>@,;:\\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[ \t])+|\Z|(?=[\[\"()<>@,;:\\\".\[\]]))|\[([^\[\]\r\\]|\\.)*\](?:(?:\r\n)?[ \t])*))*\>(?:(?:\r\n)?[ \t])*))*)?;\s*)/
---
isValidEmail("michael.jones@mulesoft.com")
```