[![Build Status](https://travis-ci.org/sdawood/dynamo-update-expression.png?branch=master)](https://travis-ci.org/sdawood/dynamo-update-expression)

# dynamo-update-expression
=============

Generate DynamoDB Update Expression by diff-ing original and updated documents.

Allows for generating update expression with no-orphan (create new nodes as you go) or deep paths with no-orphans (ideal for *predefined* document structure)

Optionally include a condition expression with you update to utilize [Optimistic Locking With Version Number](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBMapper.OptimisticLocking.html)


```js
const due = require('./dynamo-update-expression');

due.getUpdateExpression({original, modified, ...options});

due.getVersionedUpdateExpression({original, modified, versionPath = '$.path.to.version', condition = '=');

due.getVersionLockExpression({newVersion: expiryTimeStamp, condition: '<'});

// Bonus!

const {ADD, DELETE, SET} = diff(original, modified /* orphans = false*/);
```

See the options available below:

## Installation

  ```sh
  npm install dynamo-update-expression --save
  ```

## Usage

  ```js
  const due = require('dynamo-update-expression');

  const original = {...};
  const modified = {...};

  const updateExpression = due.getUpdateExpression({original, modified});

  // Use Case 1: Straight forward diff between original and modified (added, modified, removed) attributes are discovered

  // Use Case 2: To conditionally update only if the current version in DynamoDB has not changed since original was loaded

  const versionedUpdateExpression = due.getVersionedUpdateExpression({original, modified, condition = '='});

  // Conditional updates (Optimitic Version Locking)

  // To conditionally update if the new value is greater than the value in DynamoDB
  const versionedUpdateExpression = due.getVersionedUpdateExpression({original, modified, useCurrent: false, condition = '<'});

  // Use Case 3: TRY-LOCK behaviour

  // To validate that the range you are about to process hasn't been processed by a different worker
  const rangeStart = 1000;
  const updateExpression = due.getVersionLockExpression({
      versionPath: '$.path.to.rangeAttribute'
      newVersion: rangeStart,
      condition: '<'
  });

  // To `TRY-LOCK` the next 5 min, where other clients can't obtain the lock (using a similar expression), without loading `current` record
  const expiry = +new Date() + (5 * 1000 * 60)
  const lockUpdateExpression = due.getVersionLockExpression({newVersion: expiry, condition: '<'});

  ```

  Where original and modified are JSON compatible objects.

  For example:

  Original JSON:

  ```js
  const original = {
       id: 123,
       title: 'Bicycle 123',
       description: '123 description',
       bicycleType: 'Hybrid',
       brand: 'Brand-Company C',
       price: 500,
       color: ['Red', 'Black'],
       productCategory: 'Bicycle',
       inStok: true,
       quantityOnHand: null,
       relatedItems: [341, 472, 649],
       pictures: {
           frontView: 'http://example.com/products/123_front.jpg',
           rearView: 'http://example.com/products/123_rear.jpg',
           sideView: 'http://example.com/products/123_left_side.jpg'
       },
       productReview: {
           fiveStar: [
               "Excellent! Can't recommend it highly enough! Buy it!",
               'Do yourself a favor and buy this.'
           ],
           oneStar: [
               'Terrible product! Do no buy this.'
           ]
       },
       comment: 'This product sells out quicly during the summer',
       'Safety.Warning': 'Always wear a helmet' // attribute name with `.`
   };
  ```

  Modified JSON:

  ```js
  const modified = {
       id: 123,
       // title: 'Bicycle 123', // DELETED
       description: '123 description',
       bicycleType: 'Hybrid',
       brand: 'Brand-Company C',
       price: 600, // UPDATED
       color: ['Red', undefined, 'Blue'], // ADDED color[2] = 'Blue', REMOVED color[1] by setting to undefined, never pop, see why this is bestter below
       productCategory: 'Bicycle',
       inStok: false, // UPDATED boolean true => false
       quantityOnHand: null, // No change, was null in original, still null. DynamoDB recognizes null.
       relatedItems: [100, null, 649], // UPDATE relatedItems[0], REMOVE relatedItems[1], always nullify or set to undefined, never pop
       pictures: {
           frontView: 'http://example.com/products/123_front.jpg',
           rearView: 'http://example.com/products/123_rear.jpg',
           sideView: 'http://example.com/products/123_right_side.jpg' // UPDATED Map item
       },
       productReview: {
           fiveStar: [
               "", // DynamoDB doesn't allow empty string, would be REMOVED
               'Do yourself a favor and buy this.',
               'This is new' // ADDED *deep* list item
           ],
           oneStar: [
               'Actually I take it back, it is alright' // UPDATED *deep* List item
           ]
       },
       comment: 'This product sells out quicly during the summer',
       'Safety.Warning': 'Always wear a helmet, ride at your own risk!' // UPDATED attribute name with `.`
   };
  ```


=============

The returned "UpdateExpression" object would be:

```
{
    "UpdateExpression":
    "SET #color[2] = :color2, #productReview.#fiveStar[2] = :productReviewFiveStar2,
    #inStok = :inStok, #pictures.#sideView = :picturesSideView, #price = :price,
    #productReview.#oneStar[0] = :productReviewOneStar0, #relatedItems[0] = :relatedItems0,
    #safetyWarning = :safetyWarning
    REMOVE #color[1], #productReview.#fiveStar[0], #relatedItems[1], #title", // NOTE: line wrapped here for readability. Genertated UpdateExpression does not include new lines

    "ExpressionAttributeNames": {
        "#color": "color",
        "#fiveStar": "fiveStar",
        "#inStok": "inStok",
        "#oneStar": "oneStar",
        "#pictures": "pictures",
        "#price": "price",
        "#productReview": "productReview",
        "#relatedItems": "relatedItems",
        "#safetyWarning": "Safety.Warning",
        "#sideView": "sideView",
        "#title": "title"
    },
    "ExpressionAttributeValues": {
        ":color2": "Blue",
        ":inStok": false,
        ":picturesSideView": "http://example.com/products/123_right_side.jpg",
        ":price": 600,
        ":productReviewFiveStar2": "This is new",
        ":productReviewOneStar0": "Actually I take it back, it is alright",
        ":relatedItems0": 100,
        ":safetyWarning": "Always wear a helmet, ride at your own risk!"
    }
}
```

## API
#### getUpdateExpression({original, modified, orphans = false})
Generates a comprehensive update expression for added, modified, and removed attributes at any arbitrary deep paths

Parameters:
- original: original document, either fully loaded from dynamodb or a partial projection.
- modified: document that includes additions, modifications and deletes
- orphans: Use orphans = false (default) when you are using DynamoDB to store free style document structure.
DynamoDB doesn't allow SET operations on a deep path if some levels are missing. By using orphans = false, *dynamo-update-expression* would make sure to produce
SET expressions for the first ancestor node that is not in your original document. This can go as deep as required.

##### Example: getUpdateExpression({original, modified/\*, orphans = false\*/})
```js
const partial = {
    id: 123,
    title: 'Bicycle 123',
    inStock: false,
    description: '123 description'
};
const modified = {
    id: 123,
    title: 'Bicycle 123',
    inStock: true,
    stock: 10,
    description: 'modified 123 description',
    pictures: {
        topView: 'http://example.com/products/123_top.jpg'
    }
};

const updateExpression = due.getUpdateExpression({original: partial, modified});
/** generates:
{
    "ExpressionAttributeNames": {
        "#description": "description",
        "#inStock": "inStock",
        "#pictures": "pictures",
        "#stock": "stock"
    },
    "ExpressionAttributeValues": {
        ":description": "modified 123 description",
        ":inStock": true,
        ":pictures": {
            "topView": "http://example.com/products/123_top.jpg"
        },
        ":stock": 10
    },
    "UpdateExpression": "SET #pictures = :pictures, #stock = :stock, #description = :description, #inStock = :inStock"
}
**/
```

Notice how `SET #pictures = :pictures` was generated, where `:pictures` value including the whole new node as a new value. While this would successfully preserve your changes, you would be overwriting any existing Map at the path $.pictures.
Of course if your *original* document was freshly loaded from DynamoDB, then you have nothing to worry about, only the new nodes would be added.

In case you know that you are starting with a *partial* document, you would need to make a choice, to allow orphans and preserve any possible Map/List at the path,
or to overwrite the whole node with your update.
By default, the module would generates an update expression that won't be considered *invalid* by DynamoDB for including path with levels not existing in your table,
i.e. if `SET #pictures.#topView` is used, and your DynamoDB Document didn't not have `pictures` map, you would get an error: "The document path provided in the update expression is invalid for update" when you call `documentClient.update(...updateExpression)` .

In the use cases where you document has a predefined structure, and you won't want to allow free-style additions and you need to make sure that partial updates for valid deep paths are not overwriting parent nodes, set `orphans = true`.

Here is the same example with *orphans = true*

##### Example: getUpdateExpression({original, modified, orphans = true})
```js
const partial = {
    id: 123,
    title: 'Bicycle 123',
    inStock: false,
    description: '123 description'
};
const modified = {
    id: 123,
    title: 'Bicycle 123',
    inStock: true,
    stock: 10,
    description: 'modified 123 description',
    pictures: {
        topView: 'http://example.com/products/123_top.jpg'
    }
};

const updateExpression = due.getUpdateExpression({original: partial, modified, orphans: true});
/** generates:
{
    "ExpressionAttributeNames": {
        "#description": "description",
        "#inStock": "inStock",
        "#pictures": "pictures",
        "#stock": "stock",
        "#topView": "topView"
    },
    "ExpressionAttributeValues": {
        ":description": "modified 123 description",
        ":inStock": true,
        ":picturesTopView": "http://example.com/products/123_top.jpg",
        ":stock": 10
    },
    "UpdateExpression": "SET #pictures.#topView = :picturesTopView, #stock = :stock, #description = :description, #inStock = :inStock"
}
**/
```

Notice how `SET #pictures.#topView = :picturesTopView` was used. This would successfully set this attribute into an existing Map, or error if the parent path does not exist in your document.

Again, this behavior can go as deep as required, for example:

##### Example: Deep addition (into a possibly partial document) default behaviour
```js
const partial = {
    id: 123,
    title: 'Bicycle 123',
    inStock: false,
    description: '123 description'
};
const modified = {
    id: 123,
    title: 'Bicycle 123',
    inStock: true,
    stock: 10,
    description: 'modified 123 description',
    productReview: {
        fiveStar: {
            comment: 'Such a fantastic item!'
        }
    }
};

const updateExpression = due.getUpdateExpression({original: partial, modified});
/** generates:
{
    "ExpressionAttributeNames": {
        "#description": "description",
        "#inStock": "inStock",
        "#productReview": "productReview",
        "#stock": "stock"
    },
    "ExpressionAttributeValues": {
        ":description": "modified 123 description",
        ":inStock": true,
        ":productReview": {
            "fiveStar": {
                "comment": "Such a fantastic item!"
            }
        },
        ":stock": 10
    },
    "UpdateExpression": "SET #productReview = :productReview, #stock = :stock, #description = :description, #inStock = :inStock"
}
**/
```

Notice: `SET #productReview = :productReview` where `":productReview": { "fiveStar": { "comment": "Such a fantastic item!" } }`

##### Example: Deep addition (into a possibly partial document) orphans = true behavior

```js
const partial = {
    id: 123,
    title: 'Bicycle 123',
    inStock: false,
    description: '123 description'
};
const modified = {
    id: 123,
    title: 'Bicycle 123',
    inStock: true,
    stock: 10,
    description: 'modified 123 description',
    productReview: {
        fiveStar: {
            comment: 'Such a fantastic item!'
        }
    }
};

const updateExpression = due.getUpdateExpression({original: partial, modified, orphans: true});
/** generates:
{
    "ExpressionAttributeNames": {
        "#comment": "comment",
        "#description": "description",
        "#fiveStar": "fiveStar",
        "#inStock": "inStock",
        "#productReview": "productReview",
        "#stock": "stock"
    },
    "ExpressionAttributeValues": {
        ":description": "modified 123 description",
        ":inStock": true,
        ":productReviewFiveStarComment": "Such a fantastic item!",
        ":stock": 10
    },
    "UpdateExpression": "SET #productReview.#fiveStar.#comment = :productReviewFiveStarComment, #stock = :stock, #description = :description, #inStock = :inStock"
}
**/
```

Notice: `SET #productReview.#fiveStar.#comment = :productReviewFiveStarComment` where `":productReviewFiveStarComment": "Such a fantastic item!",`

The choice is yours depending on how you want the structure of your document to be, allowing free-style updates or only allowing strict-schema-like updates

#### getVersionedUpdateExpression({original = {}, modified = {}, versionPath = '$.version', useCurrent = true, condition = '=', orphans = false, currentVerion})
Generates a conditional update expression that utilizes DynamoDB's guarantee for [Optimistic Locking With Version Number](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBMapper.OptimisticLocking.html) to make sure that updates are not lost or applied out of order and that stale data is not being used for modifications.
Always remember that you can choose any attribute to be your version attribute, no matter how deeply embedded in the document.

Parameters:
- original: original document, either fully loaded from DynamoDB or a partial projection.
- modified: document that includes additions, modifications and deletes
- versionPath: JSONPATH path to your version attribute of choice, default: '$.version'
- useCurrent: if true, the value @ versionPath is read from original document is used in the condition expression, otherwise, the modified version is used.
- condition: currently supporting simple binary operators kind of string, condition expression would be for example: #version = :version, meaning the version attribute in DynamoDB < the selected version value (current or new)
- orphans: see above.
- currentVersion: Optional. If passed, allows your code to override reading currentVersion from original document, see example below.

##### Example: Only update if version in DynamoDB is (*still*) equal to original document
```js
const original = {parent: {child: 'original value'}, version: 1};
const modified = {parent: {child: 'new value'}, version: 2};
const updateExpression = due.getVersionedUpdateExpression({
    original, modified,
    condition: '='
});

/** generates:
{
    "ConditionExpression": "#expectedVersion = :expectedVersion",
    "ExpressionAttributeNames": {
        "#child": "child",
        "#expectedVersion": "version",
        "#parent": "parent",
        "#version": "version"
    },
    "ExpressionAttributeValues": {
        ":expectedVersion": 1,
        ":parentChild": "new value",
        ":version": 2
    },
    "UpdateExpression": "SET #parent.#child = :parentChild, #version = :version"
}
**/
```

If the condition is not met, the update fails with `ConditionalCheckFailedException` Error. The client can choose to refresh his copy to the latest version (by re-loading from DynamoDB) before trying again.

##### Example: Only update if version in DynamoDB does not exist (no need to pre-load original document)
```js
const modified = {coupon: {code: 'HG74XSD'}, price: 10};
const updateExpression = due.getVersionedUpdateExpression({
    modified,
    versionPath: '$.coupon.code'
});

/** generates:
{
  "ConditionExpression": "attribute_not_exists (#expectedCoupon.#expectedCode)",
  "ExpressionAttributeNames": {
    "#coupon": "coupon",
    "#expectedCode": "code",
    "#expectedCoupon": "coupon",
    "#price": "price"
  },
  "ExpressionAttributeValues": {
    ":coupon": {
      "code": "HG74XSD"
    },
    ":price": 10
  },
  "UpdateExpression": "SET #coupon = :coupon, #price = :price"
}
**/
```

##### Example: Try-Lock next 5 minutes if current expiry < now
```js
const partial = { expiry: 1499758452832 }; // now
const modified = { expiry: 1499762052832}; // now + 5 min
const updateExpression = due.getVersionedUpdateExpression({
    original: partial,
    modified,
    versionPath: '$.expiry',
    condition: '<'
});
/** generates:
{
    "ConditionExpression": "#expectedExpiry < :expectedExpiry",
    "ExpressionAttributeNames": {
        "#expectedExpiry": "expiry",
        "#expiry": "expiry"
    },
    "ExpressionAttributeValues": {
        ":expectedExpiry": 1499758452832,
        ":expiry": 1499762052832
    },
    "UpdateExpression": "SET #expiry = :expiry"
}
**/
```

Client would ideally `wait` and try again to lock the range if update failed.
Behavior is inspired by [this post](http://vilkeliskis.com/articles/distributed-locks-with-dynamodb) and yields same behavior.

##### Advanced Examples

##### Example: override condition default prefix `expected`
Notice how condition attributes are auto-prefixed with `expected` and camelCased. In general this approach is safer to avoid name/value alias collision, especially in the use cases where you SET version attribute to some new value, while your condition uses current.
In case you want to override the prefix, you can, as follows:
```js
const modified = {coupon: {code: 'HG74XSD'}, price: 10};
const updateExpression = due.getVersionedUpdateExpression({
    modified,
    versionPath: '$.coupon.code',
    orphans: true,
    aliasContext: {prefix: ''}
});

/** generates:
{
    "ConditionExpression": "attribute_not_exists (#coupon.#code)",
    "ExpressionAttributeNames": {
        "#code": "code",
        "#coupon": "coupon",
        "#price": "price"
    },
    "ExpressionAttributeValues": {
        ":couponCode": "HG74XSD",
        ":price": 10
    },
    "UpdateExpression": "SET #coupon.#code = :couponCode, #price = :price"
}
**/
```

##### Example: Override current version not_exists detection by overriding currentVerion using `currentVersion` paramter
```js
const modified = {coupon: {code: 'HG74XSD'}, price: 10};
const updateExpression = due.getVersionedUpdateExpression({
    modified,
    versionPath: '$.coupon.code',
    orphans: true,
    useCurrent: false,
    currentVersion: 'N/A', // any truthy value would do. We don't have to pre-load original document, but we want to check for inequality not `not_exists`
    condition: '<>'
});

/** generates:
{
    "ConditionExpression": "#expectedCoupon.#expectedCode <> :expectedCouponCode",
    "ExpressionAttributeNames": {
        "#code": "code",
        "#coupon": "coupon",
        "#expectedCode": "code",
        "#expectedCoupon": "coupon",
        "#price": "price"
    },
    "ExpressionAttributeValues": {
        ":couponCode": "HG74XSD",
        ":expectedCouponCode": "HG74XSD",
        ":price": 10
    },
    "UpdateExpression": "SET #coupon.#code = :couponCode, #price = :price"
}
**/
```

### Sugar
For use cases where the version attribute is always incrementing e.g. processing start index.
Also useful for auto-incrementing version, with backward compatibility with documents that were not initially versioned.

#### getVersionLockExpression({original, versionPath = '$.version', newVersion = undefined, condition = '=', orphans = false} = {})
Generates a version check/lock expression.
Useful to implement Try-Lock behaviour by locking an expiry range, or a processing range in first-winner takes it all style.
Can be used auto-version records, taking backward compatibility into consideration. See examples.
Parameters:
- original: original document. Optional in many of the use cases
- versionPath: JSONPATH path to your version attribute of choice, default: '$.version'
- newVersion: new value for the version attribute. Optional in auto-versioning use cases.
- condition: simple binary operato, condition expression would be for example: #version = :version, meaning the version attribute in DynamoDB < the selected version value (current or new)
- orphans: see above.

##### Example: version-lock with auto versioning and backward compatibility
```js
const updateExpression = due.getVersionLockExpression({});
/** geneates:
{
    "ConditionExpression": "attribute_not_exists (#expectedVersion)",
    "ExpressionAttributeNames": {
        "#expectedVersion": "version",
        "#version": "version"
    },
    "ExpressionAttributeValues": {
        ":version": 1
    },
    "UpdateExpression": "SET #version = :version"
}
**/
```

##### Example: conditional update expression for version-lock with new version auto-incremented value ', () => {
```js
const original = {version: 1}; // can be arbitrary complex document, simipliefid for the sake of clarity
const updateExpression = due.getVersionLockExpression({
   original,
   condition: '='
});

/** generates:
{
   ConditionExpression: '#expectedVersion = :expectedVersion',
   ExpressionAttributeNames: {'#expectedVersion': 'version', '#version': 'version'},
   ExpressionAttributeValues: {':expectedVersion': 1, ':version': 2},
   UpdateExpression: 'SET #version = :version'
}
**/
```
Notice above a use case where condition attribute value had to be aliased (prefixed) since there are two values for the same attribute in the UpdateExpression

##### Example: Try-Lock-Range (always incrementing) use case
```js
const newStart = 1000;
const updateExpression = due.getVersionLockExpression({
    versionPath: '$.start',
    newVersion: newStart,
    condition: '<'
});
/** generates:
{
    "ConditionExpression": "#expectedStart < :expectedStart",
    "ExpressionAttributeNames": {
        "#expectedStart": "start",
        "#start": "start"
    },
    "ExpressionAttributeValues": {
        ":expectedStart": 1000,
        ":start": 1000
    },
    "UpdateExpression": "SET #start = :start"
}
**/
```

## Possible use cases

For a more comprehensive list of possible usages see [tests](https://github.com/sdawood/dynamo-update-expression/src/dynamo-update-expression.spec.js)

## Run the tests

  ```
  npm test
  ```

## What if the document has long attribute names or many nested path levels
In some extreme cases the aliased attribute name, or the aliased deep value name might reach or exceed DynamoDB allowed limit of 255 characters (inclusive of the # character in aliases)
In those cases dynamo-update-expression truncates the name and postfix it with a counter to avoid common-prefix collision.

You'd rarely run into this, still for your peace of mind, here is an example how it would look like:

Note: Long names are shown here with (...) for the sake of readability while illustrating the behavior
##### Example: long attribute name or long deep value alias
```js

const original = {
     id: 123,
     title: 'Bicycle 123',
     description: '123 description',
     bicycleType: 'Hybrid',
     brand: 'Brand-Company C',
     price: 500,
     color: ['Red', 'Black'], // String Set if you use docClient.createSet() before put/update
     productCategory: 'Bicycle',
     inStok: true,
     quantityOnHand: null,
     relatedItems: [341, 472, 649], // Numeric Set if you use docClient.createSet() before put/update
     pictures: {
         frontView: 'http://example.com/products/123_front.jpg',
         rearView: 'http://example.com/products/123_rear.jpg',
         sideView: 'http://example.com/products/123_left_side.jpg'
     },
     productReview: {
         fiveStar: [
             "Excellent! Can't recommend it highly enough! Buy it!",
             'Do yourself a favor and buy this.'
         ],
         oneStar: [
             'Terrible product! Do no buy this.'
         ]
     },
     comment: 'This product sells out quicly during the summer',
     'Safety.Warning': 'Always wear a helmet' // attribute name with `.`
    };
const modified = {
     "id": 123,
     "description": "123 description",
     "bicycleType": "Hybrid",
     "brand": "Brand-Company C",
     "price": 500,
     "color": [
         null,
         "Black",
         "Blue"
     ],
     "productCategory": "Bicycle",
     "inStok": true,
     "quantityOnHand": null,
     "relatedItems": [
         341,
         null,
         649,
         1000
     ],
     "pictures": {
         "frontView": "http://example.com/products/123_front.jpg",
         "sideView": "http://example.com/products/123_left_side.jpg",
         "otherSideView": "pictures.otherSideView"
     },
     "productReview": {
         "fiveStar": [
             null,
             null
         ],
         "oneStar": [
             null,
             "Never again!"
         ],
         "thisIsAVeryLongAttributeNameAndHadToKeepTypingRandomWordsToTryToGetUpTo255CharactersYouWouldThinkThatThisIsEnoughOrThatItWillHappenOftenWhenYouHaveAnAttributeThatLongYouMightAlsoOpenAnIssueAboutItPleaseDoNotSinceTheLibraryDoesTrimYourNamesAndLimitAliasLen": "Value for attribute name with 255 characters excluding the parent path"
     },
     "comment": "This product sells out quicly during the summer",
     "Safety.Warning": "Value for attribute with DOT",
     "root0": "root0",
     "newParent": {
         "newChild1": {
             "newGrandChild1": "c1gc1",
             "newGrandChild2": "c1gc"
         },
         "newChild2": {
             "newGrandChild1": "c2gc1",
             "newGrandChild2": "c2gc2"
         },
         "newChild3": {}
     },
     "prefix-suffix": "Value for attribute name with -",
     "name with space": "name with spaces is also okay",
     "1atBeginning": "name starting with number is also okay",
     "thisIsAVeryLongAttributeNameAndHadToKeepTypingRandomWordsToTryToGetUpTo255CharactersYouWouldThinkThatThisIsEnoughOrThatItWillHappenOftenWhenYouHaveAnAttributeThatLongYouMightAlsoOpenAnIssueAboutItPleaseDoNotSinceTheLibraryDoesTrimYourNamesAndLimitAliasLen": [
         "Value for attribute name with 255 characters with subscript excluding the parent path"
     ]
 };

/** generates:
{
    "ExpressionAttributeNames": {
        "#1AtBeginning": "1atBeginning",
        "#color": "color",
        "#fiveStar": "fiveStar",
        "#nameWithSpace": "name with space",
        "#newParent": "newParent",
        "#oneStar": "oneStar",
        "#otherSideView": "otherSideView",
        "#pictures": "pictures",
        "#prefixSuffix": "prefix-suffix",
        "#productReview": "productReview",
        "#rearView": "rearView",
        "#relatedItems": "relatedItems",
        "#root0": "root0",
        "#safetyWarning": "Safety.Warning",
        "#thisIsAVeryLongAttributeName...LimitAliasL1": "thisIsAVeryLongAttributeName...DoesTrimYourNamesAndLimitAliasLen",
        "#thisIsAVeryLongAttributeName...LimitAliasL3": "thisIsAVeryLongAttributeName...DoesTrimYourNamesAndLimitAliasLen",
        "#title": "title"
    },
    "ExpressionAttributeValues": {
        ":1AtBeginning": "name starting with number is also okay",
        ":color2": "Blue",
        ":nameWithSpace": "name with spaces is also okay",
        ":newParent": {
            "newChild1": {
                "newGrandChild1": "c1gc1",
                "newGrandChild2": "c1gc"
            },
            "newChild2": {
                "newGrandChild1": "c2gc1",
                "newGrandChild2": "c2gc2"
            },
            "newChild3": {}
        },
        ":picturesOtherSideView": "pictures.otherSideView",
        ":prefixSuffix": "Value for attribute name with -",
        ":productReviewOneStar1": "Never again!",
        ":productReviewThisIsAVeryLongAttributeName...TrimYourNamesA2": "Value for attribute name with 255 characters excluding the parent path",
        ":relatedItems3": 1000,
        ":root0": "root0",
        ":safetyWarning": "Value for attribute with DOT",
        ":thisIsAVeryLongAttributeName...LimitAliasL4": [
            "Value for attribute name with 255 characters with subscript excluding the parent path"
        ]
    },
    "UpdateExpression": "SET #color[2] = :color2, #newParent = :newParent,
    #pictures.#otherSideView = :picturesOtherSideView, #productReview.#oneStar[1] = :productReviewOneStar1,
    #productReview.#thisIsAVeryLongAttributeName...LimitAliasL1 = :productReviewThisIsAVeryLongAttributeName...TrimYourNamesA2,
    #relatedItems[3] = :relatedItems3, #root0 = :root0,
    #thisIsAVeryLongAttributeName...LimitAliasL3 = :thisIsAVeryLongAttributeName...LimitAliasL4,
    #1AtBeginning = :1AtBeginning, #nameWithSpace = :nameWithSpace, #prefixSuffix = :prefixSuffix,
    #safetyWarning = :safetyWarning
    REMOVE #color[0], #pictures.#rearView, #productReview.#fiveStar[0], #productReview.#fiveStar[1],
    #productReview.#oneStar[0], #relatedItems[1], #title"
}
**/
```

## License

[MIT](LICENSE)