[avalanche](../README.md) › [src/utils](../modules/src_utils.md) › [TransferableInputError](src_utils.transferableinputerror.md)

# Class: TransferableInputError

## Hierarchy

  ↳ [AvalancheError](src_utils.avalancheerror.md)

  ↳ **TransferableInputError**

## Index

### Constructors

* [constructor](src_utils.transferableinputerror.md#constructor)

### Properties

* [errorCode](src_utils.transferableinputerror.md#errorcode)
* [message](src_utils.transferableinputerror.md#message)
* [name](src_utils.transferableinputerror.md#name)
* [stack](src_utils.transferableinputerror.md#optional-stack)

### Methods

* [getCode](src_utils.transferableinputerror.md#getcode)

## Constructors

###  constructor

\+ **new TransferableInputError**(`m`: string): *[TransferableInputError](src_utils.transferableinputerror.md)*

*Overrides [AvalancheError](src_utils.avalancheerror.md).[constructor](src_utils.avalancheerror.md#constructor)*

*Defined in [src/utils/errors.ts:130](https://github.com/ava-labs/avalanchejs/blob/4e59193/src/utils/errors.ts#L130)*

**Parameters:**

Name | Type |
------ | ------ |
`m` | string |

**Returns:** *[TransferableInputError](src_utils.transferableinputerror.md)*

## Properties

###  errorCode

• **errorCode**: *string*

*Inherited from [AvalancheError](src_utils.avalancheerror.md).[errorCode](src_utils.avalancheerror.md#errorcode)*

*Defined in [src/utils/errors.ts:48](https://github.com/ava-labs/avalanchejs/blob/4e59193/src/utils/errors.ts#L48)*

___

###  message

• **message**: *string*

*Inherited from [AvalancheError](src_utils.avalancheerror.md).[message](src_utils.avalancheerror.md#message)*

Defined in node_modules/typescript/lib/lib.es5.d.ts:1023

___

###  name

• **name**: *string*

*Inherited from [AvalancheError](src_utils.avalancheerror.md).[name](src_utils.avalancheerror.md#name)*

Defined in node_modules/typescript/lib/lib.es5.d.ts:1022

___

### `Optional` stack

• **stack**? : *string*

*Inherited from [AvalancheError](src_utils.avalancheerror.md).[stack](src_utils.avalancheerror.md#optional-stack)*

Defined in node_modules/typescript/lib/lib.es5.d.ts:1024

## Methods

###  getCode

▸ **getCode**(): *string*

*Inherited from [AvalancheError](src_utils.avalancheerror.md).[getCode](src_utils.avalancheerror.md#getcode)*

*Defined in [src/utils/errors.ts:55](https://github.com/ava-labs/avalanchejs/blob/4e59193/src/utils/errors.ts#L55)*

**Returns:** *string*