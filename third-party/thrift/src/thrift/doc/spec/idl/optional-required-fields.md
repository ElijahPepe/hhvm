# Optional/Required Fields

<!-- https://www.internalfb.com/intern/wiki/Thrift/Thrift_Guide/IDL/optional-required-fields/?noredirect -->

#### Thrift fields can be specified as "optional" or "required".  If neither of these specifiers is used, the default behavior is a mix of the two.

## Fields marked `optional`

* Will only be written when serializing a structure if the field is marked as "set" in the structure.
* In C++, the above rule means that you must set `myStruct.__isset.myField = true` to indicate that the value is "set" and must be serialized.
* Will be left unchanged when deserializing a structure if the field isn't present in the serialized data.

## Fields marked `required`

WARNING: Do not use in the new code. They are deprecated since ~2018. See this workplace Post from Stepan in Thrift FYI: https://fburl.com/thrift/required/nomore


* Will always be written when serializing a structure.
* ~~An exception will be thrown if the field is not present in the serialized data when deserializing.~~ ([not enforced](https://fb.workplace.com/groups/1730279463893632/permalink/2177428059178768/))
   * ~~Corollary: If a new `required` field is introduced, deserializing any existing data written prior to its introduction will **fail** with exceptions.~~
* In C++, the `__isset` member is only written for fields *not* marked required.

## Fields that don't have an `optional`/`required` specifier

* These fields will be **assumed present**.
* Will always be written when serializing a structure.
* Will be left unchanged when deserializing a structure if the field isn't present in the serialized data.

For non-primitive types:In languages where fields may be set to null (`thrift-py` and PHP, but also object types in Java), fields are considered absent if they are set to `null` (or `None`). These languages essentially treat every field as 'optional' if not marked otherwise.

NOTE: thrift-py3 behaves more like thrift-cpp2, in which all unqualified fields will have a default value, no matter primitive or not.


For primitive types:Fields must be set to `optional` to be nullable in PHP.

NOTE: In thrift-py, [**all** fields are nullable](https://www.internalfb.com/intern/wiki/Thrift_in_Python/Migrate_from_thrift-py/Types/#unqualified-fields-in-th).


#### Terse Writes

Some of the space savings of `optional` fields can be obtained with `default` storage (not `optional`, not `required`) by passing the `terse_writes` option to the compiler. `terse_writes` will suppress serializing fields where the values are the same as their present default value, when doing that comparison is cheap (e.g. i32/i64, empty strings/list/map). This will lead to smaller output and lower deserialization cost - particularly when fields are sparsely used.

## Practical Recommendations

In practice, it is often best to leave a field as unspecified (neither '`required`' nor '`optional`') unless an object is expected to be changed and re-serialized.  This is because both required and optional are susceptible to errors of omission (i.e forgetfulness) on the part of developers.

* If a developer is initializing a thrift object and forgets to set a required field, this will cause an exception to be thrown.  This may lead to crashes and other server errors.
* If a developer is initializing a thrift object in C++ and forgets to set the `__isset.<field>` corresponding to a field they are using, the data for the field won't even be sent across the wire.  Similarly, if the receiver uses the value of a field without checking `__isset.<field>`, this may lead to strange behavior.

For most types of fields, there is a reasonable default value that can be used in place of checking for `__isset.<field>`.  Lists and other data structures can be empty. Integers can often be zero.

If structures are stored in a serialized form and modified over time, it is safer to be explicit and use `optional` fields.

## Hazards

* If the default value for a tersely written field is changed, the result of deserializing a given string will change. e.g: if `1: i32 initial_gold = 100` is changed to `1: i32 initial_gold = 200`, any serialized structure which previously stored the value `100` will now appear to store the value `200`.
* Deserializing does not un-set `optional` fields which are absent, so deserializing into a single target object multiple times will lead to leftover `optional` fields from prior deserializations.
* Semantic differences across languages can lead to unexpected behavior. In these circumstances, the Thrift compiler generally implements a reasonable compromise, but it can still lead to surprising outcomes, e.g.: Maps serialized from PHP do not preserve order of keys/values.
* Merging structures with Thrift's `merge()` function works well with `optional` fields, but it is not recommended for use with `default` fields:
   * If `terse_writes` is enabled, default values *never* merge into (possibly overwriting) non-default values.
   * If `terse_writes` is not enabled, default values *always* merge into (possibly overwriting) non-default values.
