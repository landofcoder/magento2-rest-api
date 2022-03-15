## How to create REST API Endpoints

    ``Thinking Headless, make REST API``

We should create magento 2 extension to make REST API

### Define REST API Endpoints

Before create API Endpoints, should think there data which module will delivery to client, security for data, permission for data

Define API endpoint in file ``/etc/webapi.xml``

API Endpoint name should have name as this: ``V1/[end-point-name]/[action]/[params]``
end-point-name we can use module vendor prefix. It will use for all function relate this endpoint.
action we can set detail of endpoint function. Example : ``resetPassword``

Example: Customer Account Management API Endpoints

``/V1/customers`` - POST
``/V1/customers/{customerId}/password/resetLinkToken/{resetPasswordLinkToken}`` - GET

{customerId}, {resetPasswordLinkToken} are params

``/V1/customers/password`` - PUT

``/V1/customers/resetPassword`` - POST

``/V1/customers/isEmailAvailable`` - POST

at here we have "customers" is end-point-name

### Security for data

Magento 2 support Authentication for REST API. There are four account types (in order of descending permissions): `Admin`, `Integration`, `Customer` and `Guest`.

`Admin` users can access anything.

`Integration` users are only used by the OAuth authentication system. They are intended to be used in situations where a module needs API access to an installation, but admin access has not been granted. Their access to resources is limited to their custom ACL role, `self` or `anonymous`. Specific ACL roles need to be created for integration users.

`Customer` users can only access resources with a type of `self` or `anonymous`, i.e. Only the data for that specific customer.

`Guest` users can only access resources with a type of `anonymous`. If Magento cannot authenticate an API consumer, they default to the `Guest` type.


Because REST API can return more data from database. So, we should think some sensitive data should not response or need permission to access them.

#### Example:

get current logged in customer data, REST API require access with permission resource = self