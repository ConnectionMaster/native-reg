# `native-reg`

In process native module for Windows registry access.

## Example

```js
const reg = require('native-reg');

const key = reg.openKey(
  reg.HKCU,
  'Software\\MyCompany\\MySoftware',
  reg.Access.ALL_ACCESS);

const version = reg.getValue(key, 'Install', 'Version');

if (isOldVersion(version)) {
    reg.deleteTree(key, 'Install');
    
    const installKey = reg.createKey(key, 'Install', reg.Access.ALL_ACCESS);
    reg.setValue(installKey, 'Version', newVersion);
    // ...
    reg.closeKey(installKey);
}

reg.closeKey(key);
```

## API

Read `index.ts` and linked Windows documentation for full details.

### Errors

The API initially validates the arguments with the standard node `assert` library:

```js
try {
  // Should not be 'HKLM', but reg.HKLM or reg.HKEY.LOCAL_MACHINE
  reg.openKey('HKLM', 'SOFTWARE\\Microsoft\\Windows', reg.Access.READ);
  assert.fail();
} catch (e) {
  assert(e instanceof require('assert').AssertionError);
}
```

If the wrapped Windows API returns an error, with a couple of exceptions for
commonly non-failure errors (e.g. key does not exist), are thrown as JS `Error`s
with the generic error message, e.g.: `Access is denied.`, but additional
standard `errno` and `syscall` properties, for example, a common error is trying
to use `Access.ALL_ACCESS` on `reg.HKLM`, which you need UAC elevation for:

```js
try {
  // Assuming you are not running as administrator!
  reg.openKey(reg.HKLM, 'SOFTWARE\\Microsoft\\Windows', reg.Access.ALL_ACCESS);
  assert.fail();
} catch (e) {
  assert.strictEqual(e.message, 'Access is denied.');
  assert.strictEqual(e.errno, 5);
  assert.strictEqual(e.syscall, 'RegOpenKeyExW');
}
```

### Constants

This library uses Typescript `enum`s for constants, this generates a
two way name <-> value mapping.

For example: `Access.SET_VALUE` is `0x0002`, and `Access[2]` is `"SET_VALUE"`.

#### `HKEY` enum

Exports the set of predefined `HKEY`s.

