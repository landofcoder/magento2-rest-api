# Anatomy of Magento 2: API

``Thinking Headless, make REST API``

   * [Anatomy of Magento 2: API](#anatomy-of-magento-2-api)
      * [Areas for further investigation](#areas-for-further-investigation)
      * [The Basics](#the-basics)
      * [Authentication and Authorisation](#authentication-and-authorisation)
         * [Authentication account types](#authentication-account-types)
         * [Types of authentication](#types-of-authentication)
            * [Token](#token)
            * [Session](#session)
            * [OAuth](#oauth)
            * [Anonymous](#anonymous)
         * [Granting access to resources](#granting-access-to-resources)
      * [Filters](#filters)
      * [Differences between M1 and M2 APIs](#differences-between-m1-and-m2-apis)
      * [Differences between REST and SOAP](#differences-between-rest-and-soap)
      * [SOAP-specific details](#soap-specific-details)
         * [SOAP endpoint](#soap-endpoint)
         * [Lifecycle of a SOAP request](#lifecycle-of-a-soap-request)
            * [Handling the request](#handling-the-request)
            * [Dispatching the request](#dispatching-the-request)
      * [REST-specific details](#rest-specific-details)
         * [REST endpoint](#rest-endpoint)
         * [Lifecycle of a REST request](#lifecycle-of-a-rest-request)
      * [Error handling](#error-handling)
         * [REST](#rest)
         * [SOAP](#soap)
      * [Adding a new API resource](#adding-a-new-api-resource)
         * [Defining the resource methods](#defining-the-resource-methods)
         * [Create the controller and methods](#create-the-controller-and-methods)
      * [Example API calls using cURL](#example-api-calls-using-curl)
      * [Sources](#sources)
      

## Areas for further investigation
* Check videos (FM2D, Nomad Mage) for API related content
* Verify if browser-based session is only available for customer account types.
* Are integration users only used by the oauth system?
* Create demo extensions which use each authentication type.
* Verify the API versioning strategy.
    * What's the thinking behind it?
    * How does it affect code execution?
    * How to use it in custom code? Should you even do that?
* How to test API?
* Create Magento 2 API cheatsheet

Notes were taken against Magento 2.1.5

## The Basics

There are two APIs built into Magento 2: `REST` and `SOAP`. 

All Magento 2 APIs are versioned, with a major version number in each API resource method name, e.g. `catalogProductRepositoryV1`.

This means that any breaking changes will necessitate a new API resource method with a new name. Partly this was done to allow two versions of the API to co-exist when backwards incompatible changes are introduced. The version numbering schema of API methods deliberately does not follow the SemVer guidelines ([1]).

Resource method names need to follow the regex `[a-zA-Z\d]*V[\d]+`.
 
When a request is made to an API endpoint, the matching resource method name in the `webapi` config XML is called and the results returned to the API consumer.

## Authentication and Authorisation

### Authentication account types

There are four account types (in order of descending permissions): `Admin`, `Integration`, `Customer` and `Guest`.

`Admin` users can access anything.

`Integration` users are only used by the OAuth authentication system. They are intended to be used in situations where a module needs API access to an installation, but admin access has not been granted. Their access to resources is limited to their custom ACL role, `self` or `anonymous`. Specific ACL roles need to be created for integration users.

`Customer` users can only access resources with a type of `self` or `anonymous`, i.e. Only the data for that specific customer.

`Guest` users can only access resources with a type of `anonymous`. If Magento cannot authenticate an API consumer, they default to the `Guest` type.

### Types of authentication

#### Token

1. Submit a request to the appropriate `REST` or `SOAP` endpoint, specifying the username and password of an `Admin` or `Customer`.
1. Receive a token.
1. Specify this token in future requests.

Tokens never expire, but can be revoked.

The Magento 2 Dev Docs are unusually detailed about making getting a token, making future requests and explaining the different parts of a token request URI:

[Magento 2 Dev Docs: Token-based authentication](http://devdocs.magento.com/guides/v2.1/get-started/authentication/gs-authentication-token.html)

#### Session

1. Login to the website as an `Admin` or `Customer`.
1. Submit a request.
1. Receive the results. 

This type of authentication is useful for when you need to query the API for the current customer's data using a JavaScript widget, for example. 

For `Customer`s, the resources you can access with this authentication type are limited to resources of the `self` and `anonymous` type.

For `Admin`s, the resources you can access are limited to those defined in the ACL role you are assigned to.

The Magento 2 Dev Docs provide a little bit more detail:

[Magento 2 Dev Docs: Session-based authentication](http://devdocs.magento.com/guides/v2.1/get-started/authentication/gs-authentication-session.html)


#### OAuth

1. Create a new integration in the Magento admin.
1. Activate the integration in Magento.
1. Call the third party application's login page.
1. Third party application asks for a request token.
1. Magento sends the request token.
1. The third party application asks for an access token.
1. Magento sends an access token.
1. The third party application can now access Magento resources.

In Magento, a third-party extension that uses OAuth for authentication is called an integration. An integration defines which resources the extension can access. The extension can be granted access to all resources or a customized subset of resources.

As the process of registering the integration proceeds, Magento creates the tokens that the extension needs for authentication. It first creates a request token. This token is short-lived and must be exchanged for access token. Access tokens are long-lived and will not expire unless the merchant revokes access to the extension.

The Magento 2 Dev Docs are unusually detailed in this respect: 

[Magento 2 Dev Docs: OAuth-based authentication](http://devdocs.magento.com/guides/v2.1/get-started/authentication/gs-authentication-token.html)

Source: [Magento Quickies: Magento 2: Understanding Integration API Users](http://magento-quickies.alanstorm.com/post/142486885550/magento-2-understanding-integration-api-users)

#### Anonymous

1. Submit a request to the appropriate `REST` or `SOAP` endpoint.
1. Receive response
1. Profit

Tokens can be passed for anonymous requests, but they are, for obvious reasons, totally optional.

### Granting access to resources

There are three types of M2 API resource: `<ACL Rule Identifer>`, `self`, `anonymous`. 

Each of these are defined in the `ref` attribute of the `<resource/>` node.

`ACL Rule Identifier` refers to an existing Magento 2 ACL permission, e.g:
```xml
// File: vendor/magento/module-quote/etc/webapi.xml
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Webapi:etc/webapi.xsd">
    <!-- Managing Cart -->
    <route url="/V1/carts/:cartId" method="GET">
        <service class="Magento\Quote\Api\CartRepositoryInterface" method="get"/>
        <resources>
            <resource ref="Magento_Cart::manage" />
        </resources>
    </route>
</routes>
```

If you authenticate as a customer, you can only access the following two:

`self` is used with the Customer authentication account type. It means only resources with the `self` or `anonymous` type are accessible.
```xml
// File: vendor/magento/module-quote/etc/webapi.xml
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Webapi:etc/webapi.xsd">
    <route url="/V1/carts/mine" method="GET">
        <service class="Magento\Quote\Api\CartManagementInterface" method="getCartForCustomer"/>
        <resources>
            <resource ref="self" />
        </resources>
        <data>
            <parameter name="customerId" force="true">%customer_id%</parameter>
        </data>
    </route>
</routes>
```

`anonymous` is used with the Customer authentication account type. It means only resources with the `anonymous` type are accessible.
```xml
// File: vendor/magento/module-quote/etc/webapi.xml
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Webapi:etc/webapi.xsd">
    <!-- Managing guest carts -->
    <route url="/V1/guest-carts/:cartId" method="GET">
        <service class="Magento\Quote\Api\GuestCartRepositoryInterface" method="get"/>
        <resources>
            <resource ref="anonymous" />
        </resources>
    </route>
</routes>
```

## Filters

Filters allow a result set to be filtered to only return records which match specific criteria. 

Filters come in two flavours: Required and Optional. They are specified in the same way.

For example, to retrieve a product record for a specific SKU:

```bash
// REST
https://<MAGENTO_HOST_OR_IP>/<MAGENTO_BASE_INSTALL_DIR>/rest/V1/products/:sku

// SOAP (The :sku is passed in the payload as normal with SOAP requests)
http://<magento_host>/soap/default?wsdl&services=catalogProductRepositoryV1
```

Where `:sku` is the product SKU.

## Differences between M1 and M2 APIs

There is no longer an `XML-RPC` API in Magento 2 as there was in Magento 1.

The `SOAP` API no longer has a single WSDL. Instead, individual API objects/services have their own WSDL.

The entire list can be retrieved here: `http://<magento_host>/soap/default?wsdl_list=1`

[Magento 2 Dev Docs: List of service names per module](http://devdocs.magento.com/guides/v2.0/soap/bk-soap.html)

Unlike Magento 1, there are no API specific ACLs. The API shares the same ACL rules as a regular admin user.

## Differences between REST and SOAP

The SOAP and REST based APIs are, from a business logic point of view, equivalent.

Each Magento 2 API request runs through a standard Magento controller. Both `REST` and `SOAP` APIs have their own
controller for handling requests:

```php
<?php
namespace Magento\Webapi\Controller;

/**
 * Front controller for WebAPI REST area.
 */
class Rest implements \Magento\Framework\App\FrontControllerInterface
{
    // ...
}
?>
<?php

namespace Magento\Webapi\Controller;

/**
 * Front controller for WebAPI SOAP area.
 */
class Soap implements \Magento\Framework\App\FrontControllerInterface
{
	// ...
}
?>
```

## `SOAP`-specific details

### `SOAP` endpoint

To query the `SOAP` API, dispatch a request to an endpoint in the following format:

`http://<magento_host>/soap/<store_code>?wsdl&services=<serviceName1,serviceName2,..>`

 `store_code` can be one of:
 * `default`: The default store code.
 * `<store_code>`: A store code which exists in this installation.
 * `all`. This is a special value which only applies to the CMS and Product modules. If this value is specified, the API call affects all the merchant's stores. 

`serviceNameN` is the name of the resource method you want to query. E.g. If you want to get a list of Products, you would specify the `catalogProductRepositoryV1` resource method name.  

Multiple resource method names can be specified in one `SOAP` request by separating them using the `,` separator:
`http://<magento_host>/soap/<store_code>?wsdl&services=<serviceName1,serviceName2,..>`

The entire list can be retrieved for a specific installation here: `http://<magento_host>/soap/default?wsdl_list=1`

Alternatively, you can find a list of all the resource methods in a vanilla installation in the source link below.

Source: [Magento 2 Dev Docs: List of service names per module](http://devdocs.magento.com/guides/v2.0/soap/bk-soap.html)

### Lifecycle of a `SOAP` request

#### Handling the request
* API consumer hits the appropriate endpoint (e.g. `http://<magento_host>/soap/default?wsdl&services=catalogProductRepositoryV1`)
* `Magento\Webapi\Controller\Soap::dispatch` receives request, then sets the application area, checks for and validates the `WSDL`, if present and hands off handling of the request to `Magento\Webapi\Controller\Soap::handle()`.
* Magento validates the request body
    * If the request has an empty body or has invalid XML, Magento throws an `\Magento\Framework\Webapi\Exception` with message `Invalid XML`.
    * If the request has an XML document type node, Magento throws an `\Magento\Framework\Webapi\Exception` with message `Invalid XML: Detected use of illegal DOCTYPE`.
    * In either case, Magento terminates the request and responds with an `HTTP 500` error code.
* Magento generates a local `WSDL` URI (this is done to be able to pass the `WSDL` schema to a new PHP `SoapServer` object without needing authorisation) and passes it to the new PHP `SoapServer` object, along with some default options (encoding, `SOAP` version).
* When Magento creates a new `SoapServer` object using the `Magento\Webapi\Model\Soap\ServerFactory`, automatic DI injects a `\Magento\Webapi\Controller\Soap\Request\Handler` object and sets it against the PHP `SoapServer` object.
* Magento calls the `SoapServer::handle()` method, which delegates to the `\Magento\Webapi\Controller\Soap\Request\Handler::__call()` magic method to determine which controller and method should be dispatched to handle the request.

#### Dispatching the request

* Magento checks if the request needs to be made over HTTPS and if it was.
* Magento checks the request against the appropriate ACL permissions.
* Magento then tries to match the requested resource method with a resource method defined in one of the `webapi.xml` files, which are merged and cached on the first API request.
* If a cached version of the merged `webapi.xml` tree doesn't exist, a `Magento\Webapi\Model\ServiceMetadata` object is created which parses it, converts it to an array, serialises it and stores it in the cache.
    * Since the configuration files are biased towards a `REST`Ful version of the universe, the `ServiceMetadata` object is used to transform the configuration into information the `SOAP` handler object can use to match up a `SOAP` request with the PHP class and method.
    * Part of the parsing of the tree includes using PHP's Reflection API to detect all the methods in each resource class and adding that to the array which is eventually cached.
* The passed request method is then cross-referenced against the array, which is used to find the appropriate class and method used to fulfil the request.
* Magento then uses the Object Manager to instantiate the appropriate object for the request and then calls the resource method against the instantiated object and returns the response.

Source: [Magento Quickies: Magento 2: Understanding the Web API Architecture](http://magento-quickies.alanstorm.com/post/142036931270/magento-2-understanding-the-web-api-architecture)

## `REST`-specific details

### `REST` endpoint

### Lifecycle of a `REST` request

This workflow assumes that the API consumer has already obtained the necessary token to make an API request.

* API consumer hits the appropriate endpoint (e.g. `http://<magento_host>/rest/V1/products/`)
* `Magento\Webapi\Controller\Rest::dispatch()` receives request, sets the application area and then passes off to another method to process the request.
* The Service Class and Service Method are detected from the endpoint.
* The Service Method is called and the output is returned.
* The data is then converted into a scalar value or array of scalar values using the `Magento\Framework\Webapi\ServiceOutputProcessor` class.
* If any filter parameters were specified, they are applied to the data.
* The data is assigned to the response object and returned to the API consumer.

## Error handling

For both `REST` and `SOAP`, the logic calling the Service Method is wrapped in a `try ... catch` block, with exceptions being caught by PHP's built-in `Exception` class. 

Exceptions are then processed by `Magento\Framework\Webapi\ErrorProcessor::maskException()` whose function is to sterilise exception messages if not running in `developer` mode. 

### `REST`

Exceptions are caught and then assigned to the response object.

### `SOAP`

Exceptions are caught, then a new `\Magento\Webapi\Model\Soap\Fault` object is created and returned in the response.

## Adding a new `REST` API resource

* Define the resource in the `webapi.xml` file
* Add your API methods to an `Interface`
* Add a `Model` which implements your `Interface`
* Add a `preference` to your `di.xml`.

### Defining the resource methods

All resource methods are defined in a `app/code/[Vendor]/[Module]/etc/webapi.xml` file:
```xml
// File: app/code/ProjectEight/AddNewApiMethod/etc/webapi.xml
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Webapi:etc/webapi.xsd">
    <!-- Example: curl http://127.0.0.1/index.php/rest/V1/calculator/add/1/2 -->
    <route url="/V1/calculator/add/:numberOne/:numberTwo" method="GET">
        <!-- The 'add' method of the class which implements this interface will be called when this endpoint is hit -->
        <service class="ProjectEight\AddNewApiMethod\Api\CalculatorInterface" method="add"/>
        <resources>
            <!-- Anyone can access this resource -->
            <resource ref="anonymous"/>
        </resources>
    </route>
</routes>

// File: vendor/magento/module-quote/etc/webapi.xml
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Webapi:etc/webapi.xsd">
    <!-- Managing Cart -->
    <route url="/V1/carts/:cartId" method="GET">
        <service class="Magento\Quote\Api\CartRepositoryInterface" method="get"/>
        <resources>
            <resource ref="Magento_Cart::manage" />
        </resources>
    </route>
    <!-- Other routes follow ... -->
</routes>
```

Where:
* `<route>`: Defines an endpoint which can be queried by an API. 
    * `url="[endpoint]"`: Defines the URL, in `REST` syntax, even if this resource is to be used by a `SOAP` client.
    * `method="[HTTP Method]"`: Defines the HTTP method used to query this endpoint, e.g. `GET`, `POST`, `PUT`, `DELETE`.
* `<service/>`: Defines the interface (`class=""`) and PHP method name (`method="get"`) which will be called. Note that `method=""` does not refer to the HTTP method, not does it have to match the HTTP method!
* `<resources>`: Defines one or more `<resource>` nodes, which determine which permission types can access this resource.
* `<resource/>`: Defines the permission type which can access this resource (A Magento 2 ACL role identifier, `self` or `anonymous`. The latter two are explained further below).

These are the basic nodes, there are others available as well. Check the `vendor/magento/module-webapi/etc/webapi.xsd` for the definition of all the possible nodes and attributes.

### Add your API methods to an `Interface`

The type hints in the doc blocks are important, because Magento 2 uses them to infer the types of passed arguments.
 
```php
<?php
namespace ProjectEight\AddNewApiMethod\Api;
use ProjectEight\AddNewApiMethod\Api\Data\PointInterface;

interface CalculatorInterface
{
    /**
     * Add two numbers together
     *
     * @param int $numberOne
     * @param int $numberTwo
     *
     * @return int
     */
    public function add($numberOne, $numberTwo);

    /**
     * Sum an array of numbers.
     *
     * @param float[] $numbers The array of numbers to sum.
     *
     * @return float The sum of the numbers.
     */
    public function sum($numbers);

    /**
     * Compute mid-point between two points.
     * 
     *
     * @param ProjectEight\AddNewApiMethod\Api\Data\PointInterface $pointOne The first point.
     * @param ProjectEight\AddNewApiMethod\Api\Data\PointInterface $pointTwo The second point.
     *
     * @return ProjectEight\AddNewApiMethod\Api\Data\PointInterface The mid-point.
     */
    public function midPoint($pointOne, $pointTwo);
}
```
### Add a `Model` which implements your `Interface`

```php
<?php
namespace ProjectEight\AddNewApiMethod\Model;
use ProjectEight\AddNewApiMethod\Api\CalculatorInterface;
use ProjectEight\AddNewApiMethod\Api\Data\PointInterface;
use ProjectEight\AddNewApiMethod\Api\Data\PointInterfaceFactory;

class Calculator implements CalculatorInterface
{
    /**
     * Factory for creating new Point instances. This code will be automatically
     * generated because the type ends in "Factory".
     *
     * @var PointInterfaceFactory
     */
    private $pointFactory;

    /**
     * Constructor.
     *
     * @param PointInterfaceFactory $pointFactory Factory for creating new Point instances.
     */
    public function __construct(PointInterfaceFactory $pointFactory)
    {
        $this->pointFactory = $pointFactory;
    }

    /**
     * Add two numbers together
     *
     * @param int $numberOne
     * @param int $numberTwo
     *
     * @return int
     */
    public function add($numberOne, $numberTwo)
    {
        $sum = $numberOne + $numberTwo;

        return $sum;
    }

    /**
     * Sum an array of numbers.
     *
     * @param float[] $numbers The array of numbers to sum.
     *
     * @return float The sum of the numbers.
     */
    public function sum($numbers)
    {
        $total = 0.0;
        foreach ($numbers as $number) {
            $total += $number;
        }

        return $total;
    }

    /**
     * Compute mid-point between two points.
     *
     * @param PointInterface $pointOne The first point.
     * @param PointInterface $pointTwo The second point.
     *
     * @return PointInterface The mid-point.
     */
    public function midPoint($pointOne, $pointTwo)
    {
        $point = $this->pointFactory->create();
        $point->setX(($pointOne->getX() + $pointTwo->getX()) / 2.0);
        $point->setY(($pointOne->getY() + $pointTwo->getY()) / 2.0);

        return $point;
    }
}
```

### Add a `preference` to your `di.xml`.

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <preference for="ProjectEight\AddNewApiMethod\Api\CalculatorInterface"
                type="ProjectEight\AddNewApiMethod\Model\Calculator"/>
    <preference for="ProjectEight\AddNewApiMethod\Api\Data\PointInterface"
                type="ProjectEight\AddNewApiMethod\Model\Point" />
</config>
```

You can now call your new method using `REST`:
```bash
# Add two numbers
$ curl http://magento2-sample-modules.localhost.com/index.php/rest/V1/calculator/add/1/2
3
# Add an array of floating point numbers
$ curl -d '{"numbers":[1.1,2.2,3.3]}' -H 'Content-Type: application/json' http://magento2-sample-modules.localhost.com/index.php/rest/V1/calculator/sum
6.6
# Compute the mid-point between two points
$ curl -d '{"pointOne":{"x":10,"y":10},"pointTwo":{"x":30,"y":50}}' -H 'Content-Type: application/json' http://magento2-sample-modules.localhost.com/index.php/rest/V1/calculator/midpoint
{"x":20,"y":30}
```
Source [[2]]

## Example API calls using cURL

```bash
// Create an admin token for future requests
$ curl --request POST http://magento2-sample-modules.localhost.com/index.php/rest/default/V1/integration/admin/token -H "Content-Type:application/json" -d '{"username":"admin","password":"password123"}'
ewkm55iwv9kl0g9ul194lxlgwo6jp93p
// Create a new customer
$ curl --request POST http://magento2-sample-modules.localhost.com/index.php/rest/default/V1/customers -H "Content-Type:application/json" -d \
'{
	"customer": {
		"email": "simonfrost2010@gmail.com",
		"firstname": "Simon",
		"lastname": "Frost",
		"addresses": [{
			"defaultShipping": true,
			"defaultBilling": true,
			"firstname": "Simon",
			"lastname": "Frost",
			"region": {
				"regionCode": "NY",
				"region": "New York",
        "regionId":43
			},
			"postcode": "10755",
			"street": ["123 Oak Ave"],
			"city": "Purchase",
			"telephone": "512-555-1111",
			"countryId": "US"
		}]
	},
  "password": "password123"
}'
// Create a customer token for future requests
$ curl --request POST http://magento2-sample-modules.localhost.com/index.php/rest/default/V1/integration/customer/token -H "Content-Type:application/json" -d '{"username":"simonfrost2010@gmail.com","password":"password123"}'
// Create an empty basket for this customer
$ curl --request POST http://magento2-sample-modules.localhost.com/index.php/rest/default/V1/carts/mine -H "Content-Type:application/json" -H "Authorization: Bearer 11mfta00er02w5uq9q50d1xxp9gnx85k"
```

## Sources

[REST API for latest Magento version](http://devdocs.magento.com/swagger/index.html)

[1]: https://joind.in/event/nomad-mage-december-2016/schedule/list
[2]: https://alankent.me/2015/07/24/creating-a-new-rest-web-service-in-magento-2/