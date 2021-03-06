# SLONE: Serialization of Lists of Ordered Named Elements

The role of SLONE is to enable a text-based document format for machine-to-machine communication that is also human readable and trackable by line-oriented systems such as `diff` and `git`.

## ALTERNATIVES

If you are wanting machine-to-machine communication that does not need to be trackable by line-oriented systems, I recommend using [JSON](https://www.json.org/json-en.html) rather than SLONE as JSON is almost universally supported.

If you are wanting **human**-to-machine communication, such as a config file, I recommend using [YAML](https://yaml.org/) rather than SLONE. YAML is very forgiving whereas SLONE is **VERY** strict and is frustrating to manually type. YAML is also mostly line-oriented. It too has more universal support.

If you are wanting somewhat-line-oriented machine-to-machine communication but are not as concerned with repeatability or byte-level reproduction, [XML](https://www.w3.org/XML/) is also a fantastic alternative.

## GOALS

The specific goals of SLONE are:

1. Make It Easy to Track Changes At The Line Level

   Utilities such as `diff` and `git` should be able to pinpoint the precise differences between two SLONE documents at a line-by-line level.
   
   For example, when adding an entry to a list, the new entry is on it's own line (showing an "add") and should not effect any other lines.

2. Make Almost No Presumptions Of A System's Unique Data Types

   SLONE only supports the following types:

   * strings (UTF8)
   * unknown (aka null)
   * nothing (non-existence)
   * named lists (albeit the names can be "missing" to make them array-like)

   Very specifically, SLONE does not _directly_ support numbers, dates, vectors, etc. except as strings.

3. Yet Support Pretty Much Any Data Type

   When a string value is stored, it is prefixed with another string representing it's type.
   
   In general, SLONE should enable schema management, but not take part in it.

3. Support Fast Simple Single-Pass Parsing

   A document should be verifiable and parsed in a program using a single pass with nothing more than, perhaps, a few simple state variables.

   This is one of the reasons that the format is so strict. Keep in mind that the spec is designed to be human readable; it not meant to be edited or written by humans.

4. Idempontency (Absolutely Consistent Replication)

   If a SLONE document is read by a library and then output to another document, the content of the new document should be absolutely identical to the original.

   Put another way: if a program reads a file encoded in SLONE (without a schema) into a variable, and then it serializes that variable into a new file, then a MD5 checksum both files should return the same result.

   There is only one way to write a SLONE document for any particular collection of data.

5. Make No Presumtion Of The Human Language Either

   Other than the word "SLONE" itself on the first line, the SLONE specification should only use symbols and whitespace to format the
   content of the document.

   It should work equally well with English, Spanish, Mandarin, Arabic, Maltese, and ancient Sumerian.

6. Human Readable, But Not Human Writable

   While a person could, in theory, write a SLONE document by hand, doing so is NOT a goal. It would be generally frustrating. Other specifications such as JSON or YAML are much better suited to this due to their flexibility.

## DESCRIPTION

Data is serialized into "entries", where each entry consists of one or more lines of text.

All text is in UTF8 format. All UTF8 text should be normalized and composed (NFC).

Each line is terminated by a single NewLine (`\n`) character. There is no line length limit.

## FIRST LINE

The very first line of the document is a descriptor that is not part of the data. The descriptor is `#! SLONE 1.0`, exactly, followed by a NewLine. A document missing this line is in error. The spacing and capitalization is not optional.

The word "SLONE" is used to provide an indication that the document contains SLONE content. The "1.0" is for the currently only version possible.

For this specification, the text is a series of "characters" in UTF-8 code point format . So, characters can be 1, 2, or 4 bytes long; depending on the language code page. See published UTF-8 specification for details.

The document ends with NewLine on the last line. There are never any empty lines in the SLONE document.

## OPTIONAL SCHEMA REFERENCE

The second line of a document may optionally indicate a schema. This is done by prefixing the line with a `#%` followed by a space and additional text.

For version 1.0 of SLONE, the "additional text" does not have a specific meaning.

```slone1.0
#! SLONE 1.0
#% https://schemaserver.local/api/slone1.0/person-detail.schema
"Larry" : person = {
  "main home" : building = {
    "mailing address" : address = {
      "street" = {
        _ = "1234 Main St"
        _ = "Unit 3"
      }
      "postal code" : zip_code = "90210"
    }
  }
}
```

## ENTRIES

An entry in a SLONE document is a series of elements of the form:

```
INDENT name SPACE (COLON SPACE type) EQUALS value
```

If type is nothing, then the "COLON SPACE type" part MUST be ommited.

Quick details:

- A `name` string (or `_`) is a reference name for the data entry.
- A colon (`:`) followed by `type` for the data entry describes the value's format or structure. If there is no declared type then both the colon and the `type` are not included.
- An equals symbol (`=`) used before the value of an entry.
- A `value` for the data entry is the string expression of the content, which can be:
  - A string (simple or long).
  - A `?` to indicate that the value is unknown. In SQL parlance, this is often called a NULL.
  - A subdocument, indicated started with a `{` symbol sequence at the start and a `}` symbol sequence at the end.

General notes:

- A null (unknown) element is indicated by a question mark (`?`). It can only be used with values
- A nothing (not-applicable or never-existant) element is indicated by an underscore (`_`). This can be used for names.
- A short quoted string starts and ends with a quote symbol (`"`); all on one line.
- A long quoted string. It starts with a pipe symbol (`|`) and ends with a pipe symble (`|`) on multiple lines.
- The order of the entries is significant.

Unless the `name` or `value` are multiline, each entry will fit on a single line of text followed by a NewLine character.

A SPACE is exacly one UTF8 0x20 character.

The `=` EQUALS symbol is the UTF8 0x3D character.

The `:` COLON symbol is the UTF8 0x3A character.

### INDENT

Every line starts with indentation if there is any indentation.

For each level of indent, a pair of spaces is included. Every time a multiline structure is started, the content of that structure is is given an indent that one greater than the current indent level. Once the structure ends, the previous indent level is restored.

Specifically, a pair of spaces is two UTF8 SPACE characters (0x20).

One of the known and accepted downside to this specification is that a deeply nested source of data can be expensive storage-wise due to this indentation.

### ENTRY NAME

An entry name is a string (and only a string) that represents the name of the data contained in the entry.

An entry name may be of any length, but having an entry name greater than 80 characters makes for a difficult-to-read document.

The name may also be not-specified using the underscore symbol (`_`). This is common behavior with lists and arrays.

The name MUST NOT be marked as unknown (null) using the `?` character.

Names do not have to be unique. This is handled in a variety of ways by the programs that read and write SLONE documents. But, to be universal, it is recommended that multiple values of the same name be allowed (kept in the same order as the document.)

Example:

```slone1.0
#! SLONE 1.0
"foo" = _ "bar"
|
  "A really really really really really really really really really really really r"
  "eally really really really really really really really really really really real"
  "ly really really really really really really really really really long name"
| : int32 = "99"
_ : string = "xyz"
"target" : someArray = {
  _ : string = "a"
  _ : string = "b"
}
```

In this example, there are four entries. The first one is named "foo". The second one has a name longer than 80 characters. The third entry has no name.

The fourth entry is named "target" and it's value, in turn, has a pair of entries that do not have names.

### ENTRY TYPE

An entry type is a string (and only a string) that represents how the data contained in the entry should be interpreted. If included it is preceded by a colon and a space.

The string has a very strict set of limitations to it's content. Specifically:

- It is limited to 32 characters. (32 unicode code points).
- It may not contain WHITESPACE.
- It may not contain PUNCTUATION.

Essentially, it is limited to a short number of visible characters. It is a case-sensitive string.

There can also be no type string. For that, do not includ anything and do not use the preceding colon character.

The type string MUST NOT be marked as unknown (null) using the `?` character.

Otherwise, the content of the type is string is open. The SLONE specification DOES NOT specify what a type "means" in any sense. That is to be determined by the context of the applications reading and writing from the documents.

Example:

```slone1.0
#! SLONE 1.0
"foo" = "bar"
|
  "A really really really really really really really really really really really r"
  "eally really really really really really really really really really really real"
  "ly really really really really really really really really really long name"
| : int32 = "99"
_ : string = "xyz"
"target" : someArray = {
  _ : string = "a"
  _ : string = "b"
}
```

In this example, there are four entries. 

The first does not specify a type. Please note that there is no visible colon.

The second has a type of "int32". While this might imply that it is storing a signed 32-bit integer, the specification does not state this. The various programs reading/writing these documents must come to a common agreement independent of the SLONE specification.

The third has a type of "string".

The fourth entry has a type of "someArray". It's value is a document that has a pair of entries that have types of "string".

### ENTRY VALUE

An entry value is the data being stored. In can one of the following:

- A string (simple or long).
- A `?` to indicate that the value is unknown. In SQL parlance, this is often called a NULL.
- A subdocument, indicated started with a `{` symbol sequence at the start and a `}` symbol sequence at the end.

An entry value MUST NOT be nothing (`_`). If entry does not have value, it should simply be not included in the document.

## SIMPLE STRING ENCODING

A simple string is less than or equal to 80 characters long.

Please keep in mind that a character is a UTF8 code point, so it can easily be more than 80 _bytes_ long.

A simple string starts with a double-quote symbol ("). It then continues with each unicode character in the string, but inserting needed substitutions as found in the _String Escape Sequence Table_. It then ends with unescaped double-quote symbol (").

A simple string is always fully expressed on one line.

The character codes between 01 and 1F (hex) are otherwise known as "control characters". Strictly speaking, they are outside of the unicode character set. But as a practical matter most unicode libraries accept them. SLONE also accepts them. If a control character code is seen, encode it with it's replacement in the _String Escape Sequence Table_ below. If there is no matching entry, then encode it in the form of  `\0x??` where `??` is replaced with the two digit hex code.

Escapement does NOT "add characters". For example, the string `a\tb` is three characters long because the `\t` sequence counts as one character.

Examples:

- a form feed character would be encoded as `\f`.
- an ASCII bell character would be encoded as `\0x07`.
- a person's name such as Joe "Smiley" Smith would be encoded as `Joe \"Smiley\" Smith`.

SLONE does not support the NUL character code (00). So, `\0x00` is not legitimate.

#### String Escape Sequence Table

| sequence | decimal | hex | description |
| -------- | ------- | --- | ----------- |
| \\t      | 9       | 09  | tab (horizontal) |
| \\n      | 10      | 0A  | new line |
| \\v      | 11      | 0B  | vertical tab |
| \\f      | 12      | 0C  | form feed |
| \\r      | 13      | 0D  | carriage return |
| \\e      | 27      | 1B  | escape |
| \\"      | 34      | 22  | double quote |
| \\\\     | 92      | 5D  | backslash |

## BOTH RTL AND LTR LANGUAGE SUPPORT

A string's characters should be ordered by content rather than display. For example, Arabic is a language shown from right to left. So the phrase "They greeted each other" would be displayed:

<div dir="rtl">
?????? ???????????????? ?????????? ??????????
</div>

However, it would be encoded as 

- character 0 is ??
- character 1 is ??
- character 2 is ??
- character 3 is a space

and so on.

While the English version of the SLONE documention shows most examples in English in a LTR display, that is simply an
artifact of the display. The SLONE document is neither LTR or RTL. It is simply a series of characters in content order.

## LONG STRING ENCODING

A long string is more than 80 characters long.

A long string starts by ending the current line with a `|` sequence (followed by the NewLine that ends all lines.) On each following line, a portion of the string is indented on a line by itself as a simple string. The number of characters used is determined by the following ruleset (in order).

1. If the remainder of the string is 40 characters or less, use the entire remainder of the string. Ignore the remaining rules.
2. If, after the 40th character, there is a new line (`\\n`) sequence found, use every character up to and including the new line sequence. Ignore the remaining rules.
3. Use the next 80 characters of the remaining string.

After all of the string has been expressed, start a new line with a `|` sequence. This indicates the end of the string.

Why is rule #2 and #3 in above? For a long string, is not uncommon for text data "items" to be separated by new lines. As such, it would be nice if modifying an item that changes it's length would only trigger a change detection on one or a few SLONE lines rather than the entire remainder of the long string. Rules #2 and #3 make this more likely (though not guaranteed).

If the long string was being used for a value, a `|` will sit on it's own line at the end.

If the long string was being used for a name, the entry will continue with the space, equals symbol, etc.

Example:

```slone1.0
#! SLONE 1.0
"short" = "abc abc abc abc abc abc abc abc abc abc"
"long" = |
  "A really really really really really really really really really really really r"
  "eally really really really really really really really really really really real"
  "ly really really really really really really really really really long value"
|
|
  "A really really really really really really really really really really really r"
  "eally really really really really really really really really really really real"
  "ly really really really really really really really really really long name"
| = "foo"
"Fire and Ice by Robert Frost" = |
  "Some say the world will end in fire,\nSome say in ice.\n"
  "From what I???ve tasted of desire\nI hold with those who favor fire.\n"
  "But if it had to perish twice,\nI think I know enough of hate\n"
  "To say that for destruction ice\nIs also great\nAnd would suffice."
|
"two_lines_of_csv_numbers" = |
  "10001,10002,10003,10004,10005,10006,10007,10008,10009,10010,10011,10012,10013,10"
  "014,10015,10016,10017,10018,10019,10020,10021,10023,10024,10025,10026\n"
  "20001,20002,20003,20004,20005,20006,20007,20008,20009,20010,20011,20012,20013,20"
  "014,20015,20016,20017,20018,20019,20020,20021,20023,20024,20025,20026"
|
"a doc" = {
  "two_lines_of_csv_numbers" = |
    "10001,10002,10003,10004,10005,10006,10007,10008,10009,10010,10011,10012,10013,10"
    "014,10015,10016,10017,10018,10019,10020,10021,10023,10024,10025,10026\n"
    "20001,20002,20003,20004,20005,20006,20007,20008,20009,20010,20011,20012,20013,20"
    "014,20015,20016,20017,20018,20019,20020,20021,20023,20024,20025,20026"
  |  
}
```

## SUBDOCUMENTS

A value element can be a another document aka a subdocument.

And, the value of an entry in a subdocument can also be a subdocument, and so forth. In other words, SLONE's nature is recursive since subdocuments can subtend subdocuments without limit.

A subdocument is started with a `{` character sequence. Then each following line indented further and contains the data entries for the subdocument. After the document ends, place `}` on a line by itself at the restored indentation level.

Example:

```slone1.0
#! SLONE 1.0
"Larry" : person = {
  "main home" : building = {
    "mailing address" : address = {
      "street" : string_array =  {
        _ = "1234 Main St"
        _ = "Unit 3"
      }
      "postal code" : zip_code = "90210"
    }
  }
}
```

## ORDER OF ENTRIES

The order of the entries is **significant**. For example,

```slone1.0
#! SLONE 1.0
"name" : person_name = "John Smith"
"age" : int32 = "27"
```

is different than

```slone1.0
#! SLONE 1.0
"age" : int32 = "27"
"name" : person_name = "John Smith"
```

While the documents contain the same data, because the order is different, the SLONE specification considers them to be different documents.

The SLONE specification does **not** determine what the order should be. That is a matter to be determined by the application(s) using the documents.

Here are some possiblities:

1.  The applications could cooperatively decide the order of entries based on some external specification or schema. In fact, the SLONE specification allows for a `#%`-prefixed second line for documenting such a schema.

2.  The applications could cooperatively choose a default order, such as alphabetical order by entry name, for ordering the entries.

3.  The applications could honor the original ordering when reading the documents and endevour to keep that order when writing out any changes. Even then, the rule-set for this would need to be cooperatively agreed to.

## HANDLING NOTHING VS NULL VS EMPTY IN LIBRARIES

Most computer languages do not have a core way to differentiate _null_ (unknown) from _nothing_ (does not exist).

_One_ of the reasons that SLONE does not support nothing for a value is to make this limitation less important. That way either null or nothing, but never both, is allowed for each element in an entry. Specifically, "name" and "type" support nothing (`_` for "name", ommission for "type") but not null (`?`). And "value" supports null (`?`) but not nothing (`_`).

(The other reason it is not allowed is to remove a source of ambiguous serialization.)

Pretty much all languages fully support the concept of "empty". For example, in python, an empty array is `[]` and an empty integer is `0`.

```python
    # test.py
    #
    # a possible python example, where python's None keyword is used to
    # represent both null and nothing depending on context.
    #
    # So, for a name (key), the keyword "None" means nothing
    #
    x[""] = someValue   # empty; the name is an empty string
    x[None] = someValue # nothing; there is no name
    #                   # null is not allowed.
    #                   # (Philosophically, an "authoritative" unknown name makes no sense anyway.)
    #
    # So, for a value, the keyword "None" means null.
    #
    x["a"] = ""         # "a" entry is empty; the string has no content
    del x["b"]          # "b" entry is nothing; this is performed by ommission rather than assignment
    x["c"] = None       # "c" entry is null; the string's content is not known.
```
