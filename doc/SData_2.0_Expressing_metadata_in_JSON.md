#SData 2.0: Expressing metadata in JSON

Version 0.4a

By Tudor Milencovici

## Introduction

The interplay between consumers and providers is a request-response interaction underlined by data transfers between the two parties.

Initially, consumers were static and coded-to-fit the information on the provider; in such a setting, the raw information (i.e. records exposed by the application) is sufficient, since the other aspects (user interface, handling of information, etc.) are hard-coded in the client. In time, clients became more versatile, able to adapt to the dynamic requirements of one (or several) providers. Influencing and controlling the client behavior relies on out-of-band delivery of descriptive information to the client: the metadata.

SData 2.0, particularly in its JSON format, is focused on the support of intelligent clients or third-party applications and views the transport and handling of metadata as an important protocol element. This document describes the SData manner in which providers expose JSON metadata at feed, entry and property level.

Compliance to the features presented in this document is stated in the last section, <a href="#compliance">Compliance</a>.

## Attributes of a JSON metadata representation

Practical experiences, as well as feedback from teams using JSON, indicate that the metadata expression should possess at least the following attributes:

* Delivery in JSON
* Availability at the resource kind level and ideally, with even finer granularity
* Retrievable
    * alongside a regular feed (similar to the `includeSchema` URL parameter)
    * from a predefined location (similar to the `$schema` URL segment)
* Ability to express the metadata defined in the SData 1.x specification
* Ability to override at any level (feed/entry/property)
* As unobtrusive as possible and not overly verbose

## A look at the existing SData metadata

In SData 1.x, metadata is found in two places:

* Represented in an XSD schema
* Embedded in a response (ex: links, ATOM elements)

### Schemas

Schemas are the subject of a significant portion of the SData 1.x protocol. The reasons to give them such attention are manifold:

* They allow a clear description of the resource kinds available in a contract
* They are expressed in a standard language (XSD)
* They can be used by tools to generate native objects and validate the contents passed between consumer and provider

Practical experience however, uncovered a number of disadvantages:

* Schemas are a static description of resource kinds. Without an override mechanism available at the level of feeds/entries/properties, it is too inflexible for the needs of dynamic consumers
* Schemas are costly to generate/maintain if the underlying application changes
* Schema fragments, such as the information presented in the `$schema` of a resource kind, are
difficult to create/maintain and extremely difficult to use for validation. This forces a client to operate on the whole schema – a resource intensive endeavor.
* Operating on XSDs is costly, especially in a technology with little affinity towards XML - such as JavaScript clients
* The XSD language is too formal and does not exhibit the lightness and directness a typical JSON consumer is accustomed to

The ideal role for schemas seems therefore to be in the context of application/systemic integration and it is to be expected that schemas will remain an important aspect in this area. When matched against the criteria laid out for a JSON solution, schemas are a poor fit.

### <a name="embedded">Embedded metadata</a>

