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
