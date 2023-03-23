---
title: XSLTProcessor.transformToDocument()
slug: Web/API/XSLTProcessor/transformToDocument
page-type: web-api-instance-method
browser-compat: api.XSLTProcessor.transformToDocument 
---

{{APIRef("XSLT")}}

The `transformToDocument()` method of the {{domxref("XSLTProcessor")}} interface transforms the {{DOMxRef("Node")}} source given in parameter using the XSLT stylesheet associated with `XSLTProcessor` with the {{domxref("XSLTProcessor.importStylesheet")}}.

## Syntax

```js-nolint
transformToDocument(source)
```

### Parameters

- `source`
  - : The {{DOMxRef("Node")}} source to apply the XSLT stylesheet to.

### Return value

A {{domxref("Document")}}. The actual interface depends on the [output method](https://www.w3.org/TR/1999/REC-xslt-19991116#output) of the stylesheet:

| Output method | Result interface                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------ |
| `html`        | {{domxref("HTMLDocument")}}                                                                            |
| `xml`         | {{domxref("XMLDocument")}}                                                                             |
| `text`        | {{domxref("XMLDocument")}} with a single root element `<transformiix:result>` with the text as a child |

## Examples

## Specifications

{{Specifications}}

## Browser compatibility

{{Compat}}

## See also

- {{domxref("XSLTProcessor.transformToDocument()")}}