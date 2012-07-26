#JSON formatted SData responses

By Tudor Milencovici

Version: 0.7a

## Purpose

This paper describes the JSON format for SData content. The focus is placed on the structural aspects of content and representation of the protocol-mandated information. The details of metadata usage are handled in a separate document.

The document is aimed at a technical audience who has had a prior exposure to the SData protocol.

The information in this document has not been approved by the standard body governing SData. As such, it should be treated as a preview and a snapshot to the upcoming JSON aspect in SData.

## Introduction to the JSON formalism

JSON is an open, text-based data exchange format (see [RFC 4627](http://www.ietf.org/rfc/rfc4627.txt)). Like XML, it is human-readable, and platform independent. Data formatted according to the JSON standard is lightweight and can be parsed by JavaScript implementations with incredible ease, making it an ideal data exchange format for web applications.

The information transported through JSON is always in form of the following basic types:
* strings: double-quoted Unicode (UTF-8 by default), with backslash escaping
* numbers: double-precision floating point format
* Boolean values: `true` or `false`
* `null`

The JSON building block is the name-content tuple. The name is always a quote-enclosed string and is separated from the content by a ":" character. The content of a JSON tuple can be either:

* A value in one of the basic types, example:
    `"firstName" : "Ted"`
* An **ordered** set of contents, delimited by square brackets ("[" "]"), example:
    `"continents" : [ "Europe" , "Africa", "Asia", "Americas",
"Australia", "Antarctica" ]`
* A JSON object. JSON objects are **unordered** sets of name-content pairs. The only restriction is that within a set the names are unique. The name-content pairs are separated by a comma (",") and enclosed by curly brackets ("{" "}"), example:
    `{ "street" : "Augartenstrasse", "number" : 1, "city" : "Karlsruhe", "country" : "Germany"}`

This is it! JSON is simple, yet very expressive and also less verbose than XML.

## Formatting native objects in JSON

Representing a native data container in JSON is a straightforward task, guided by the following rules:
* Scalar properties and their values are represented as JSON name-value pairs
* Structural elements and sub-objects are represented as JSON objects
* Collections are represented as JSON arrays

### Expressing a simple data object in JSON

Let us consider a simplified `SalesOrderLine` object of the following structure and values:
* productName -> "iPad"
* productID -> 437
* lineNumber -> 1
* orderedQuantity -> 2
* unitPrice -> 399.95

We note that the object contains scalar properties that translate easily into JSON name-value pairs. Thus, the JSON representation would be:

``` javascript
{
	"productName": "iPad",
	"productID": "437",
	"lineNumber": 1,
	"orderedQuantity": 2,
	"unitPrice": 399.95
}
```

### Expressing a data object with embedded structures in JSON

Let us alter the example above by grouping the product-related properties in an own structure:

* product
    * name -> "iPad"
    * ID -> "437"
* lineNumber -> 1
* orderedQuantity -> 2
* unitPrice -> 399.95

In this case, the product structure would be represented as a JSON object in its own right. Thus, the JSON representation is:

``` javascript
{
    "product": {
		"productName": "iPad",
		"productID": "437"
	},
	"lineNumber": 1,
	"orderedQuantity": 2,
	"unitPrice": 399.95
}
```

### Expressing an object with an embedded collection

A usual example of an object containing a collection of data elements is the `SalesOrder`. It contains a collection of line items for the individual positions in the order itself. A simplified `SalesOrder` structure is:

* orderDate
* shipDate
* contact
    * firstName -> "John"
    * lastName -> "Doe"
    * email -> "john.doe@acme.com"
* orderLines – *containing*
    * lineNumber -> 1
    * orderedQuantity -> 2
    * unitPrice -> 399.95
    * product
        * name -> "iPad"
        * ID -> 437
* ***and***
    * lineNumber -> 2
    * orderedQuantity -> 1
    * unitPrice -> 323.00
    * product
        * name -> "Samsung Galaxy S2"
        * ID -> 932
* subtotal -> 1021.95

In the translation we observe the following:

* `orderDate`, `shipDate` and `subtotal` are scalar properties that should be represented as JSON
name-value pairs
* `contact` is a structural element, and therefore to be represented as a JSON object. The contact properties `firstName`, `lastName` and `email` are scalar properties expressed as name-value JSON pairs
* `orderLines` is a collection, and should be represented as a JSON array. The array elements, the individual line items are structural elements translating into JSON objects.
* `product` is a structural element, mapping to a JSON object
* All remaining properties are scalar, hence representable as name-value JSON pairs.

The resulting JSON representation is:

``` javascript
{
    "orderDate": "2001-07-01",
	"shippedDate": null,
	"contact": {
		"firstName": "John",
		"lastName": "Doe",
		"email": "john.doe@acme.com"
	}, // end contact
	"orderLines": [{
		"lineNumber": 1,
		"orderedQuantity": 2,
		"unitPrice": 399.95,
		"product": {
			"name": "iPad",
			"ID": "437"
		} //end product
	}, {
		"lineNumber": 2,
		"orderedQuantity": 1,
		"unitPrice": 323.00,
		"product": {
			"name": "Samsung Galaxy S2",
			"ID": "932"
		} //end product
	}], // end orderLines
	"subtotal": 1021.95
}
```

## Requesting JSON formatted SData content

The JSON and ATOM+xml formats are of equal value from a protocol perspective but providers will likely choose a default format among the two. To ensure that, if a provider supports JSON, the response is in JSON, the consumer needs to specify this in its `HTTP Accept` header of the request by:

    Accept: application/json;vnd.sage=sdata

An alternative is to use the `format` query parameter:

    http://www.example.com/sdata/myApp/myContract/prod/accounts?format=application/json;vnd.sage=sdata

The first mechanism should be used when the consumer (the user agent) will systematically request the JSON format. The second one is more appropriate when the consumer normally uses ATOM but switches to JSON occasionally.

## JSON responses

Provider responses to requests in (and for) JSON are in one of the following forms:

* An **entry**: this is the representation of an individual resource and contains the native, JSON formatted objects
* A **feed**: is a collection of entries
* A **diagnosis** : returned in case something went wrong with the request
* A **tracking object**: returned for an asynchronous call to enable subsequent polling of results

### JSON entries

Entries encompass a single resource such as `Customer`, or `SalesOrder`. A payload will contain a single entry will only when the request operates specifically a single entry (ex: GET on a [single resource URL](http://interop.sage.com/daisy/sdata/AnatomyOfAnSDataURL/SingleResourceURL.html) or PUT/POST/DELETE operations).

In addition to the native properties of the resource, SData allows within an entry the presence of several protocol-defined properties meant to ease consumption. These are described in the table below:

<table>
<tr><td></td><td>Description</td><td>Compliance</td></tr>
<tr><td>$url</td>
    <td>URL pointing to the resource.<br/>
        The URL should be represented as relative to the value of the $baseUrl of the enclosing feed.<br/>
Examples:<br/>
given the "$baseUrl": "http://ex.com/MyApp/-/-/"<br/>
 "$url": "customers(‘1234’)"<br/>
 "$url": "customers?where=$uuid eq ‘ab1C43sd-c1asd-c2sT’"<br/>
If a $baseUrl property is not specified, then the URL <b>MUST</b> be absolute.<br/>
Examples:<br/>
"$url": " http://ex.com/MyApp/-/-/customers(‘1234’)"<br/>
"$url": "http://ex.com/MyApp/-/-/customers?where=$uuid eq ‘ab1C43sd-c1asd-c2sT’"</td>
    <td>MAY</td></tr>
<tr><td>$key</td>
    <td>the native primary key identifying the resource</td></td>
    <td>MAY</td></tr>
<tr><td>$uuid</td>
    <td>UUID identifying the resource</td>
    <td>MAY</td></tr>
<tr><td>$title</td>
    <td>humanly readable description of the resource</td>
    <td>MAY</td></tr>
<tr><td>$updated</td>
    <td>the time stamp of the last update of the resource, formatted according to ISO 8601 dateTime specification</td>
    <td>MAY</td></tr>
<tr><td>$etag</td>
    <td>opaque identifier assigned by the provider to a version of a resource</td>
    <td>MAY</td></tr>
<tr><td>$properties<sup><a href="#foot-1" name="ref-1">1</a></sup></td>
    <td>Object containing metadata for the properties of the resource</td>
    <td>MAY</td></tr>
<tr><td>$links<sup><a href="#foot-2" name="ref-2">2</a></td>
    <td>Object containing links that present functional aspects of the resource (ex: edit, lookup, create). They are to be understood as hypermedia controls</td>
    <td>MAY</td></tr>
<tr><td>$diagnosis</td>
    <td>Object containing a more detailed indication of errors and warnings encountered by the provider during the execution of a request</td>
    <td>MAY</td></tr>
<tr><td>native properties</td>
    <td>Native properties of the object</td>
    <td>MAY</td></tr>
</table>

An example is the JSON variant of the Typical SData Entry in the SData documentation:

``` javascript
...
$baseUrl: "http://www.acme.com/MyApp/-/-/",
...
{
    "$url": "salesOrders('43660')",
	"$updated": "2008-03-31T13:46:45Z",
	"$key": "43660",
	"$title": "Sales Order 43660",
	"$etag": "gJaGtgHyuAwW6jMI4i0njA==",
	"orderDate": "2001-07-01",
	"shipDate": null,
	"contact": {
		"$url": "contacts('216')",
		"$key": "216"
	},
	"subTotal": 1553.10
}
```
### JSON feeds

Feeds are collections of entries, returned by operations targeting several resources in parallel such as read or query operations on resource kinds. A feed is a JSON object with all entries contained in the `$resources` array – this is the only required property of a feed.

A feed may also contain a set of SData-defined properties, which are described in the table below:

<table>
<tr><td></td><td>Description</td><td>Compliance</td>
<tr><td>$resources</td><td>Array containing the individual entries</td><td><b><text color="red">MUST</b></td></tr>
<tr><td>$baseUrl</td><td>
URL leading to the resource kind level of an application. The URL MUST end in a "/"
Example:
"$baseUrl": "http://www.acme.com/MyApp/MyContract/-/"</td><td>MAY</td></tr>
<tr><td>$url</td>
    <td>
        URL pointing to the resources returned.<br/>
        The URL should be represented as relative to the $baseUrl value.<br/>
        Examples:
        <ul><li>"$url": "customers"</li>
            <li>"$url": "customers?where=name ge 'm'"</li></ul>
        If a $baseUrl property is not specified, then the URL MUST be absolute.<br/>
        Examples:
        <ul><li>"$url": " http://ex.com/MyApp/-/-/customers"</li>
            <li>"$url": "http://ex.com/MyApp/-/-/customers?where=name ge 'm'"</li></ul></td>
    <td>MAY</td></tr>
<tr><td>$title</td>
    <td>humanly readable description of the resource</td>
    <td>MAY</td></tr>
<tr><td>$updated</td>
    <td>the time stamp of the last update of the resource, formatted according to <a href="http://www.w3.org/TR/NOTE-datetime">ISO 8601 dateTime</a> specification</td><td>MAY</td></tr>
<tr><td>$links<sup><a href="#foot-3" name="ref-3">3</a></sup></td>
    <td>Object containing links that present functional aspects of the feed (ex: refresh, first-page, schema, template, ...). They are to be understood as <a href="http://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven">hypermedia controls</a></td><td>MAY</td></tr>
<tr><td>$diagnosis</td>
    <td>Object containing a more detailed indication of errors and warnings encountered by the provider during the execution of a request</td><td>MAY</td></tr>
</table>

An example is the JSON variant of the [SData Typical Feed](http://interop.sage.com/daisy/sdata/16-DSY.html?branch=1):

``` javascript
{
    "$baseUrl": "https://www.example.com/MyApp/-/-/""$url": "salesOrders",
	"$title": "Sage App | Sales Orders",
	"$totalResults": 31465,
	"$startIndex": 1,
	"$itemsPerPage": 10,
	"$resources": [{
		"$updated": "2008-03-31T13:46:45Z",
		"$key": "43660",
		"$title": "Sales Order 43660",
		"$etag": "gJaGtgHyuAwW6jMI4i0njA==",
		"orderDate": "2001-07-01",
		"shipDate": null,
		"contact": {
			"$url": "contacts('216')",
			"$key": "216"
		},
		"subTotal": 1553.10
	}, {
		"$updated": "2008-03-31T13:46:45Z",
		"$key": "43661",
		"$title": "Sales Order 43660",
		"$etag": "3nqPeQqoGoxQB5xf3NIijw==",
		"orderDate": "2001-07-01",
		"shipDate": null,
		"contact": {
			"$url": "contacts('281')",
			"$key": "281"
		},
		"subTotal": 39422.12
	}]
}
```

### JSON diagnosis

The [diagnosis](http://interop.sage.com/daisy/sdata/AnatomyOfAnSDataFeed/ErrorPayload.html) is an object that contains information about the status (information, warning, error) of a request's execution. Such an object **MUST** be present in a response **if** errors were encountered during the execution. The xml format of diagnosis is described in the SData protocol in the [3.10: Error payload section](http://interop.sage.com/daisy/sdata/AnatomyOfAnSDataFeed/ErrorPayload.html).

The JSON format of the diagnosis objects supports the following properties:

<table>
<tr><td></td><td>Description</td><td>Compliance</td></tr>
<tr><td>$severity</td>
    <td>Severity of the diagnosis entry. Possible values are:
    <ul><li>Info</li>
        <li>Warning</li>
        <li>Transient</li>
        <li>Error</li>
        <li>Fatal</li></ul></td>
    <td><b>MUST</b></td></tr>
<tr><td>$sdataCode</td>
    <td>The SData diagnosis code</td>
    <td><b>MUST</b></td></tr>
<tr><td>$applicationCode</td>
    <td>Application specific diagnosis code</td>
    <td>MAY</td></tr>
<tr><td>$message</td>
    <td>Friendly message for the diagnosis</td>
    <td><b>SHOULD</b></td></tr>
<tr><td>$stackTrace</td>
    <td>Stack trace – to be used with care</td>
    <td>MAY</td></tr>
<tr><td>$payloadPath</td>
    <td>XPath expression that refers to the payload element responsible for the error</td>
    <td>MAY</td></tr>
</table>

An example of a diagnoses JSON object is shown below:

``` javascript
{
    "$diagnoses": [{
		"$severity": "error",
		"$sdataCode": "BadWhereSyntax",
		"$message": "Invalid query syntax",
		"$applicationCode": "2403"
	}]
}
```

### JSON tracking

The tracking object **MUST** be returned by a provider in response of an [asynchronous operation](http://interop.sage.com/daisy/sdata/ServiceOperations/AsynchronousOperations.html). SData describes a set of [properties that can be provided in a tracking](http://interop.sage.com/daisy/sdata/AnatomyOfAnSDataFeed/TrackingPayload.html) object. For JSON, these are:

<table>
<tr><td></td><td>Description</td><td>Compliance</td></tr>
<tr><td>$phase</td>
    <td>End user message describing the current phase of the operation.</td>
    <td>MAY</td></tr>
<tr><td>$phaseDetail</td>
    <td>Detailed message for the progress within the current phase.</td>
    <td>MAY</td></tr>
<tr><td>$progress</td>
    <td>Percentage of operation completed.</td>
    <td>MAY</td></tr>
<tr><td>$elapsedSeconds</td>
    <td>Time elapsed since operation started, in seconds.</td>
    <td><b>MUST</b></td></tr>
<tr><td>$remainingSeconds</td>
    <td>Expected remaining time, in seconds.</td>
    <td>MAY</td></tr>
<tr><td>$pollingMillis</td>
    <td>Delay (in milliseconds) that the consumer should use before polling the service again.
    <td><b>MUST</b></td></tr>
</table>

An example of a tracking JSON object is shown below:

``` javascript
{
    "$tracking": {
		"$phase": "Archiving FY 2007",
		"$phaseDetail": "Compressing file archive.dat",
		"$progress": 12.0,
		"$elapsedSeconds": 95,
		"$remainingSeconds": 568,
		"$pollingMillis": 500
	}
}
```

## A note to SData ATOM+xml users

If you are intimately aware of the ATOM+xml format of SData as described in the [SData 1.x standard](http://interop.sage.com/daisy/sdata/Introduction.html), you will have noticed that some ATOM mandated elements are not present in the JSON objects of this document. The reason is that these are meaningful in a syndication context but have little relevance in the general SData application. The elements omitted are:

* Envelope markup `<feed>` : no longer needed due to representation as a JSON object
* Envelope markup `<entry>` : no longer needed due to representation as a JSON object
* `<id>` : information is carried by the $url element
* `<link rel='self'>` : information is carried by the $url element
* `<author>`
* `<category>`

---

<a name="foot-1" href="#ref-1">1</a> <br/>
<a name="foot-2" href="#ref-2">2</a> These will be described in detail in another paper on JSON metadata and its usage.
<a name="foot-3" href="#ref-3">3</a> These will is described in detail in another paper on JSON metadata and its usage.

