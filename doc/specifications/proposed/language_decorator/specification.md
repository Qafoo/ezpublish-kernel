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

-   de\_DE
-   de\_AT
-   en\_GB
-   …

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

-   WIP-PR: <https://github.com/ezsystems/ezpublish-kernel/pull/1932>
-   Netgen Site API: <https://github.com/netgen/ezplatform-site-api>

### Implementation Notes

-   Check if PAPI modifications can be implemented as a pure decorator
-   Check if value object constraints can be implemented as decorators,
    too

TODO
----

-   \[ \] Analyze ways to implement prioritized languages
-   \[ \] Analyze decorator omitting the requirement to specify
    languages at all
-   \[ \] Check which methods are missing language parameters in SPI
-   \[ \] Check what can be done on SPI and value object level to return
    only one single language for content

Decorator Stacking / Configuration
----------------------------------

Besides the simplification of the language API there are other
requirements for the PAPI which all can make use of extracting the
different concerns beside storing the content into decorators.

![image](decorator.svg)

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

-   **Extensibility**

    If there are new concerns it gets more obvious that those should be
    added as another decorator and not be implemented as changes to the
    existing implementation.

-   **Simplicity**

    The code of decorators working on just one concern (permissions) is
    usually a lot lot simpler then an implementation which implements
    persistence, permissions and caching in the same method. In practice
    this means that a lot of private methods like `internalCreateRole`
    can be moved into their own classes.

-   **Testability**

    If one decorator only fulfils one single concern we can test those
    decorators more easily which will lead to additional stability.

-   **Composability**

    Being able to compose and use your own decorator stack enables us to
    implement new use cases like importers in a far more sensible way
    then right now.

There is also a drawback to this approach:

-   **Layering**

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
returning different values then the decorated classes would return.

The returned values must fulfill the full interface, which basically
means containing all public properties, which the already existing
classes contain. Since we decorate an API it must be possible to use it
exactly like the earlier API.

We suggest to use value objects implementing the following principle:

    trait ValueDecorator
    {
        protected $aggregate;

        public function __get($property)
        {
            return $this->aggregate->$property;
        }

        public function __set($property, $value)
        {
            return $this->aggregate->$property = $value;
        }

        // @TODO: Same for __isset, __clone, …

        public function getAggregate()
        {
            return $this->aggregate;
        }
    }

This works for all value objects with read-only properties which are
usually the value objects returned by the PAPI. This decoration approach
with the automatic dispatching does not work with value objects with
public properties (update and create structs).

We can then implement concrete value object extensions, like:

    class ContentInfo extends OriginalContentInfo
    {
        uses ValueDecorator;

        protected $languageResolver;

        public function __construct(
            OriginalContentInfo $aggregate,
            LanguageResolver $languageResolver
        ) {
            $this->aggregate = $aggregate;
            $this->languageResolver = $languageResolver;
        }

        public function getName()
        {
            return $this->languageResolver->resolveName($this->aggregate);
        }

        // @TODO: More convinience accessors…
    }

This way the extended `ContentInfo` can work exactly like the original
one and add additional methods maintaining full compatibility.

Optionally we could also extend the `__get()` method to return computed
values on property access. While this leads to a clean API from a
developers perspective this usually also leads to really cluttered
`__get()` methods.

Risks:

-   There are some value objects which are returned which contain public
    properties while they probably shouldn't, like the `SearchResult`.
    As long as all values affected by translations properly use
    protected properties to simulate read-only access this will work.
    Problematic classes could be:

        Content/SectionStruct.php
        21:    public $identifier;
        28:    public $name;

        User/Limitation.php
        52:    public $limitationValues = array();

    The only class I found which may really be affected by the mentioned
    problem is the `SectionStruct`\_\_. But since the `$name` is
    documented as a string it is probably not translated and we are good
    to go.

-   [Traits](https://qafoo.com/blog/072_utilize_dynamic_dispatch.html)
    are generally considered bad practice. To work around a missing
    language feature we can assume this is fine in this case. We are
    doing the same already with the `ValueObject` base class, which also
    could be a trait.

An important benefit of this approach is that it will work even if other
layers also return different values. If we re-construct the object into
our custom implementation without dispatching anything to the original
implementation those potential additional information would be lost.

@TODO: Loading / wrappings fields? What are sensible helpers here
besides what already exists?

SPI Analysis
------------

@TODO: Where are language parameters missing? What can be done to
optimize loading of data?
