Languages
=========

Use Case / Motivation
---------------------

The motivation behind this issue is that most of the times a user of the
PAPI only works with a single language. The operations will operate
primarily on the language the website currently displays. Secondary
languages only come into play as fallback languages for content. For
this 95%-case the API should be simplified.

If the current user of the website, for example, choose "de\_DE" as
their locale the API should return German content. The developer might
indicate a prioritized language list which defines the fallback if
content is not available in German, like:

- de\_DE
- de\_AT
- en\_GB
- …

This means content should only be fetched and returned in "en\_GB" if
the content is not available in neither "de\_DE", nor in "de\_AT".

If content is found the content will be returned in all available
languages right now. This imposes [additional programming
effort](https://doc.ez.no/display/DEVELOPER/Content+Rendering#ContentRendering-RenderingContentitems)
on the user of the API since they must select the correct language for
each field in a content structure. The API shall provide a simple
mechanism to just access the field / content in the requested language.

To reduce the overall overhead we should change the SPI to only return
single languages where it makes sense. The BC break in the SPI would be
OK, since there is no declared stability for the SPI nor are there
alternative implementations.

Realated:

- WIP-PR: <https://github.com/ezsystems/ezpublish-kernel/pull/1932>
- Netgen Site API: <https://github.com/netgen/ezplatform-site-api>

### Implementation Notes

- Check if PAPI modifications can be implemented as a pure decorator
- Check if value object constraints can be implemented as decorators,
  too

Decorator Stacking / Configuration
----------------------------------

Besides the simplification of the language API there are other
requirements for the PAPI which all can make use of extracting the
different concerns beside storing the content into decorators.

![PAPI Decorators](decorator.svg)

Right now there already is a decorator to implement Signals on top of
the PAPI.

There are requirements (importing, …) where it makes sense to use the
PAPI without permissions, which are currently built right into the
storage layer implementation. Extracting a permission decorator would
make sense to enable usage without any permission checks.

We could also extract a caching layer which could cache the PAPI
responses where it makes sense to improve the responsiveness of the
PAPI.

Since there is only one sensible order for the decorators, as
illustrated in the graphic above, there is probably no need to make it
easily possible to re-order those layers through configuration. Each
layer should be exposed in its entirety through the Dependency Injection
Container (DIC) with a common postfix.

The current service identifiers, for example `ezpublish.api.repository`
should point by default as aliases to languages layer
(`ezpublish.api.repository.languages`). We might want to make the layer
configurable where the alias points to.

To implement this there are a couple of tasks to execute:

-   Extract permission logic into a decorator
-   Extract caching into a decorator
-   Implement aliasing for PAPI services

There are a couple of benefits behind this approach:

- **Extensibility**

  If there are new concerns it gets more obvious that those should be
  added as another decorator and not be implemented as changes to the
  existing implementation.

- **Simplicity**

  The code of decorators working on just one concern (permissions) is
  usually a lot lot simpler then an implementation which implements
  persistence, permissions and caching in the same method. In practice
  this means that a lot of private methods like `internalCreateRole`
  can be moved into their own classes.

- **Testability**

  If one decorator only fulfils one single concern we can test those
  decorators more easily which will lead to additional stability.

- **Composability**

  Being able to compose and use your own decorator stack enables us to
  implement new use cases like importers in a far more sensible way
  then right now.

There is also a drawback to this approach:

- **Layering**

  More layers can make it harder to debug the code. Each layer gets
  less complex for itself though which makes them less error prone.
  Users will seldomly have to debug every layer. It can still make the
  overall system harder to understand.

Language Selection Logic
------------------------

We suggest to implement a
[LanguageResolver](/ezsystems/ezpublish-kernel/tree/master/eZ/Publish/Core/Repository/SiteAccessAware/Helper/LanguageResolver.php)
which is initialized by the developer with a prioritized language list.
It should implement all necessary methods to resolve language
information for the lower level APIs, just like the `getLanguages()`
method in the prototype.

The `LanguageResolver` should also be used to fetch the correct language
version for names and fields in the mapper classes. There is code
duplication in that regard right now in the prototype.

Every affected service decorator should receive a `LanguageResolver`
instance together with the aggregate like it is already implemented in
the prototype.

Returned Content Objects
------------------------

The current prototype violates the Liskov Substitution Principle by
returning different values then the decorated classes would return. We **must
not** return different objects then the originals in a decorator:

The returned values must fulfill the full interface, which basically
means containing all public properties, which the already existing
classes contain. Since we decorate an API it must be possible to use it
exactly like the earlier API.

We should extend the existing value objects with more convinience methods. As
outlined below the most important translated properties are `$name` and
`$description`. We can add `getName()` and `getDescription()` methods to all
affected value objects.

These methods can either receive a parameter or return the single string, if
only one is available. They will be accessible in templates using
`{object.name}`.

SPI Analysis
------------

List of values containing translated properties (`$name`, `$description`, …):

- [Content/Field](/ezsystems/ezpublish-kernel/tree/master/eZ/Publish/SPI/Persistence/Content/Field.php)
- [Content/ObjectState](/ezsystems/ezpublish-kernel/tree/master/eZ/Publish/SPI/Persistence/Content/ObjectState.php)
- [Content/ObjectState/Group](/ezsystems/ezpublish-kernel/tree/master/eZ/Publish/SPI/Persistence/Content/ObjectState/Group.php)
- [Content/Type](/ezsystems/ezpublish-kernel/tree/master/eZ/Publish/SPI/Persistence/Content/Type.php)
- [Content/Type/FieldDefinition](/ezsystems/ezpublish-kernel/tree/master/eZ/Publish/SPI/Persistence/Content/Type/FieldDefinition.php)
- [Content/Type/Group](/ezsystems/ezpublish-kernel/tree/master/eZ/Publish/SPI/Persistence/Content/Type/Group.php)
- [Content/UrlAlias](/ezsystems/ezpublish-kernel/tree/master/eZ/Publish/SPI/Persistence/Content/UrlAlias.php)
- [Content/VersionInfo](/ezsystems/ezpublish-kernel/tree/master/eZ/Publish/SPI/Persistence/Content/VersionInfo.php)
- [User/Role](/ezsystems/ezpublish-kernel/tree/master/eZ/Publish/SPI/Persistence/User/Role.php)

Methods returning values containing translations:

    Content\ObjectState\Handler

    ::loadGroup($groupId): \eZ\Publish\SPI\Persistence\Content\ObjectState\Group
    ::loadGroupByIdentifier($identifier): \eZ\Publish\SPI\Persistence\Content\ObjectState\Group
    ::loadAllGroups($offset = 0, $limit = -1): \eZ\Publish\SPI\Persistence\Content\ObjectState\Group[]
    ::loadObjectStates($groupId): \eZ\Publish\SPI\Persistence\Content\ObjectState[]
    ::load($stateId): \eZ\Publish\SPI\Persistence\Content\ObjectState
    ::loadByIdentifier($identifier, $groupId): \eZ\Publish\SPI\Persistence\Content\ObjectState
    ::getContentState($contentId, $stateGroupId): \eZ\Publish\SPI\Persistence\Content\ObjectState

    Content\UrlAlias\Handler

    ::listGlobalURLAliases($languageCode = null, $offset = 0, $limit = -1): \eZ\Publish\SPI\Persistence\Content\UrlAlias[]
    ::listURLAliasesForLocation($locationId, $custom = false): \eZ\Publish\SPI\Persistence\Content\UrlAlias[]
    ::lookup($url): \eZ\Publish\SPI\Persistence\Content\UrlAlias
    ::loadUrlAlias($id): \eZ\Publish\SPI\Persistence\Content\UrlAlias

    Content\Handler

    ::load($id, $version, array $translations = null): \eZ\Publish\SPI\Persistence\Content
    ::loadVersionInfo($contentId, $versionNo): \eZ\Publish\SPI\Persistence\Content\VersionInfo
    ::loadDraftsForUser($userId): \eZ\Publish\SPI\Persistence\Content\VersionInfo[]
    ::listVersions($contentId, $status = null, $limit = -1): \eZ\Publish\SPI\Persistence\Content\VersionInfo[]

    Content\Type\Handler

    ::loadGroup($groupId): \eZ\Publish\SPI\Persistence\Content\Type\Group
    ::loadGroupByIdentifier($identifier): \eZ\Publish\SPI\Persistence\Content\Type\Group
    ::loadAllGroups(): \eZ\Publish\SPI\Persistence\Content\Type\Group[]
    ::loadContentTypes($groupId, $status = Type::STATUS_DEFINED): \eZ\Publish\SPI\Persistence\Content\Type[]
    ::load($contentTypeId, $status = Type::STATUS_DEFINED): \eZ\Publish\SPI\Persistence\Content\Type
    ::loadByIdentifier($identifier): \eZ\Publish\SPI\Persistence\Content\Type
    ::loadByRemoteId($remoteId): \eZ\Publish\SPI\Persistence\Content\Type
    ::getFieldDefinition($id, $status): \eZ\Publish\SPI\Persistence\Content\Type\FieldDefinition

    User\Handler

    ::loadRole($roleId, $status = Role::STATUS_DEFINED): \eZ\Publish\SPI\Persistence\User\Role
    ::loadRoleByIdentifier($identifier, $status = Role::STATUS_DEFINED): \eZ\Publish\SPI\Persistence\User\Role
    ::loadRoleDraftByRoleId($roleId): \eZ\Publish\SPI\Persistence\User\Role
    ::loadRoles(): \eZ\Publish\SPI\Persistence\User\Role[]

Only one method (`Content\Handler::load`) of these already has a
`$translations` parameter. Each of these methods could receive a
`$translations` parameter (defaulting to `null`) to limit the fetched
translations to a defined list or a single translation.

### Migrating Requested Languages

We generally want to request less data (in less languages) from the SPI.
Usually the content should only be returned in one single language.

The concept we want to transport is the following:

* Prioritized language (example: `ger_DE`)
* Secondary language (example: `fra_FR`)
* …

* Configured main language (example: `eng_GB')

Depending on the languages available for a given content node the resolving
should work like:

* If content is available in `ger_DE`, only return content in `ger_DE`.

* If content is not available in `ger_DE` but in `fra_FR`, return content only
  in `fra_FR`.

* If content is available in neither `ger_DE` nor `fra_FR` return content in
  `eng_GB` which should always be available, otherwise return "Not Found".

The SPI currently allows to pass an `array $translations` argument to one
single method (`Content\Handler::load()`). The mentioned language concept is
hard to express using an array, especially in a backwards compatibile way.

Right now the array implies to fetch all languages which are given in the
array. In most cases people just want to fetch one single language and define
fallbacks to other languages. To make it easy to distinguish between both
concepts we will define two new value objects:

    class LangaugesFetchAll extends LanguageList implements \Traversable {
        /**
         * Unordered list of languages to fetch
         *
         * @var string[]
         */
        public $languages = [];

        /**
         * Optionally specify custom main language
         *
         * @var string
         */
        public $mainLanguage = null;
    }

    class PrioritizedLanguage extends LanguageList implements \Traversable {
        /**
         * Ordered language list
         *
         * @var string[]
         */
        public $languages = [];

        /**
         * Optionally specify custom main language
         *
         * @var string
         */
        public $mainLanguage = null;
    }

In version one the array will be converted into a `LangaugesFetchAll` object to
reproduce the current behaviour. We will already document that this behaviour
will change in version 2 and you *should* pass the correct object already.
(Emit a deprecated notice?)

In version 2 we will convert incoming arrays into a `PrioritizedLanguage`
object to make the 99% case easier to use. The conversion is implemented in the
PAPI.

We therefor must remove the `array` type hint in the SPI and PAPI. With PHP 7.1
we might be able to add a [`iterable`](https://secure.php.net/manual/en/migration71.new-features.php#migration71.new-features.iterable-pseudo-type)
type hint again since the `LanguageList` object shall fulfill the
`Traversable` interface. In the SPI we can allready type hint on
`LanguageList`.

The SPI should the implement the logic to either fetch only one language
(`PrioritizedLanguage`), a couple of defined languages (`array`) or all
available languages (`LangaugesFetchAll`).

Sooner or later all methods mentioned above might be migrated and extended with
such a `$translations` parameter. We should prioritize the methods where we
expect the largest performance benefits for the overall system. For entities
where (with the current storage) all languages are always fetched anyways
(mostly `$name` and `$description`) we might postpone it for another while or
filter the languages on client side.

Language Decorator BC Break
---------------------------

Using the Language Decorator results in a different behaviour of the PAPI.
Usually only one single language version will be returned instead of all
languages like it is right now. This is a behavioural change which is not a
formal BC break since each API and every value object still maintains
compatibility and just contain less data.

If we assume that developers are iterating over language versions right now or
requesting defined languages expecting that this language might not be set, no
code should break but it might be that less content is shown in some cases.

If code right now assumes that the main language is always set and we stop
fetching the main language because another prioritized language is available
**this** might break some code.

As this document suggest using the language decorator will be optional so
people could still be using the undecorated implementation in this case.

