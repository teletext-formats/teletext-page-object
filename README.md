# Teletext Page Object Format
A JSON based format to unambiguously represent teletext pages for page creation and storage in web based systems and APIs.

## Introduction
There exist many ways to represent the data comprising a teletext page, each with their own advantages and disadvantages for any particular use case.
Like all good standards creators, the authors believe that all the world's ills can best be solved by the introduction of yet another competing format[^xkcd].

The goals of this format are to provide a method of storing and conveying teletext pages which is:
 - unambiguous
 - easily generated and parsed using standard routines
 - able to describe any valid teletext page content
 - able to convey metadata controlling their use within a service
 - text based and somewhat human readable/editable

[JSON](https://www.json.org/) was chosen as the underlying technology and it was further decided to formalise the structure using a [JSON Schema](https://json-schema.org/) to facilitate easy validation of objects on import etc.

The primary use case for this format is within teletext service management systems, and the APIs for updating and distributing those services.
The objects may represent complete pages ready to be broadcast, or partial content used in the generation of pages (i.e. templating functionality).
The format is **not** intended for archival of teletext packet data[^archival].

## Versioning
This format and subsequent revisions are versioned using the SchemaVer philosophy[^schemaver].

This document describes version 1-0-0 as defined by the JSON schema located at [https://teletext-formats.github.io/teletext-page-object/schema/1-0-0/teletext-page-object.json](https://teletext-formats.github.io/teletext-page-object/schema/1-0-0/teletext-page-object.json).

The schema is the canonical definition of a valid page object, which the remainder of this document will describe in broader terms.

## The `page` object
This is the root object described by this format. It contains the following properties:

 - An optional [`pageNumber`](#the-pagenumber-property) property named `number`
 - An optional [`controlFlags`](#the-controlflags-object) object named `control`
 - An optional array of [`packet`](#the-packet-object) objects named `packets`
 - An array of [`subpage`](#the-subpage-object) objects named `subpages`
 - An optional boolean flag named `timecode`
 - An optional boolean flag named `inherit`
 - An optional [`metadata`](#the-metadata-object) object named `metadata`

If no `number` is specified the page shall not be passed on to an inserter for transmission. This is intended to facilitate the preparation of templates and draft pages not yet assigned to a page number.

`number` must not end in `FF`.

`control` sets the default header control bits for subpages.

`packets` is an array containing a variable number of common [`packet`](#the-packet-object) objects which set defaults for subpages.
Each packet object must describe a unique packet number and designation code[^unique].

`subpages` is an array containing one or more [`subpage`](#the-subpage-object) objects.
Subpages are defined in transmission order.

`timecode` sets the timecode flag default for all subpages.

Page objects have a concept of inheritance, where unspecified data is inherited as a default from a parent object.
For example a page object may define default packet data to be used in all subpage objects.

The page object inherits the value of any unspecified control bits from a system default or parent object outside the scope of this specification[^parentobject].

Similarly default packet data is inherited unless the `inherit` flag is set `false`.

`number` and `subpages` cannot be inherited.

## The `packet` object
This object defines a single teletext packet within a page or subpage.
It must contain the following properties:

 - A packet number in the range 0 to 28 named `number`
 - One of the following packet data properties:
     - [`text`](#the-text-property)
     - [`linking`](#the-linking-property)
     - [`triplets`](#the-triplets-property)
     - [`hamming`](#the-hamming-property)
     - [`base64`](#the-base64-property)
 - A designation code in the range 0 to 15 named `dc` if packet contains `triplets` or `linking` data.

#### The `text` property
A string containing up to 40 characters of 7-bit text and spacing attribute codes.
The control codes `0x00`-`0x1F`, quotation marks (`"`), and reverse slash (`\`), must be escaped using standard JSON escape sequences.

Where the string is less than 40 characters in length the remainder of the packet shall be interpreted as being filled with the character `0x20` (space).

When `number` equals 0, the first eight characters should be filled with character `0x20` (space), and shall be discarded by an inserter.

This property is valid only when `number` is in the range 0 to 25.

#### The `linking` property
An object containing an array of six `link` objects, and an optional `linkControl` byte.

Each `link` object may contain a [`pageNumber`](#the-pagenumber-property) property named `page`, and a [`pageSubcode`](#the-pagesubcode-property) property named `subcode`.

If `page` is absent the default value `8FF` shall be used.
If `subcode` is absent the default value `3F7F` shall be used.

`subcode` requires `page` and must equal `3F7F` if the value of `page` ends in `FF`.

The `linkControl` byte is encoded as an integer from 0 to 15.
The default value is `15`.
`linkControl` is only valid for packets with `dc` equal to 0.

This property is only valid when `number` equals 27,  and `dc` is in the range 0 to 3.

#### The `triplets` property
An array of thirteen Hamming 24/18 *triplets*.
Each triplet is stored as an unencoded 18-bit integer.

The first byte of the packet shall be filled appropriately by an inserter based on the designation code, or page and packet coding.

This property is valid only when `number` is in the range 1 to 28.
When `number` equals 27, `dc` must be in the range 4 to 5.

For packets 1 to 25 the value in `dc` **does not** address unique packets[^unique].

#### The `hamming` property
An array containing forty Hamming 8/4 *nibbles*.
Each nibble is stored as an unencoded 4 bit integer.

The first byte will be replaced by an inserter if required by the Page Coding per Section 9.4.1.2 of the Enhanced Teletext specification[^spec].

#### The `base64` property
A string containing 40 bytes of raw binary packet data encoded as 56 base64 characters 

The data must include any parity and error correction bits required, and shall be transmitted as-is.
The first byte shall be ignored by an inserter and replaced by the designation code when `number` is in the range 26 to 28.

This encoding is intended only for binary data or to support non-standard packet encodings.

This property is valid only when `number` is in the range 1 to 28.

## The `subpage` object
This object defines a subpage and may contain any of the following properties:

 - An explicit [`pageSubcode`](#the-pagesubcode-type) property named `subcode`
 - A boolean flag named `timecode`
 - A [`controlFlags`](#the-controlflags-object) object named `control`
 - An array of [`packet`](#the-packet-object) objects named `packets`
 - A boolean flag named `inherit`

In the absence of an explicit `subcode`, the following rules apply:
 1. If `timecode` is false, the subpage will be numbered automatically[^subpagenumbering] according to Annex A.1 of the Enhanced Teletext specification[^spec].
 2. If `timecode` is true, the transmitted subcode will encode the current time per Annex E.2 of the Enhanced Teletext specification[^spec].

In the absence of `timecode` the default is inherited from the parent page object.

`subcode` and `timecode` shall be ignored for data pages (e.g. MOT, POP, GPOP, DRCS, GDRCS, MIP).

`subcode` must not equal `3F7F`.

`control` sets the header control bits for the subpage.
Any unspecified control bits are inherited from the parent page object.

`packets` is an array containing a variable number of [`packet`](#the-packet-object) objects.
Each packet object must describe a unique packet number and designation code[^unique].

Default packet data is inherited from the parent page object unless the `inherit` flag is set `false`.

## The `pageNumber` type
A teletext page number as defined in Section 3.1 of the Enhanced Teletext specification[^spec].
The value must be a valid page number, encoded as a three digit hexadecimal string.

## The `pageSubcode` type
A teletext page subcode as defined in Section 3.1 of the Enhanced Teletext specification[^spec].
The value must be a valid subcode, encoded as a four digit hexadecimal string.

## The `controlFlags` object
An object containing any combination of boolean page control bits, and a National Option Character subset.

This object can define all teletext page header control bits except for `C11` (magazine serial) which is under the control of the inserter.

| Bit(s) | Name | Type |
|--|--|--|
| `C4` | `erasePage` | `boolean`
| `C5` | `newsflash` | `boolean`
| `C6` | `subtitle` | `boolean`
| `C7` | `suppressHeader` | `boolean`
| `C8` | `update` | `boolean`
| `C9` | `interruptedSequence` | `boolean`
| `C10` | `suppressPage` | `boolean`
| `C12-C14` | `language` | `integer`

Any unspecified properties inherit their value from the [root page object](#the-page-object), else a system default or parent object outside the scope of this specification[^parentobject].

The `C9` (interrupted sequence) bit is provided to allow pages to be hidden within rotating headers. An inserter shall ignore this setting when a page is transmitted out of sequence.

The `C4`, `C8`, and `C12`-`C14` bits have special meaning for presentation enhancement data pages (e.g. MOT, POP, GPOP, DRCS, GDRCS) and may be generated automatically by an inserter ignoring these settings.

## The `metadata` object
An object containing the following optional properties:

- A string named `title`
- A string named `author`
- A string named `comment`
- A data-time[^datetime] string named `created`
- A data-time[^datetime] string named `validFrom`
- A data-time[^datetime] string named `validUntil`
- An integer named `cycleSeconds`
- An integer named `cycleCycles`

No specific constraints are given to the `title`, `author`, and `comment` strings by this specification, however it is suggested that they should be human-readable, and that `title` and `author` are kept to a short length.

The `created` date-time is expected to represent the time a page object was created or modified.
`validFrom` and `validUntil` are intended to facilitate automatic scheduling of page transmission.

`cycleSeconds` and `cycleCycles` indicate the desired subpage cycling interval in seconds or magazine page cycles and are mutually exclusive.

[^xkcd]: [The xkcd webcomic "How standards proliferate"](https://xkcd.com/927)
[^archival]: Raw packet data formats e.g. *t42* streams can contain invalid packet data. Additionally this format does not define certain details such as transmission related page control flags.
[^schemaver]: [https://snowplow.io/blog/introducing-schemaver-for-semantic-versioning-of-schemas](https://snowplow.io/blog/introducing-schemaver-for-semantic-versioning-of-schemas)
[^unique]: This rule is not enforced by the current schema for technical reasons. The behaviour of software encountering duplicate packet numbers is undefined.
[^subpagenumbering]: Where a single subpage is defined the implicit subcode shall be `0000`. Otherwise, an incrementing decimal count is applied to each implicitly numbered subpage, starting at `0001`. This counter is not affected by explicitly numbered subpages. Implicit subcodes are undefined for pages with greater than 79 implicitly numbered subpages.
[^spec]: [https://www.etsi.org/deliver/etsi_en/300700_300799/300706/01.02.01_60/en_300706v010201p.pdf](https://www.etsi.org/deliver/etsi_en/300700_300799/300706/01.02.01_60/en_300706v010201p.pdf)
[^parentobject]: The intention is to permit the creation of objects which describe groups of page objects and define common content such as page banners.
[^datetime]: An ISO 8601 date and time string in the format defined in [https://tools.ietf.org/html/rfc3339#section-5.6](https://tools.ietf.org/html/rfc3339#section-5.6)
