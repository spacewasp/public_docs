# How I Was Paid $9,000 for a Critical Vulnerability in Adobe Commerce (CVE-2024-34102)

From time to time, I participate in bug bounty programs. When I choose a target, I base my decision on its popularity. I found a great target called **Magento**. Later, I discovered that it belonged to **Adobe**. As soon as the bug was fixed, I was surprised by the attention it received and came across this [article](https://sansec.io/research/cosmicsting). This vulnerability, `CVE-2024-2961`, has been named `CosmicString`.

## Beginning

The latest version of a product should be downloaded from https://github.com/magento/magento2. At that time, the latest version was `2.4.6-p2`. There is also an alternative way to install it from https://marketplace.digitalocean.com/apps/magento-2-open-source. However, this is still a **vulnerable** version at the time of writing!

## Entry point

**Magento** is an HTTP PHP server application. In most cases, such applications have two global entry points: **User Interface** and **API**. I started with the **REST API** because it is faster to start using `Python`, there is no `CSRF protection`, and so on. **Magento** uses **REST API**, **GraphQL**, and **SOAP**. Let's focus on the **REST API**. Summary notes are [here](https://developer.adobe.com/commerce/webapi/rest/use-rest/notes/).


For example, **REST API** request looks like

```
POST /rest/default/V1/carts/mine/estimate-shipping-methods HTTP/1.1
Host: foo.example
Content-Type : application/json
Content-Length: 1402

{
    "address": {
        "region": "incididunt",
        "region_id": -67220712,
        "region_code": "nisi laboris aute in Duis",
        "country_id": "consectetur fugiat ",
        "street": [
            "in non fugiat consequat",
            "fugiat aliqua non commodo"
        ],
        "telephone": "dolor enim nisi culpa",
        "postcode": "ex Excepteur reprehenderit",
        "city": "laborum cupidatat est ut",
        "firstname": "voluptate minim ",
        "lastname": "do exercitation",
        "email": "ut dolore occaecat sunt",
        "company": "eu officia in",
        "custom_attributes": [
            {
                "attribute_code": "anim nulla cupidatat Lorem aute",
                "value": "voluptate Lore"
            },
            {
                "attribute_code": "dolore",
                "value": "tempor proident nostrud"
            }
        ],
        "customer_address_id": 22984299,
        "customer_id": -43661864,
        "extension_attributes": {
            "gift_registry_id": 17076753
        },
        "fax": "adipisicing id",
        "id": 3672246,
        "middlename": "et tempor enim",
        "prefix": "sunt dolor esse eiusmod",
        "same_as_billing": -57436048,
        "save_in_address_book": -15023339,
        "suffix": "anim ipsum proident",
        "vat_id": "est elit Duis "
    }
}
```

## Implementation

The first challenge is to discover all possible URLs for **REST API**. There is the configuration file `webapi.xml`, it connects URLs with PHP request handlers.
Here is part of it:

```xml
    <route url="/V1/carts/:cartId/estimate-shipping-methods" method="POST">
        <service class="Magento\Quote\Api\ShipmentEstimationInterface" method="estimateByExtendedAddress"/>
        <resources>
            <resource ref="Magento_Cart::manage" />
        </resources>
    </route>

    <route url="/V1/guest-carts/:cartId/collect-totals" method="PUT">
        <service class="Magento\Quote\Api\GuestCartTotalManagementInterface" method="collectTotals"/>
        <resources>
            <resource ref="anonymous"/>
        </resources>
    </route>
```

*Important note*: some of **REST API**'s can be used without authentication, like `/V1/guest-carts/:cartId/collect-totals`, because
```xml
    <resources>
        <resource ref="anonymous"/>
    </resources>
```

I did some research and found that each interface class has a hardcoded implementation in the file `di.xml`.
Here is part of it:
```xml
    <preference for="Magento\Quote\Api\ShipmentEstimationInterface" type="Magento\Quote\Model\ShippingMethodManagement" />
```

As soon as we send an HTTP request, the method `estimateByExtendedAddress` is called from the class `ShippingMethodManagement`.

let's take a look at this method:

```php
    public function estimateByExtendedAddress($cartId, AddressInterface $address)
    {
        /** @var Quote $quote */
        $quote = $this->quoteRepository->getActive($cartId);

        // no methods applicable for empty carts or carts with virtual products
        if ($quote->isVirtual() || 0 == $quote->getItemsCount()) {
            return [];
        }
        return $this->getShippingMethods($quote, $address);
    }
```

There is one question: How is plain text from the HTTP body converted to an `AddressInterface` object? What does the deserialization process look like? Also, if the **REST API** is public, we can try to deserialize data without any authentication.

Let's dive deep.

## Deserialization

The logic of the whole deserialization process is tricky. It is little bit simplified, but in general these steps are true. Let's examine an example involving `AddressInterface $address`

1. find implementation of `AddressInterface` (in file `di.xml`). `Magento\Customer\Api\Data\AddressInterface` => `Magento\Customer\Model\Data\Address`

2. find constructor of class `Address`
```php
    public function __construct(
        \Magento\Framework\Api\ExtensionAttributesFactory $extensionFactory,
        AttributeValueFactory $attributeValueFactory,
        \Magento\Customer\Api\AddressMetadataInterface $metadataService,
        $data = []
    ) {
        ...
    }
```
3. Check if the `json` HTTP body contains `extensionFactory`, `attributeValueFactory`, `metadataService`, or `data` field. For example, if `extensionFactory` is present, recursively repeat the process from step 1. Otherwise, retrieve the default generic value for the class `ExtensionAttributesFactory`.

4. Check if there is value http body `set` + value is one of methods of class `Address`. For example `aaaa`, then method is `setaaaa`. if so, call it and resolve only parameter (process from step 1)

The issue is the deserialization is too flexible. It uses user data and system data without separation

## Recovering all inputs parameters

To discover all possible input parameters, I patched the program code:

1. The deserializer checks if a parameter (for example,  `extensionFactory`) is in the JSON body. Always it retrieves `yes` and stores it in the dictionary.

2. The same applies to the `set` methods.

3. The dictionary is written to disk as a new JSON file.

here is example of output:

```json
{
    "method": "POST",
    "uri": "\/rest\/V1\/guest-carts\/:cartId\/gift-message",
    "data": {
        "cartId": "aaaaaaaaaaaaaaaaaaaaaa",
        "giftMessage": {
            "context": {
                "eventDispatcher": {
                    "instanceName": "aaaaaaaaaaaaaaaaaaaaaa",
                    "shared": true
                },
                "appState": {
                    "configScope": {
                        "areaList": {
                            "default": "aaaaaaaaaaaaaaaaaaaaaa"
                        },
                        "defaultScope": "aaaaaaaaaaaaaaaaaaaaaa",
                        "CurrentScope": "aaaaaaaaaaaaaaaaaaaaaa"
                    },
                    "mode": "aaaaaaaaaaaaaaaaaaaaaa",
                    "AreaCode": "aaaaaaaaaaaaaaaaaaaaaa"
                }
            },
            "resource": {
                "context": {
                    "resource": {
                        "resourceConfig": {
                            "instanceName": "aaaaaaaaaaaaaaaaaaaaaa",
                            "shared": true
                        },
                        "tablePrefix": "aaaaaaaaaaaaaaaaaaaaaa"
                    }
                },
                "connectionName": "aaaaaaaaaaaaaaaaaaaaaa"
            },
            "GiftMessageId": 42,
            "CustomerId": 42,
            "Sender": "aaaaaaaaaaaaaaaaaaaaaa",
            "Recipient": "aaaaaaaaaaaaaaaaaaaaaa",
            "Message": "aaaaaaaaaaaaaaaaaaaaaa",
            "ExtensionAttributes": {
                "EntityId": "aaaaaaaaaaaaaaaaaaaaaa",
                "EntityType": "aaaaaaaaaaaaaaaaaaaaaa"
            }
        }
    }
}
```

## Use gathered parameters

The data has not been filtered at all. For example, if we send an HTTP request with this body, we will get interesting errors.

like 

```
"Class \"aaaaaaaaaaaaaaaaaaaaaa\" does not exist",
```

or from another request

```
"Warning: SessionHandler::read(): open(aaaaaaaaaaaaaaaaaaaaaa\/sess_aaaaaaaaaaaaaaaaaaaaaa, O_RDWR) failed: No such file or directory (2) in ...
```

## Vulnerability

In one request, it was possible to deserialize `SimpleXMLElement`. Let's take a look at the parameters:

```php
public __construct(
    string $data,
    int $options = 0,
    bool $dataIsURL = false,
    string $namespaceOrPrefix = "",
    bool $isPrefix = false
)
```

Thus, an attacker could pass parameters in `data` similar to a classic `XXE attack`, and we can control `options` to set `LIBXML_NOENT|LIBXML_PARSEHUGE`, which is disabled by default. Pay attention to `dataIsURL`, because in the [article](https://sansec.io/research/cosmicsting) there was a flawed patch that allowed an attacker to bypass it.

Using this blind `XXE attack` with reverse connection, an attacker can download `env.php`
```xml
<!ENTITY file SYSTEM "../app/etc/env.php">
```

using this key from `env.php`

```
    'crypt' => [
        'key' => '4ba8fa7a14627e3b812858091ed163ec'
    ],
```

It is possible to get **admin** access to **REST API**, **GraphQL** or **Soap**

## Time line

1. submitted to HackerOne `December 20, 2023`

2. submitted to Adobe from Hacker one `January 8, 2024`

3. bounty was paid `May 21, 2024`

4. `CVE-2024-34102` published `June 11, 2024`

## Hot fix for hot fix from SanSec

```php
if (strpos(file_get_contents('php://input'), 'sourceData') !== false) {
    header('HTTP/1.1 503 Service Temporarily Unavailable');
    header('Status: 503 Service Temporarily Unavailable');
    exit;
}
```