[`createKey`](#createkey), [`openKey`](#openkey), [`loadAppKey`](#loadappkey) and
[`openCurrentUser`](#opencurrentuser) will return other values for `HKEY`s,
which at the moment are objects with a single property `native` with the native
HKEY handle as a V8 `External` value, for use in other native packages, or `null`
after it is closed.
For these values you can call [`closeKey`](#closekey) once you are done to clean up
early, but they will be closed by the garbage collector if you chose not to.

```ts
export enum HKEY {
  CLASSES_ROOT                   = 0x80000000,
  CURRENT_USER                   = 0x80000001,
  LOCAL_MACHINE                  = 0x80000002,
  USERS                          = 0x80000003,
  PERFORMANCE_DATA               = 0x80000004,
  PERFORMANCE_TEXT               = 0x80000050,
  PERFORMANCE_NLSTEXT            = 0x80000060,
  CURRENT_CONFIG                 = 0x80000005,
  DYN_DATA                       = 0x80000006,
  CURRENT_USER_LOCAL_SETTINGS    = 0x80000007,
}
```

##### Shorthands

Also exports the standard shorthand HKEY names:

```ts
export const HKCR = HKEY.CLASSES_ROOT;
export const HKCU = HKEY.CURRENT_USER;
export const HKLM = HKEY.LOCAL_MACHINE;
export const HKU = HKEY.USERS;
```

##### `isHKEY`

Helper returns if the argument is a valid-looking `HKEY`. Most APIs will throw
an assertion error if `hkey` that does not return true for this is used.

Valid HKEY values are,
* The values of the [`HKEY` enum](#hkey-enum)
* Objects returned from [`createKey`](#createkey), [`openKey`](#openkey),
  [`loadAppKey`](#loadappkey) and [`openCurrentUser`](#opencurrentuser)
* `External` values that represent HKEYs (for example, from another node
  addon)
* Non-`0` 32-bit values that represent the pointer value of HKEYs
  (for example, from another node addon) - not that this is unreliable on
  64-bit applications, and should be avoided.

```ts
export function isHKEY(hkey: any): boolean;
```

#### `Access` enum

Specifies access checks for opened or created keys. Not always enforced for
opened keys:

> Certain registry operations perform access checks against the security descriptor of the key, not the access mask specified when the handle to the key was obtained. For example, even if a key is opened with a samDesired of KEY_READ, it can be used to create registry keys if the key's security descriptor permits. In contrast, the RegSetValueEx function specifically requires that the key be opened with the KEY_SET_VALUE access right.

```ts
// from https://docs.microsoft.com/en-nz/windows/desktop/SysInfo/registry-key-security-and-access-rights
export enum Access {
  // Specific rights
  QUERY_VALUE         = 0x0001,
  SET_VALUE           = 0x0002,
  CREATE_SUB_KEY      = 0x0004,
  ENUMERATE_SUB_KEYS  = 0x0008,
  NOTIFY              = 0x0010,
  CREATE_LINK         = 0x0020,

  // WOW64. See https://docs.microsoft.com/en-nz/windows/desktop/WinProg64/accessing-an-alternate-registry-view
  WOW64_64KEY         = 0x0100,
  WOW64_32KEY         = 0x0200,

  // Generic rights.
  READ              = 0x2_0019,
  WRITE             = 0x2_0006,
  EXECUTE           = READ,

  ALL_ACCESS        = 0xF_003F,
}
```

#### `ValueType` enum

Types for registry values.

```ts
export enum ValueType  {
  NONE                       = 0, // No value type
  SZ                         = 1, // Unicode nul terminated string
  EXPAND_SZ                  = 2, // Unicode nul terminated string
                                  // (with environment variable references)
  BINARY                     = 3, // Free form binary
  DWORD                      = 4, // 32-bit number
  DWORD_LITTLE_ENDIAN        = 4, // 32-bit number (same as REG_DWORD)
  DWORD_BIG_ENDIAN           = 5, // 32-bit number
  LINK                       = 6, // Symbolic Link (unicode)
  MULTI_SZ                   = 7, // Multiple Unicode strings
  RESOURCE_LIST              = 8, // Resource list in the resource map
  FULL_RESOURCE_DESCRIPTOR   = 9, // Resource list in the hardware description
  RESOURCE_REQUIREMENTS_LIST = 10,
  QWORD                      = 11, // 64-bit number
  QWORD_LITTLE_ENDIAN        = 11, // 64-bit number (same as REG_QWORD)
}
```

### Raw APIs

These APIs fairly directly wrap the Windows API linked, only abstracting some of
the allocation and general usage style.

The exception is [`enumKeyNames`](#enumkeynames) and [`enumValueNames`](#enumvaluenames)
which iterate to build a list and only return the names, and not other properties.

#### `Value`

Raw registry values returned from [`queryValueRaw`](#queryvalueraw) and [`getValueRaw`](#getvalueraw)
are simply Node `Buffer`s with an additional `type` property from `ValueType`:

```ts
export type Value = Buffer & { type: ValueType };
```

#### `createKey`

Wraps [`RegCreateKeyExW`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-regcreatekeyexw)

> Creates the specified registry key. If the key already exists, the function opens it. Note that key names are not case sensitive.

You must call [`closeKey`](#closekey) on the result to clean up.

```ts
export function createKey(
  hkey: HKEY,
  subKey: string,
  access: Access,
  options: CreateKeyOptions = 0,
): HKEY;
```

#### `openKey`

Wraps [`RegOpenKeyExW`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-regopenkeyexw)

> Opens the specified registry key. Note that key names are not case sensitive.

Returns `null` if `subKey` does not exist under `hkey`.

You must call [`closeKey`](#closekey) on the result to clean up.

```ts
export function openKey(
  hkey: HKEY,
  subKey: string,
  access: Access,
  options: OpenKeyOptions = 0,
): HKEY | null;
```

#### `loadAppKey`

Wraps [`RegLoadAppKeyW`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-regloadappkeyw)

> Loads the specified registry hive as an application hive.

You must call [`closeKey`](#closekey) on the result to clean up.

```ts
export function loadAppKey(file: string, access: Access): HKEY;
```

#### `openCurrentUser`

Wraps [`RegOpenCurrentUser`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-regopencurrentuser)

> Retrieves a handle to the `HKEY_CURRENT_USER` key for the user the current thread is impersonating.

Only makes sense to use if you have access to Windows user impersonation. Maybe take a look at my
currently WIP [Windows user account package](https://github.com/simonbuchan/native-users-node).

You must call [`closeKey`](#closekey) on the result to clean up.

```ts
export function openCurrentUser(access: Access): HKEY;
```

#### `enumKeyNames`

Wraps [`RegEnumKeyExW`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-regenumkeyexw)
iterated to get the sub key names for a key.

> Enumerates the subkeys of the specified open registry key.

```ts
export function enumKeyNames(hkey: HKEY): string[];
```

#### `enumValueNames`

Wraps [`RegEnumValueW`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-regenumvaluew)
iterated to get the value names for a key.

> Enumerates the values for the specified open registry key.

```ts
export function enumValueNames(hkey: HKEY): string[];
```

#### `queryValueRaw`

Wraps [`RegQueryValueExW`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-regqueryvalueexw)
without additional parsing.

> Retrieves the type and data for the specified value name associated with an open registry key.

You may want to use [`queryValue`](#queryvalue) instead.

Returns `null` if `valueName` does not exist under `hkey`.

```ts
export function queryValueRaw(hkey: HKEY, valueName: string): Value | null;
```

#### `getValueRaw`

Wraps [`RegGetValueExW`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-reggetvaluew)
without additional parsing.

> Retrieves the type and data for the specified registry value.

You may want to use [`getValue`](#getvalue) instead.

Returns `null` if `subKey` or `valueName` does not exist under `hkey`.

```ts
export function getValueRaw(
  hkey: HKEY,
  subKey: string,
  valueName: string,
  flags: GetValueFlags = 0,
): Value | null;
```

#### `deleteKey`

Wraps [`RegDeleteKeyW`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-regdeletekeyw)

> Deletes a subkey and its values. Note that key names are not case sensitive.

Returns true if the key existed before it was deleted.

```ts
export function deleteKey(hkey: HKEY, subKey: string): boolean;
```

#### `deleteTree`

Wraps [`RegDeleteTreeW`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-regdeletetreew)

> Deletes the subkeys and values of the specified key recursively.

Returns true if the key existed before it was deleted.

```ts
export function deleteTree(hkey: HKEY, subKey: string): boolean;
```

#### `deleteKeyValue`

Wraps [`RegDeleteKeyValueW`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-regdeletekeyvaluew)

> Removes the specified value from the specified registry key and subkey.

Returns true if the value existed before it was deleted.

```ts
export function deleteKeyValue(
  hkey: HKEY,
  subKey: string,
  valueName: string,
): boolean;
```

#### `deleteValue`

Wraps [`RegDeleteValueW`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-regdeletevaluew)

> Removes a named value from the specified registry key. Note that value names are not case sensitive.

Returns true if the value existed before it was deleted.

```ts
export function deleteValue(hkey: HKEY, valueName: string): boolean;
```

#### `closeKey`

Wraps [`RegCloseKey`](https://docs.microsoft.com/en-us/windows/desktop/api/winreg/nf-winreg-regclosekey)

> Closes a handle to the specified registry key.

For convenience, `null` or `undefined` values are allowed and ignored.

```ts
export function closeKey(hkey: HKEY | null | undefined): void;
```

### Format helpers

#### `parseValue`

Returns the JS-native value for common `Value` types:
- `SZ`, `EXPAND_SZ` -> `string`
- `BINARY` -> `Buffer`
- `DWORD` / `DWORD_LITTLE_ENDIAN` -> `number`
- `DWORD_BIG_ENDIAN` -> `number`
- `MULTI_SZ` -> `string[]`

Throws if the type is not one of the above!

For convenience, passes through `null` so missing values don't have
to be specially treated.

```ts
export type ParsedValue = number | string | string[] | Buffer;
export function parseValue(value: Value | null): ParsedValue | null;
```

#### `parseString`

Parses `SZ` and `EXPAND_SZ` (etc.) registry values.

```ts
export function parseString(value: Buffer): string;
```

#### `parseString`

Parses `SZ` and `EXPAND_SZ` (etc.) registry values.

```ts
export function parseString(value: Buffer): string;
```

#### `parseMultiString`

Parses `MULTI_SZ` registry values.

```ts
export function parseMultiString(value: Buffer): string[];
```

#### `formatString`

Formats a string to `SZ`, `EXPAND_SZ` (etc.) format.

```ts
export function formatString(value: string): Buffer;
```

#### `formatMultiString`

Formats an array of `string`s to `MULTI_SZ` format.

```ts
export function formatMultiString(values: string[]): Buffer;
```

#### `formatDWORD`

Formats a `number` to `DWORD` / `DWORD_LITTLE_ENDIAN` format.

```ts
export function formatDWORD(value: number): Buffer;
```

#### `formatQWORD`

Formats a `number` to `QWORD` format.

```ts
export function formatQWORD(value: number): Buffer;
```

### Formatted value APIs

These APIs wrap the raw value APIs with the formatting helpers.

#### `setValue{Type}`

Sets the registry value with a matching `{Type}` and formatted value.

For example, `setValueSZ` is `setValueRaw(hkey, valueName, ValueType.SZ, formatString(value))`.

```ts
export function setValueSZ(
  hkey: HKEY,
  valueName: string,
  value: string,
): void;
export function setValueEXPAND_SZ(
  hkey: HKEY,
  valueName: string,
  value: string,
): void;
export function setValueMULTI_SZ(
  hkey: HKEY,
  valueName: string,
  value: string[],
): void;
export function setValueDWORD(
  hkey: HKEY,
  valueName: string,
  value: number,
): void;
export function setValueQWORD(
  hkey: HKEY,
  valueName: string,
  value: number,
): void;
```

#### `getValue`

Wraps `getValueRaw` in `parseValue`.

```ts
export function getValue(
  hkey: HKEY,
  subKey: string,
  valueName: string,
  flags: GetValueFlags = 0,
): ParsedValue | null;
```

#### `queryValue`

Wraps `queryValueRaw` in `parseValue`.

```ts
export function queryValue(hkey: HKEY, valueName: string): ParsedValue | null;
```
