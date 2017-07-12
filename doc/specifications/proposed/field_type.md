Field Type Creation Simplification
==================================

Overview
--------

First provide a couple of default implementations on the
`eZ\Publish\Core\FieldType\FieldType` class. Omit all optional methods
from the
[tutorial](https://doc.ez.no/display/EZP/eZ+Publish+5+Field+Type+Tutorial).

Include additional "follow up" sections in the tutorial describing which
methods should be overwritten in which use cases, like:

- Verify value object integrity
- Add configurable validations
- Add field type settings
- Implement a custom value object
  - Implement custom hashing
- Enhance the indexing of the field type
- Store or retrieve external data

This way the developer quickly gets to a functional working result which
then can be extended. If a new developer has to touch too many classes
or other code artifacts (configuration …) it will be really hard for
them to debug all those common subtile issues (naming, …).

There are additional methods which can be implemented by default, even
it reduces the default functionality. But it will allow external
developers to come up with an extensible working state a lot faster:

    protected function checkValueStructure(CoreValue $value)
    {
        // Just do not check anything by default
    }

    public function getEmptyValue()
    {
        return new {$this->valueClass}();
    }

    public function getName(SPIValue $value)
    {
        // @TODO: Check if __toString is actually defined and throw an
        // exception otherwise, because it is not part of the SPIValue
        // interface – or add the method to the interface.
        return (string) $value;
    }

    protected function getSortInfo(CoreValue $value)
    {
        return $this->getName($value);
    }

    public function fromHash($hash)
    {
        if ($hash === null) {
            return $this->getEmptyValue();
        }

        // The default constructor at least works for the top level objects.
        // For more complex values a manual conversion is necessary.
        return new {$this->valueClass}($hash);
    }

    public function toHash(SPIValue $value)
    {
        // Simplest way to ensure a deep structure is cloned and converted into
        // scalars and has maps.
        return json_decode(json_encode($value), true);
    }

Especially the hashing functions define assumptions about the vlaue
object which might or might not be true. They expect the default
constructor which assignes all array keys as properties on the value
object. This must be documented when documenting custom value objects
and their hashing methods. The hash methods can still be overwritten…

This leaves only two methods to implement:

- `getFieldTypeIdentifier()`

  Obviously necessary.

- `createValueFromInput($inputValue)`

  This requires explanation how this integrates with the edit template
  (which is missing entirely in the tuorial) and where this data
  comes from. Web developers tend to try stuff in the web interface
  and the connecting dots are missing to be able to do this.

  We *could* also make this method optional by adding a
  `::fromInput()` method to the value object and providing default
  implementations for the pre-defined value classes (see below).

### The Type Class

Right now it is quite tedious to develop a new field type because of
various issues:

- Development Experience: Watching fatal errors and SF HTML error
  responses in AJAX requests in the browser debug bar. (Probably
  resolved in next version)

- Tutorial missing edit and settings templates – the latter causes
  Internal Server Errors and a broken view.

  There is a pull request to make the settings template optional again
  and thus it should be documented in the relevant tutorial section
  (about settings).

  The edit template seems necessary to properly try out the field type
  and should be documented.

- Field type indentifier MUST NOT contain dashes so that Twig template
  blocks work. We should probably document (and verify or normalize)
  them as `/^[a-zA-Z0-9_]+$/`.

  Internal normalization probably isn't an option since a developer
  would have to use different identifiers in Twig template blocks then
  the one specified in the field type implementation. We could assert
  on a proper identifier in the test case. The documentation suggests
  to use `.` as a separator in the name – we should ensure this works
  fine with Twig since I guess this could be considered an operator by
  Twig, too.

We should probably document the field type base test class
`eZ\Publish\Core\FieldType\Tests\FieldTypeTest` which allows to test
drive a new field type, making sure all the trivial errors will not
occur (hidden in AJAX responses).

To make it even easier we can again provide sensible defaults for all
methods which are optional to implement:

- `getValidatorConfigurationSchemaExpectation`
- `getSettingsSchemaExpectation`

### The Value Class

The value class must be defined by the developer, while they can use a
default one. We should define some based on common types, like:

    class NumericValue extends Value
    {
        public $value = 0;

        public function __toString(): string
        {
            return (string) $this->value;
        }
    }

This value can then just be referenced to the type constructor in the
dependency injection container so that the `$valueClass` property is set
properly in the type.

### The Converter

Together with the hash related methods most of a converter can also be
implemented with sane defaults:

    class Converter implements ConverterInterface
    {
        protected $indexColumn;

        public function __construct($indexColumn = 'sort_key_string')
        {
            $this->indexColumn = $indexColumn;
        }

        public function toStorageValue(FieldValue $value, StorageFieldValue $storageFieldValue)
        {
            $storageFieldValue->dataText = json_encode($value->data);
            $storageFieldValue->sortKeyString = $value->sortKey;
            $storageFieldValue->sortKeyInt = $value->sortKey;
        }

        public function toFieldValue(StorageFieldValue $value, FieldValue $fieldValue)
        {
            $fieldValue->data = json_decode($value->dataText, true) ?: [];
            $fieldValue->sortKey = $value->sortKeyInt ?? $value->sortKeyString;
        }

        public function toStorageFieldDefinition(FieldDefinition $fieldDef, StorageFieldDefinition $storageDef)
        {
        }

        public function toFieldDefinition(StorageFieldDefinition $storageDef, FieldDefinition $fieldDef)
        {
        }

        public function getIndexColumn()
        {
            return $this->indexColumn;
        }
    }

This implementation allows to configure the mandatory definition of the
index column and provides a mostly sane defintionition for everything
else. Thus we do not need to implement anything either, but just
configure the converter in the dependency injection container.

### The Templates

The last missing bit are the templates a developer has to specify. Since
the settings template will again be made optional, this leaves two
templates to be defined:

- The view
- The edit template

#### Template Generation

Since those templates follow defined structures we should probably even
write a generator for these. The generator could create a full bundle
(people could then manually merge with an existing bundle) based on the
specification of:

- An identifier
- The value class to use

It then has to generate:

- The type extension (both methods)
- The dependency injection container configuration
- The templates
- The configuration and DIC extension to register the templates

### Testing The Field Type

It would be really helpful to include instructions on how to use and
test a field type in the user interface since this is what most
developers will want to do. This is also essential to make sure the
templates integrate properly.

Custom Storage
--------------

With the new `eZ\Publish\SPI\FieldType\GatewayBasedStorage` together
with the `eZ\Publish\SPI\Tests\FieldType\BaseIntegrationTest` it should
be simple enough for developers to develop a custom field storage
implementation.

A dedicated tutorial section should guide the developer through setting
up the test, running it and developing the field storage this way. The
documentation on the test case should be revisited for this, and the
tutorial should probably explicitely mention the `stop-on-failure` flag,
which is really useful for developing new implementations for against
existing tests:

    bin/phpunit --stop-on-failure path/to/my/FieldTypeIntegrationTest.php

This should be mentioned together with registering the new custom
storage, which is also is yet undocumented. To make sure this also works
we can replace the `getCustomHandler()` method in the
`BaseIntegrationTest` with access to the actual Dependency Injection
Container to make sure this integration also works. Maybe also add a
test explicitely testing that the returned handler is correct and
providing sensible failure output with helpful text or links.

As far as I see it we can provide a defualt implementation in the
`GatewayBasedStorage` for `getContentTypeId()` since it only affects
kernel related tables:

    /**
     * Retrieve the ContentType ID for the given $field.
     *
     * @param \eZ\Publish\SPI\Persistence\Content\Field $field
     *
     * @return int
     */
    public function getContentTypeId(Field $field)
    {
        $query = $this->connection->createQueryBuilder();
        $query
            ->select($this->connection->quoteIdentifier('contentclass_id'))
            ->from($this->connection->quoteIdentifier('ezcontentclass_attribute'))
            ->where(
                $query->expr()->eq('id', ':fieldDefinitionId')
            )
            ->setParameter(':fieldDefinitionId', $field->fieldDefinitionId);

        $statement = $query->execute();

        $row = $statement->fetch(\PDO::FETCH_ASSOC);

        if ($row === false) {
            throw new RuntimeException(
                sprintf(
                    'Content Type ID cannot be retrieved based on the field definition ID "%s"',
                    $field->fieldDefinitionId
                )
            );
        }

        return intval($row['contentclass_id']);
    }

This does not solve the set up issues:

- A non existing table must be created
- An external database / web service might need to be reset during
  tests

But since the schema management inside eZ Publish is yet to be defined
we can skip on this for now and document this as a potential issue in
the repective tutorial section. People can still overwrite
`setupBeforeClass` to implement this manually in their integration test.