A single look at the typical feed example in the SData specification ([Section 3.1](http://interop.sage.com/daisy/sdata/AnatomyOfAnSDataFeed/TypicalFeed.html)) reveals the extensive amount of metadata contained in a feed:
* ATOM elements such as `title`, `id`, `author`, `updated`
* Links : `self`, `first`, `last`, `template`, `Post`
* SData elements and attributes: `key`, `url`, `lookup`, `categories`

These account for roughly 90% of the response volume, which is prohibitive in a web/mobile context where the bandwidth is usually limited and costly. Moreover, should this information be cached by a client, the memory requirements on the client are extensive.

It is interesting to observe that the embedded metadata is largely redundant (ex: `id` element and `'self'` link); also, most of the URL information differs from one entry to another by a single piece of information (ex: the URLs of two `SalesOrder` resources differ only in the value of the key). This suggests that a pattern-oriented composition mechanism would significantly ‘slim-down’ responses, but such a mechanism is contrary to the [ATOM specification](http://www.ietf.org/rfc/rfc4287) (ex: `ID` field as defined in section 4.2.6).

There is therefore an opportunity to design things more efficiently in JSON where we are no longer constrained by the ATOM syndication requirements.

### Observations

The above two aspects as presented in SData 1.x - taken together or individually - do not possess the attributes desired for an optimal JSON solution. An ideal solution would:

* keep the metadata by-and-large separate from the raw information
* provide a cacheable, overridable and compressed metadata representation
* reduce the embedded metadata to a minimum

The solution proposed by SData 2.0 is designed with the above criteria in mind. The following chapters give a detailed description of this solution.

## JSON metadata in SData

Metadata in the SData JSON format surfaces at the structural and functional level:

* **Embedded metadata (structural)**: found throughout the response payloads
* **Prototype (structural)**: a resource that contains the metadata for a representation. It has a rather static nature and SHOULD support versioning (`eTag` or `modifiedDate` constructs). This is overridden and extends by embedded metadata.
* **Merge process (functional)**: combines the information from the payload with that of a prototype
* **Substitution process (functional)**: defines the process whereby a complete resource representation is obtained

The underlying characteristics are:

* Metadata expressed in JSON and relates to resource kind representations – there is no overarching prototype that describes an entire contract (like the schema does)
* The metadata of a representation SHOULD be collected in a resource called **prototype**.
    * Representations and prototypes are ideally in an 1-1 relationship
    * Prototypes contain the all the metadata of a representation (ex: structure, type, links)
* Metadata CAN be embedded in responses, but is easy to separate from the raw payload.
* Embedded metadata *extends/overrides* the metadata of the associated prototype if one such exists.  If no prototype is present, the embedded metadata is the only kind available to the consumer
* The metadata supports an underlying simple string composition process that can embed resource
specific properties in metadata values. This results in a concise formulation of metadata.
* A complete resource (i.e. containing both the data and the metadata) is obtained by a merge process between the raw and the metadata contained the prototype

## Embedded metadata

SData payloads contain metadata in two forms:
* Elements and attributes defined at the protocol level, found in the [sdata.xsd](http://interop.sage.com/daisy/sdata/g1/206-DSY.html) document
* Attributes that enhance existing elements by describing capabilities and restrictions in normed SData terms, found in the [sme.xsd](http://interop.sage.com/daisy/sdata/g1/206-DSY.html) document. These usually augment the schema definitions.

Metadata inclusion in a JSON response is governed by the following rules:
* The names of SData metadata structures are the same with those described in the [sdata.xsd](http://interop.sage.com/daisy/sdata/g1/206-DSY.html) and
[sme.xsd](http://interop.sage.com/daisy/sdata/g1/206-DSY.html) documents, but prefixed by a dollar sign '$'.
* *SData metadata for an object* is presented as properties of the objects, *along-side the native properties* of the object. Given the leading $, it is possible to distinguish among the two.
* *SData metadata* pertaining to *properties* of an object is collected in a `$properties` object within the object itself. The structure of the `$properties` is as follows:
    * properties of the `$properties` mirror the native properties of the original object (have the same name)
    * the contents are objects enclosing the metadata elements and attributes pertaining to an
individual native property

Embedded metadata should be an exception, meant to override/extend the information contained in the prototype. In this manner, it is possible to significantly reduce the volume of information transferred from the provider to the consumer.

### Example:

Consider a Product object with the following structure and data:

* Product
    * name -> "iPhone"
    * ID -> "4711"
    * unitPrice -> 459.00
    * stock -> "available" [attribute "readOnly" -> "true"]

In this example, the value of the `stock` property is computed by the underlying application and therefore `readOnly`. Retrieving this products by means of a GET operation on `.../Products('4711')` we should get the following format in JSON for the product in question:

    ... // SData pre-amble containing feed and entry information and structure
    {
        "$url" : "https://acme.com/sdata/myapp/-/-/products('4711')",
        "$key" : "4711",
        "name" : "iPhone",
        "ID" : "4711",
        "unitPrice" : 459.00,
        "stock" : "available",
        "$properties" : {
            "stock" : { "readOnlyField" : true}
        }
    }

The result shows the SData required elements `$url` and `$key` embedded at the same level as the native properties of the object. Additionally, the `$properties` contains a stock object with the name-value pair passing the readOnlyField attribute value to the caller.

### Requesting embedded metadata

Providers decide on the whether or not to support metadata and if so whether to provide it by default. A consumer can however specifically request metadata-enhanced responses through URL parameters as shown below:

    http://acme.com/sdata/myApp/-/-/products?<b>$include=$metadata</b>

## <a name="protype">SData prototype</a>

The following subsections introduce the SData prototypes by:
* describing the format and function of the $prototype object
* indicating how a `$prototype` object can surface in a response
* showing how prototypes can be retrieved

### $prototype object

A prototype bundles the metadata of a representation. If metadata is leveraged, the usage of prototypes in the JSON context is strongly recommended but **not mandatory**.

A prototype is a JSON object and is formed according to the rules laid out in the document [JSON
formatting of SData responses](https://github.com/Sage/sdata/blob/master/doc/JSON_formatted_SData_responses.md). Its first level properties are:

* `$prototypeId` property [**MUST**] : contains a value that uniquely defines a particular prototype for a resource kind (Note: there can be several prototypes for a single resource kind, see discussion under [Retrieving the prototype of a resource kind](#retrieving))
* `$prototypeTitle` property [**SHOULD**] : human-readable description of the prototype usage
* `$prototype`<sup><a href="#foot-1" name="ref-1">1</a></sup> property: contains the URL to access the prototype associated and is usually embedded in the current payload. Providers **MUST** include this property **if-and-only-if** there exists a prototype for the resources returned.
* The metadata associated with the resource kind as a whole. The name of such properties starts
with a **$** and will typically (but not exclusively) contain the elements defined in the [sdata.xsd](http://interop.sage.com/daisy/sdata/g1/206-DSY.html) and [sme.xsd](http://interop.sage.com/daisy/sdata/g1/206-DSY.html) documents of the SData 1.x specification. Examples are: `$url`, `$uuid`, `$key`, `$title`, `$isMandatory`.
* `$properties` object
* `$links` object

#### `$properties` object

The `$properties` object encapsulates the properties of the resource kind as they would be returned by a `GET` operation in JSON and has the following characteristics:

* The object structure is maintained, matching 1-1 the JSON structure in the payload
* An individual property is a JSON object whose name does NOT start with $ and contains:
    * The metadata for the object represented as name-value pairs. The name will start with a **$**
* Each object contains a metadata *name-value pair* describing its type as follows
    * The name **MUST** be `$type`
    * The value is a string describing the logical type of the object as one of:
        * `"application/x-string"` for a string property
        * `"application/x-integer"` for an integer property
        * `"application/x-date"` for a date property; format conforms to [ISO8601](http://www.w3.org/TR/NOTE-datetime)
        * `"application/x-dateTime"` for a DateTime property; format conforms to [ISO8601](http://www.w3.org/TR/NOTE-datetime)
        * `"application/x-time"` for a time property; format conforms to [ISO8601](http://www.w3.org/TR/NOTE-datetime)
        * `"application/x-boolean"` for a boolean property
        * `"application/x-currency"` for a currency type; conforms to [ISO 4217](http://www.currency-iso.org/iso_index/iso_tables/iso_tables_a1.htm)
        * `"application/x-locale"` for a locale information; conforms to the format specified in the HTTP [Accept-Language header](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html) and detailed in the [language tags](http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.10) specification of [RFC 2616 section 3](http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html).
        * `"application/x-country"` for a country information; conforms the language tags specification of [RFC 2616 section 3](http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html).
        * `"application/x-reference"` indicates that the property is a reference to another resource kind. In this case, the property MUST contain a `$url` property
        * `"application/<IANA mime type>"` for a [IANA defined mime type](http://www.iana.org/assignments/media-types/index.html)
* Non-scalar properties MUST contain the `"$isCollection" : true` name-value pair
* If the property is a property-container (`"$isCollection" : true` or `"$type" : "application/x-reference"`) then:
    * The metadata pertaining to the object itself is presented in the body of the object as $ prefixed
strings
    * A $item object is defined in the body of the object. This:
        * Replaces the $properties
        * Is an envelope for the definition of the prototype of the sub-objects
* The above definitions are applied recursively to the sub-properties.
* For a feed object, the $type property MAY be represented as a JSON array containing the various supported MIME types

##### Example:

Consider the following Address resource kind with metadata enclosed in square brackets:

* ID: [integer] [isMandatory]
* Street: [string] [title="Street"] [isMandatory=true]
* StreetNumber: [integer] [title="Number"]
* City: [string] [title="City"] [isMandatory=true]
* PostalCode: [string] [title="ZipCode"] [isMandatory=true]
* Country: [referenceToCountryResourceKind] [isMandatory=true]
    * Name: [string]
    * ISOCode: [string]
* [url="http://www.example.com/sdata/MyApp/-/-/addresses"]

This would result in the following prototype object:

    {
        "$prototypeId": "list",
        "$prototypeTitle": "List of addresses",
        "$prototype": "http://www.ex.com/sdata/MyApp/-/-/$prototypes/Address('list')",
        "$url": "http://www.ex.com/sdata/MyApp/-/-/addresses",
        "$title": "Address",
        "$properties": {
            "ID": {
                "$title": "AddressId",
                "$type": "application/x-integer",
                "$isMandatory": true
            },
            "Street": {
                "$title": "Street",
                "$type": "application/x-string",
                "$isMandatory": true
            },
            "StreetNumber": {
                "$title": "Number",
                "$type": "application/x-integer"
            },
            "City": {
                "$title": "City",
                "$type": "application/x-string",
                "$isMandatory": true
            },
            "PostalCode": {
                "$title": "ZipCode",
                "$type": "application/x-string",
                "$isMandatory": true
            },
            "Country": {
                "$title": "Country",
                "$type": "application/x-reference",
                "$prototype": "{$baseUrl}/$prototypes/countries('detail')",
                "$url": "http://www.ex.com/sdata/MyApp/-/-/countries('{ISOCode}')",
                "$isMandatory": true,
                "$item": {
                    "$properties": {
                        "Name": {
                            "$title": "Country name",
                            "$type": "application/x-string",
                            "$isMandatory": true
                        },
                        "ISOCode": {
                            "$title": "Country code",
                            "$type": "application/x-string",
                            "$isMandatory": true
                        }
                    }
                }
            }
        }
    }

In the prototype above, the scalar properties (`ID`, `Street`, ...) are objects containing respectively the `$title`, `$type` and (if appropriate) the `$isMandatory` metadata.

The `Country` property is a reference to the countries resource kind. Accordingly, it has a type of
`"application/x-reference"` and a `$url` specification. The metadata for the sub-properties of `Country` (i.e. `Name` and `ISOCode`) are defined in the body of the `$item` object.

The `{ISOCode}` formalism will be presented in the [Substitution process](#substitution) section.

#### $links object

The `$links` object contains, well... links. Links correspond by-and-large to the [xml <link>](http://www.w3.org/TR/WD-xml-link-970406.html#sec3.) element already employed in SData 1.x. The function of links is to bundle the components required for semantically complex operations, i.e. operations that go beyond CRUD; examples are `$print` or `$lookup` and particularly SData services/queries of a resource kind.

The name of a link object plays a role similar to the rel attribute of an xml link and is either:

* Standardized by IANA (see [IANA link relations](http://www.iana.org/assignments/link-relations/link-relations.xml) documentation). These start always with a **$**
* [Defined in the SData](http://interop.sage.com/daisy/sdata/AnatomyOfAnSDataFeed/FeedLevelLinks.html) protocol (including lookup). These start always with a **$**
* Services and queries: names do NOT start with $

Properties of a link object are presented in the table below:

<table>
<tr><td></td><td>Description</td><td>Compliance</td></tr>
<tr><td>$url</td>
    <td>the URL of the targeted resource(s)</td>
    <td><b>MUST</b></td></tr>
<tr><td>$title</td>
    <td>human-readable name of the link</td>
    <td>MAY</td></tr>
<tr><td>$type</td>
    <td>the return type of the request.<br/>
Default: MIME type for the request<br/>
Should several MIME types be supported, these are to be presented in the form of a JSON value array</td>
    <td>MAY</td></tr>
<tr><td>$method</td>
    <td>the HTTP verb used to access the URL.<br/>
Default: GET</td>
    <td>MAY</td></tr>
<tr><td>$invocationMode</td>
    <td>expresses the manner of invocations supported by the provider.<br/>
Allowable values are:<br/>
        <ul><li>sync [default],</li>
            <li>async,</li>
            <li>syncOrAsync.</li></ul></td>
    <td>MAY</td></tr>
<tr><td>$batchingMode</td>
    <td>expresses the manner of invocations supported in batch by the provider.
Allowable values are:
        <ul><li><b>none</b> [<b>default</b>: indicates batching is not supported],</li>
            <li>sync,</li>
            <li>async,</li>
            <li>syncOrAsync</li></ul></td>
    <td>MAY</td></tr>
<tr><td>$requestParams</td>
    <td>JSON object containing the description of request’s individual parameters.<br/>
The <b>omission</b> of $requestParams indicates that the request has <b>no parameters</b>.<br/><br/>
An individual parameter object has a unique name and contains the following properties:
        <ul><li>$title [optional]: human-readable parameter name</li>
            <li><b>$type [mandatory]</b>: defines the type of the parameter</li></ul></td>
    <td>MAY</td></tr>
<tr><td>$response</td>
Contains either:
        <ul><li>a prototype formatted JSON object to be returned</li>
            <li>the URL of the prototype of the object to be returned</li></ul></td>
    <td>MAY</td></tr>
</table>

##### Examples:

The following is an example of the links that associated with a fictional `SalesOrder` resource kind:

    {
        "$ID": "list",
        "$url": "http://www.ex.com/sdata/MyApp/-/-/SalesOrders",
        "$title": "SalesOrders",
        "$properties": {...},
        "$links": {
            "$print": {
                "$title": "Print",
                "$url": "{$url}",
                "$type": "application/pdf"
            },
            "$update": {
                "$title": "Update",
                "$url": "{$url}",
                "$method": "PUT"
            },
            "$delete": {
                "$title": "Delete order {id}?",
                "$url": "{$url}",
                "$method": "DELETE"
            },
            "createBOM": {
                "$title": "Create Bill of Materials",
                "$url": "{$url}/$service/createBOM",
                "$method": "POST",
                "$response": "{$baseUrl}/$prototypes/createBOM",
                "$batchingMode": "syncOrAsync"
            }
        }
    }

The above example shows some typical links for a resource kind: `$print`, `$update`, `$delete`.

It also contains the description of an SData service `createBOM` that returns a bill of materials for a sales order. The service specification contains the expected `$title`, `$url` and `$method` properties and indicates that it is accessible in directly in `sync` mode (through omission, the default being sync) or for batching, in `syncOrAsync` mode. The response is described by its own prototype to be found under: `{$baseUrl}/$prototypes/createBOM`.

The following is an example showing an SData query associated with a `Products` resource kind (described in the SData documentation [Example of Named Query](http://interop.sage.com/daisy/sdata/596-DSY/591-DSY.html)):

    {
        "$url": "http://www.ex.com/sdata/MyApp/myContract/-/products",
        "$title": "Products",
        "$properties": {},
        "$links": {
            "$print": {
                "$title": "Print",
                "$url": "{$url}",
                "$type": "application/pdf"
            },
            "$update": {
                "$title": "Update product",
                "$url": "{$url}",
                "$method": "PUT"
            },
            "reOrder": {
                "$title": "List of products to be reordered",
                "$url": "{$url}/$queries/reorder",
                "$method": "GET",
                "$requestParams": {
                    "_family": {
                        "$title": "product category",
                        "$type": "application/x-string"
                    },
                    "_threshold": {
                        "$title": "minimal in-stock threshold",
                        "$type": "application/x-integer"
                    }
                },
                "$response": {
                    "productID": {
                        "$title": "Product ID",
                        "$type": "application/x-string"
                    },
                    "description": {
                        "$title": "Description",
                        "$type": "application/x-string"
                    },
                    "inStock": {
                        "$title": "Quantity in stock",
                        "$type": "application/x-integer"
                    }
                }
            }
        }
    }

In the above example, the interesting part is the definition of the `reOrder` named query. This query
provides a list of products with stock below a certain threshold value. The specification of the query
contains the `$requestParams` object that contains a description of the two parameters accepted by
the query, namely `_family` and `_threshold`. It also describes the structure of the return with the
three properties: `productID`, `description` and `inStock`.

#### Prototypes surfacing in the payload

A prototype is in effect metadata describing an object. Any payload object associated with a prototype
indicates this by either:
* *Reference*: through the metadata name-value pair: `"$prototype": "<url of the
prototype>"`
* *Value*: by incorporating a metadata object named `$prototype` in the response

##### Examples

###### Specification by reference

    {
        "$url": "{$baseUrl}/addresses?creditLimitExceeded=true",
        "$title": "Addresses of accounts with exceeded credit limit",
        "$prototype": "{$baseUrl}/$prototypes/addresses(‘list’)",
        "$resources": [
            {
                "ID": "7123a",
                "Street": "Lerchenweg",
                "StreetNumber": 11,
                "PostalCode": 71711,
                "City": "Marbach am Neckar",
                "Country": {
                    "Name": "Germany",
                    "ISOCode": "GER"
                }
            }
        ]
    }

###### Specification by value

    {
        "$url": "{$baseUrl}/addresses?creditLimitExceeded=true",
        "$title": "Addresses of accounts with exceeded credit limit",
        "$resources": [
            {
                "ID": "7123a",
                "Street": "Lerchenweg",
                "StreetNumber": 11,
                "PostalCode": 71711,
                "City": "Marbach am Neckar",
                "Country": {
                    "Name": "Germany",
                    "ISOCode": "GER"
                }
            }
        ],
        "$prototype": {
            "$prototypeId": "list",
            "$prototypeTitle": "List of addresses",
            "$prototype": "http://www.ex.com/sdata/MyApp/-/-/$prototypes/addresses(‘list’)",
            "$url": "http://www.ex.com/sdata/MyApp/-/-/addresses",
            "$title": "Address",
            "$properties": {
                "ID": {
                    "$title": "AddressId",
                    "$type": "application/x-integer",
                    "$isMandatory": true
                },
            ...
            }
        }
    }

###<a name="retrieving">Retrieving the prototype of a resource kind</a>

The prototype of a resource kind is intimately related to the representation of a resource. This means that it is possible to have several prototypes, each describing individual representations of a resource.
This is easy to see when looking at the differences between a *feed* of resources and an *individual* resource: in the first case the information is succinct, while in the second it would be rather extensive.  Another example is the prototype for a representation of a resource in a mobile context (where bandwidth and screen area are prime assets) and that for a FAT client.

Prototypes are reasonably static. This means that they should be retrieved once, cached and then applied many times. Consequently, a versioning mechanism (`eTag` or `modifiedDate`) would greatly benefit the client-side handling of prototype and therefore providers SHOULD support such a mechanism.

Prototypes can be retrieved:

* Alongside a payload
* In stand-alone form by means of a dedicated URL

#### Embedding a prototype in a response: the include=$prototype specification

The most efficient manner to retrieve a representation’s prototype is to request it embedded into the response. This is done by adding the `"include=$prototype"` specification the URL of the request.

##### Examples:

* GET http://www.ex.com/sdata/MyApp/-/-/countries('GER')?<b>include=$prototype</b>
* GET http://www.ex.com/sdata/MyApp/-/-/addresses/?creditLimitExceeded=true&<b>include=$prototype</b>,$children

As shown in the last example, if an include parameter was already specified, it can be extended to contain the `$prototype` specification.

#### Retrieval from a dedicated URL

The prototypes exposed by an application are resources exposed under the dedicated `$prototypes` URL segment of an application. Prototypes are grouped per resource kind and can be accessed individually using the SData single resource formalism with the prototype ID.

##### Examples:

* `http://www.acme.com/sdata/MyApp/-/-/$prototypes` : a GET on this URL would return a list of all prototypes available
* `http://www.ex.com/sdata/MyApp/myContract/-/$prototypes/SalesOrders` : a GET on this URL will return all the prototypes available for SalesOrders
* `http://www.ex.com/sdata/MyApp/myContract/-/$prototypes/SalesOrder('detail')`: a GET on this URL will return the SalesOrder prototype with $prototypeID = ‘detail’.

### <a name="merge">Merge process</a>

For the consumer of a JSON formatted response that leverages metadata, objects are obtained in their entirety by merging the prototype with the payload information. The information in the payload has precedence, so care must be taken to avoid overwriting payload information with the prototype contents.

This ensures that several stated attributes for a JSON solution are met:

* Expressible in JSON
* Metadata is available at the resource kind level and ideally even finer-granular levels
* Ability to override metadata at any level (feed/entry/property)
* Reduce verbosity of transferred information.

The merge process is a conceptual process, meaning that a consumer will likely use a variety of local techniques to efficiently implement it while maintaining the same overall effect.

#### Example

Let us start from the example prototype provided in the $properties section of this document:

    {
        "$baseUrl": "http://www.ex.com/sdata/MyApp/-/-/",
        "$url": "{$baseUrl}/addresses",
        "$prototypeId": "list",
        "$prototype": "{$baseUrl}/$prototypes/addresses(‘{$prototypeId}’)",
        "$title": "addresses",
        "$properties": {
            "ID": {
                "$title": "AddressId",
                "$type": "application/x-integer",
                "$isMandatory": true
            },
            "Street": {
                "$title": "Street",
                "$type": "application/x-string",
                "$isMandatory": true
            },
            "StreetNumber": {
                "$title": "Number",
                "$type": "application/x-integer"
            },
            "City": {
                "$title": "City",
                "$type": "application/x-string",
                "$isMandatory": true
            },
            "PostalCode": {
                "$title": "ZipCode",
                "$type": "application/x-string",
                "$isMandatory": true
            },
            "Country": {
                "$title": "Country",
                "$type": "application/x-reference",
                "$prototype": "{$baseUrl}/$prototypes/countries(‘detail’)",
                "$url": "http://www.ex.com/sdata/MyApp/-/-/countries(‘{ISOCode}’)",
                "$isMandatory": true,
            "$item": {
                "$properties": {
                    "Name": {
                        "$title": "Country name",
                        "$type": "application/x-string",
                        "$isMandatory": true
                    },
                    "ISOCode": {
                        "$title": "Country code",
                        "$type": "application/x-string",
                        "$isMandatory": true
                    }
                }
            }
        }
    }

And combine it with the following JSON formatted payload:

     {
          "$baseUrl": "http://www.ex.com/sdata/MyApp/-/-/",
          "$url": "{$baseUrl}/addresses?creditLimitExceeded=true",
          "$title": "Addresses of accounts with exceeded credit limit",
          "$prototype": "{$baseUrl}/$prototypes/addresses(‘list’)",
          "$resources": [
               {
                    "ID": "7123a",
                    "Street": "Lerchenweg",
                    "StreetNumber": 11,
                    "PostalCode": 71711,
                    "City": "Marbach am Neckar",
                    "Country": {
                         "Name": "Germany",
                         "ISOCode": "GER"
                    },
                    "$properties": {
                         "PostalCode": {
                              "$type": "application/x-integer"
                         }
                    }
               },
               {
                    "ID": "hw7631",
                    "Street": "Fleet Street",
                    "StreetNumber": 31,
                    "City": "London",
                    "PostalCode": "EC4Y 8EQ",
                    "Country": {
                         "Name": "United Kingdom",
                         "ISOCode": "GBR"
                    }
               }
          ]
     }

The above payload provides, in addition to the payload (in **bold**) a series of specific metadata (in *italic*) for:

* The URL of the feed (`$url`)
* The title of the feed (`$title`)
* The type of the first – German – address, that must be numeric according to German rules

After the merge process, the logical JSON object will contain the following:

    {
        "$baseUrl": "http://www.ex.com/sdata/MyApp/-/-/",
        "$url": "{$baseUrl}/addresses?creditLimitExceeded=true",
        "$prototypeId": "list",
        "$prototype": "{$baseUrl}/$prototypes/addresses(‘list’)",
        "$title": "Addresses of accounts with exceeded credit limit",
        "$resources": [
            {
                "ID": "7123a",
                "Street": "Lerchenweg",
                "StreetNumber": 11,
                "PostalCode": 71711,
                "City": "Marbach am Neckar",
                "Country": {
                    "Name": "Germany",
                    "ISOCode": "GER"
                },
                $properties: {
                    "ID": {
                        "$title": "AddressId",
                        "$type": "application/x-integer",
                        "$isMandatory": true
                    },
                    "Street": {
                        "$title": "Street",
                        "$type": "application/x-string",
                        "$isMandatory": true
                    },
                    "StreetNumber": {
                        "$title": "Number",
                      $type": "application/x-integer",
                    },
                    "City": {
                        "$title": "City",
                        "$type": "application/x-string",
                        "$isMandatory": true
                    },
                    "PostalCode": {
                        "$title": "ZipCode",
                        "$type": "application/x-integer",
                        "$isMandatory": true
                    },
                    "Country": {
                        "$title": "Country",
                        "$type": "application/x-reference",
                        "$url": "http://www.ex.com/sdata/MyApp/-/-/countries(‘{ISOCode}’)",
                        "$prototype": "{$baseUrl}/$prototypes/countries(‘detail’)",
                        "$isMandatory": true,
                        "Name": {
                            "$title": "Country name",
                            "$type": "application/x-string",
                            "$isMandatory": true
                        },
                        "ISOCode": {
                            "$title": "Country code",
                            "$type": "application/x-string",
                            "$isMandatory": true
                        }
                    } // end Country
                } //end $properties of first address
            }, //end first address
            { // second address
                "ID": "hw7631",
                "Street": "Fleet Street",
                "StreetNumber": 31,
                "City": "London",
                "PostalCode": "EC4Y 8EQ",
                "Country": {
                    "Name": "United Kingdom",
                    "ISOCode": "GBR"
                }, // end country
                $properties {
                    ...
                    "PostalCode": {
                        "$title": "ZipCode",
                        "$type": "application/x-string", // !!!
                        "$isMandatory": true
                    },
                    ...
            	} //end $properties of second address
            } //end second address
        ] // end $resources
    } //end feed

Please note that the `$type` of the `PostalCode` property of the first address object is
`"application/x-integer"` as overridden by the provider.

##<a href="substitution">Substitution process</a>

The substitution step ensures that metadata is adapted to the enclosing context. The process is logically the following:

* For every *metadata* string property in the object do
    * If the string contains an *un-escaped* `"{identifierString}"` substring then
        * If the `identifierString` has a defined value within the scope then
            *Replace the `"{identifierString}"` construct with the corresponding value in string format

### Example

To continue the example described in the [Merge process](#merge) section, we apply the process described above and substitute the string values as follows:

* `$baseUrl` value into:
    * `$url` at the feed level
    * `$prototype` at the feed level
    * `$prototype` at the `Country` property level
* ISOCode value into:
    * The respective `$url` of the `Country` JSON object

The substitutions (marked with `// SUBST`) and the resulting logical JSON object are shown below:

    {
        "$baseUrl": "http://www.ex.com/sdata/MyApp/-/-/",
        "$url": "http://www.ex.com/sdata/MyApp/-/-/addresses?creditLimitExceeded=true", // SUBST
        "$prototypeId": "list",
        "$prototype": "http://www.ex.com/sdata/MyApp/-/-/$prototypes/addresses(‘list’)", // SUBST
        "$title": "Addresses of accounts with ex credit limit",
        "$resources": [
            {
                "ID": "7123a",
                "Street": "Lerchenweg",
                "StreetNumber": 11,
                "PostalCode": 71711,
                "City": "Marbach am Neckar",
                "Country": {
                    "Name": "Germany",
                    "ISOCode": "GER"
                },
                "$properties": {
                    "ID": {...},
                    "Street": {...},
                    "StreetNumber": {...},
                    "City": {...},
                    "PostalCode": {...},
                    "Country": {
                        "$title": "Country",
                        "$type": "application/x-reference",
                        "$prototype": "http://www.ex.com/sdata/MyApp/-/-/$prototypes/countries(‘detail’)", // SUBST
                        "$url": "http://www.ex.com/sdata/MyApp/-/-/countries(‘GER’)" // SUBST
                    }
                }
            },
            {
                "ID": "hw7631",
                "Street": "Fleet Street",
                "StreetNumber": 31,
                "City": "London",
                "PostalCode": "EC4Y 8EQ",
                "Country": {
                    "Name": "United Kingdom",
                    "ISOCode": "GBR"
                },
                "$properties": {
                    "ID": { ...},
                    "Street": {...},
                    "StreetNumber": {...},
                    "City": {...},
                    "PostalCode": {...},
                    "Country": {
                        "$prototype": "http://www.ex.com/sdata/MyApp/-/-/$prototypes/countries(‘detail’)", // SUBST
                        "$url": "http://www.ex.com/sdata/MyApp/-/-/countries(‘GBR’)" // SUBST
                    }
                }
            }
        ]
    }

## <a name="compliance">Compliance</a>

A provider MAY support metadata in his payloads. If this aspect is supported, the provider MAY choose
to support [prototypes](#prototype) but MUST support [embedded metadata](#embedded).

If a prototype for the targeted resource exists, the provider MUST be returned in the payload for a `GET` request with the `"?include=$prototype"` specification; otherwise, the specification has no effect.

On receipt of a `GET` request with the "`?include=$prototype"` specification, a provider SHOULD embed metadata in the response if-and-only-if metadata is supported. The amount of metadata returned is a provider specific decision. A reasonable expectation is that, if prototypes are supported, the embedded metadata would consist only of the overrides to the prototype.

A consumer MAY leverage the metadata existent in a response. If it does so then, *unless otherwise specified in the underlying contract*, it:

* MUST apply the [substitution process](#substitution)
* If a prototype exists, then this MUST be retrieved and the [merge process](#merge) MUST be applied.

---
<a href="#ref-1" name="foot-1">1</a> This may appear circular, but this is not the case. This makes a sense when applied to sub-resources or when overridden in a response or when retrieved via the `include=$prototype` parameter as will be presented in upcoming sections.
