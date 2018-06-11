# Module Unification Namespaces

- Start Date: 2018-03-02
- RFC PR:
- Ember Issue:

# Module Unification with Ember Addons

## Summary

Ember addons can use the module unification filesystem layout (i.e. an addon
with a `src/` directory) in place of the legacy `app/` or `addon/` folders. The
`src/` folder encapsulates a module unification "package", and Ember
applications can access components, helpers, and services from that package.
Accessing a component from a package requires the `{{use}}` helper.

## Motivation

[RFC #143](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md)
introduced a new filesystem layout for Ember applications we've called "Module
Unification". The RFC described semantics for "namespaces" and the concrete
example of components from another namespace or within an addon namespace being
invoked as `{{my-namespace::my-component}}` in the section [Addon
modules](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md#addon-modules).

The syntax of `::` allowed us to specify an **explicit** namespace for an
invocation. The lookup would only occur within the prefixed namespace. The RFC
described this explicit behavior.

Around the same time the Ember community was working on the Module Unification
design NPM announced a new feature: Scoped packages. Scoped
packages have a composite name in the format `${scope}/${packageName}`.
For example:

```js
// Import the default export of the moment package.
// Often this is node_modules/moment/index.js
import moment from 'moment';

// Import the default export of a scoped package.
// For example node_modules/@angular/router/index.js
import { RouterModule } from '@angular/router';
```

**The addition of `@npmscope/` as a valid part of package names is in conflict with
the original design of Module Unification namespace invocation syntax.** For
example here are several invocation examples which would be valid presuming
Module Unifications' original namespace proposal, NPM scopes, and angle bracket
invocation syntax:

```hbs
{{#@npmscope/package-name::component-name}}
  Some content.
{{/@npmscope/package-name::component-name}}

{{#@npmscope/package-name}}
  Some content. Would invoke an implict component "main" in the package.
{{/@npmscope/package-name}}

<@Npmscope/PackageName::Name>
  Some content.
</@Npmscope/PackageName::Name>

<@Npmscope/PackageName>
  Some content.
</@Npmscope/PackageName>
```

These examples are in violation of several design constraints:

- **Use of `@` when invoking a component from a scoped package should be avoided.**
  Recent versions of Ember and Ember RFCs have introduced the sigil `@` to
  templates with a specific meaning. `{{@someArg}}` would read an argument pass to
  a component, and arguments are themselves passed with the `@` prefix. For
  example `<MyComponent @someArg={{someObj}}>`. Arguments can also be invoked
  (`{{@someArg}}` per [RFC 226](https://github.com/emberjs/rfcs/blob/master/text/0226-named-blocks.md#unified-renderable-syntax)) or be created from a block (`<@someArg= >` per [RFC
  317](https://github.com/emberjs/rfcs/blob/named-block-syntax/text/317-named-block-syntax.md#summary)). As the meaning of `@` is very specific and important in Ember templates,
  we should not create visual ambiguity and confusion by using it for component
  lookup.
- **Use of `/` when invoking a component from a scoped package should be avoided.**
  When using angle brackets multiple closing tags are very difficult to
  disambiguate from multiple closing tags. We’ve already discouraged use of `/` in
  component names by disallowing in Module Unification layout apps. An example for
  visual comparison:
    - `</@Npmscope/PackageName::Name>`
    - `</@Npmscope></PackageName::Name>`
- **NPM package names should be treated as literal strings.** The issues here arose
  because we made an assumption that NPM package names would be a single word or
  a dasherized series of words. That was incorrect. We should instead assume that
  new features may be added to NPM in the future. Our invocation solution should
  treat NPM package names as strings where any characters would be valid.

I suggest two further constraints:

- **Use of scopes should not be discouraged.** It could be tempting to accept a design
  which works well for “normal” npm packages but requires more effort for users of
  scoped packages. By accepting one of these options we would a) make Ember more
  difficult to learn since you must learn two different solutions for the same
  problem of addon component invocation and b) by making un-scoped packages more
  ergonomic we would discourage addon authors from using the scoped packages
  feature. In practice, scoped packages seem like a good feature for authors to
  consider and in some corporate settings use of them may not be optional.
- **Resolution rules should be consistent across parts of Ember.** For example
  normalization rules for package names in templates and in service injections
  should be the same. Given that a developer is using Ember Data they should not
  refer to it in some places as EmberData and others as `ember-data`.

This RFC describes a new helper `{{use}}` for explicit importing of a component or
helper from a package. This replaces the `::` syntax.

- **Introduce a `{{use}}` helper which is a logical subset of JavaScript’s
  `import`.** This will be used to make all imports explicit and declarative. It
  will also have the side effect of making isolation of the `{{component}}` helper
  to a narrow scope of components inherit in that helper's design. More on this
  below.
- **Drop the single line package+name invocation syntax (`::`)**. This syntax is the root
  cause of our issues. Later in this document I present two alternatives that
  strive to maintain it, but I remain unconvinced of their value. I'm making an
  assumption that for most addons they are used only in one or two templates
  making this a low-burden. The most notable exceptions to that assumption may be
  a) truthy helpers and b) translation helpers.
- **Drop implicit "main" rules from module unification.** Because implicit main is a
  *resolution rule* and not an invocation syntax rule, there is no way for us to
  know if `<Foo />` is an app component or a package name + main without
  significant analysis of the program (with a build-time resolver). As a runtime
  resolver is sure to be part of common Ember usage for a long while, it seems
  beneficial to remove this ambiguity. Additionally, that whole-program knowledge
  is needed to disambiguate this case may make it more difficult to write tooling
  for templates.

This RFC makes the additional suggestion that addons have an **implicit**
namespace. This makes writing an addon much like writing an app. Things in the
`src/` directory are invoked by their names, and things in other namespaces
require `{{use}}`.

## Detailed design

Only **services**, **components**, and **helpers** from an addon can be resolved
by an application.

To help the RFC be approachable I'll describe behaviors using plausible
filenames. The actual implementation is based on module unification specifiers,
so filenames used below may have alternative forms that are valid.

An addon name of `gadget` or `gadget-lib` is used in these examples.

Additionally I'll be using `{{` or `<` to invoke components in examples. It is implied
that `(component` would work with the string version of the `{{` invocation.

### The `{{use}}` helper

The `{{use}}` helper is a keyword in Glimmer templates with unique syntax.
Its functionality is a subset of the ES `import` keyword, but the syntax is
different.

We can use similar terminology as the [ES `import`
spec](https://www.ecma-international.org/ecma-262/6.0/#sec-imports) to describe
`{{use}}`:

* *UseDeclaration*:
  * `{{` `use` ImportsList FromClause `}}`
* *ImportsList*:
  * *ImportSpecifier*
  * *ImportsList*, *ImportSpecifier*
* *ImportSpecifier*:
  * *ImportedBinding*
  * *IdentifierName* `as` *ImportedBinding*
* *FromClause*:
  * `from` *ModuleSpecifier*
* *ModuleSpecifier*
  *StringLiteral*
* *ImportedBinding*
  *BindingIdentifier*


### Explicit namespaces

#### Explicit namespaces in templates

To invoke a specific item from an addon, the addon's namespace must be passed as
a prefix to the invocation. For example an invocation of `{{gadgets::try-me}}`
from an application would attempt each of the following ordered list:

* With addon namespace:  `gadgets/src/ui/components/try-me/template.hbs`
* With addon namespace:   `gadgets/src/ui/components/try-me/component.js`
* An invocation with `::` should never be treated as a property. Throw.

Since the presence of `::` makes it unambiguous that a component or helper is
being invoked, the "dash rule" is loosened. For example an invocation of
`{{gadgets::tryme}}` from an application would attempt each of the following
ordered list:

* With addon namespace:  `gadgets/src/ui/components/tryme/template.hbs`
* With addon namespace:  `gadgets/src/ui/components/tryme/component.js`
* An invocation with `::` should never be treated as a property. Throw.

This permits Ember addons to have an in-template API that is isolated from
conflict but not overly verbose. For example `{{animated::explode}}` would be
valid as a component or helper, and within the same app you could have
`{{mind::explode}}` available.

#### Explicit namespaces in services

Services likewise specify namespaces with a prefix. For example an injection via
`inject(‘ember-simple-auth::session)` would resolve with these steps:

* With addon namespace:  `ember-simple-auth/src/services/session.js`
* No such service was found. Throw.

#### Namespaces containing `ember-` or `ember-cli-`

In [RFC #143: Module Unification - Main
Modules](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md#main-modules).
it was described that packages starting with `ember-` or `ember-cli-` would
have that prefix stripped during build time, and so invocation could ignore it.
For exmaple `{{ember-simple-auth::login}}` would be incorrect for a component
in the ember-simple-auth addon, and `{{simple-auth::login}}` would be correct.

**In this RFC that design is rejected**. Ember should be conservative about how
it changes the names of packages. A new developer installing a package should
not need to internalize a bunch of naming rules, knowing the name of the
package itself should be sufficient.

#### Explicit namespaces based on NPM scopes

NPM scopes present a tricky point of design. Since an NPM scope will already
have an `@` in the name of the scope, allowing them to be invoked in a
standard manner causes visual ambiguity. Additionally a packge from an NPM
scope will include a `/`. This increases confusion since `/` also denotes a
closing mustache or HTML tag. For example:

```hbs
{{! these are not very visually distinct }}
{{@scope/package::component-name}}
{{@scope-name}}

<@scope/package::ComponentName>
   Oh my, so many special characters
</@scope/package::ComponentName>

{{#@scope/package}}
   Oof.
{{/@scope/package}}
```

For this reason, NPM scoped packages will not be treated as invokable. Instead
the component helper must be use to invoke a scoped package. For example:

```hbs
{{component '@scope/package::component-name'}}

{{#component '@scope/package::component-name'}}
  Seems good
{{/component}}
```

In the future the addition of a namespace import helper for Glimmer templates
may further improve support for NPM scopes.

See "Explicit namespaces based on NPM scopes: Alternatives" for other options
that were considered here.

### Invoking a namespace's "main"

"Main" items can always be explicitly invoked. For example: `{{gadgets::main}}`.

"main" is also implicitly resolved if the namespace itself is the invocation,
however without the `::` the dash rule is in effect.

For example `{{gadgets}}` is not invokable and would be a property lookup.
However  `{{gadget-lib}}` would attempt each of the following ordered list:

* Local lookup:  `src/ui/components/some-source/-components/gadget-lib/template.hbs`
* Local lookup: `src/ui/components/some-source/-components/gadget-lib/component.js`
* Top level:  `src/ui/components/gadget-lib/template.hbs`
* Top level:  `src/ui/components/gadget-lib/component.js`
* With addon namespace, implicit "main":  `gadget-lib/src/ui/components/main/template.hbs`
* With addon namespace, implicit "main":  `gadget-lib/src/ui/components/main/component.js`
* A property.

Services likewise may inject an implicit "main":  `inject('ember-simple-auth')`
would inject the main export of ember-simple-auth, for example
`ember-simple-auth/src/services/main.js`.

The same rules apply to namespaces including an NPM scope. For example
`{{@scope/package}}` would resolve "main". An `@` as a first character and later
`/` satisfies the invokability rule similar to `::`. String with an initial `@`
and later `/` should not be property lookups. 

### Implicit namespaces via "source"

Ember invocations already have an implicit namespace. From an app you can invoke
`{{try-me}}` and it looks in the `src/` directory. It isn't required to be
explicit and type `{{my-app-name::try-me}}`.

This section on implicit namespaces extends that same logic to addon namespaces.

Note that when an explicit namespace is used (a `::` string) all local lookup
and implicit namespace is skipped. The invoked item must be found in the
specified namespace at the top level.

#### Implicit namespaces in templates

*Metadata on compiled Glimmer templates already contains the path on disk of
that template. We would like to expand this metadata to also include an absolute
specifier templates.*

When an explicit namespace is not present an implicit namespace may be
discovered via the `source` of a resolution. Once a lookup is returned from a
namespace, the `source` attached to later un-explicit lookups would cause the
namespace to be retained.

For example an explicitly namespaces invocation from an application:

```
{{! my-app/src/ui/components/some-component/template.hbs }}
{{animated::explode}}
```

Would plausibly cause this template with an in-explicit invocation to be
rendered:

```
{{! animated/src/ui/components/explode/template.hbs }}
{{implements-explode}}
```

Which in turn could render this template from the same addon namespace:

```
{{! animated/src/ui/components/implements-explode/template.hbs }}
Implemented.
```

Specifically `{{implements-explode}}` would would attempt each of the following
ordered list:

* Local lookup:  `animated/src/ui/component/explode/-components/implements-explode/template.hbs`
* Local lookup:  `animated/src/ui/component/explode/-components/implements-explode/component.js`
* In addon namespace:  `animated/src/ui/component/implements-explode/template.hbs`
* In addon namespace:  `animated/src/ui/component/implements-explode/component.js`
* Is a property.

#### Implicit namespaces in services

Service injections do not currently have source information encoded. Here we
propose adding that information via an AST transform and argument to
`inject.service`. This additional information provided by Ember CLI at build
time mirrors how `source` is already a compile-time addition for templates.

For example the following file:

```
// gadgets/src/services/main.js
import Service, { inject } from '@ember/service';

export default Service.extend({
  maguffin: inject()
});
```

Would inject a service `maguffin` from the addon while preserving the implicit
namespace of `gadgets` (`gadgets/src/services/maguffin.js`). The implicit
namespace is encoded at build time by an AST transform which adds a `source`
option to the injection. The post-transform file would look like:

```
// gadgets/src/services/main.js
import Service, { inject } from '@ember/service';

export default Service.extend({
  maguffin: inject('maguffin', {
    source: 'gadgets/src/services/main'
  })
});
```

All uses of Service  `inject` would have `source` added, including those in
components or helpers.

Because services are not configured in the Ember module unification config to be
an available nested collection for any other collection, services are never
subject for local lookup resolution despite using `source`.

### The addon migration path

#### Module Unification app and addon

The rules in this document generally describe Module Unification apps and
addons.

#### Classic app and module unification addon

When a classic Ember app consumes a module unification addon it can invoke the
components, helpers, and services from the addon according to the rules of this
RFC with the exclusion of local lookup.

#### Module Unification app and classic addon

Since module unification apps have no `app/` directory there is no location for
classic addons to merge themselves into. Instead, Ember CLI will detect when a
Module Unification app is using a classic addon and then re-export files from
that addon into the `src/` tree of the app.

For example a classic addon:

```
{{! gadgets/app/templates/components/try-me.hbs }}
Try me.
```

Would cause a file to be generated:

```
// src/ui/components/try-me/template.js
rexport default from 'gadgets/app/templates/components/try-me';
```

In this way classic addon would still merge into the `src` of a module
unification app as a transition step, preserving its API during the transition
period.

These re-exports will be created for components, helpers, and services.

### Namespace and source APIs on `owner`

Owner objects (the `ApplicationInstance` instance) in Ember should have new
options introduced that allow developers to programmatically interact with these
features.

At this level the `::` syntax is *not* introduced. This is because that syntax
does not adhere to the serialized absolute specifier syntax. Instead we rely on
a discrete option.

#### lookup

```
lookup(fullName, {
  source, // Optional source of the lookup
  namespace // Explicit namespace
});
```

#### factoryFor

```
factoryFor(fullName, {
  source, // Optional source of the lookup
  namespace // Explicit namespace
});
```

#### resolveRegistration

```
resolveRegistration(fullName, {
  source, // Optional source of the lookup
  namespace // Explicit namespace
});
```

#### hasRegistration

```
hasRegistration(fullName, {
  source, // Optional source of the lookup
  namespace // Explicit namespace
});
```


#### Registration specific APIs

Several owner APIs are specific to registrations. Without the introduction of
absolute specifiers, several of these APIs have ambiguous cases that arise once
namespace and source are added features to the container. The following changes
are suggested:

**unregister**

This API accepts a fullName and removes an entry in the registry and clears any
instances of it from the singleton caches (including the singleton instance
cache).

This API has ambiguity without large changes. A resolution will always win over
a registration. What if you resolve a factory with local lookup,  should
unregister clear all resolved items with the passed `fullName` regardless of
`source` including their singleton instances? Only the cached item with the
`fullName` and no `source`? Only if the cache is for something that was in the
registry and not resolved?

We lack a canonical representation of some factories after adding source and
namespace features. For example "main" export from an addon can be referenced by
several different lookup paths (with an explicit namespace, with any of several
sources). In the future, we're considering absolution specifiers the path
forward.

Instead `unregister` should be changed to throw a deprecation warning if you
attempt to un-register a factory already looked up. This has the effect of
removing the cache question entirely and focusing this interface on the
registry.

**inject**

Injections by type will still pertain to source or namespace based lookups. This
APIs should not be extended to accept `source` and `namespace` since that is not
a canonical way to refer to a specific lookup.

**register, registerOptions**

As much as possible in Ember registry APIs should be isolated from the container
proper.  These APIs should not be extended to accept `source` and `namespace`
since that is not a canonical way to refer to a specific lookup.

**registeredOption, registeredOptions**

These APIs should not be extended to accept `source` and `namespace` since there
is no way to register options for those lookups by canonical name.

### Adding support to the Ember Resolver

The Ember resolver is responsible for accepting the `source` and `namespace`
arguments in addition to the `fullName` being requested and returning a resolved
path.

Merely passing these additional values to the resolver's `resolve` method is
inadequate. Ember's container manages a singleton cache which maps most closely
to the name of the resolved item, thus it needs not only a factory in response
to a lookup but also a key to manage that cache.

Instead we will use the `expandLocalLookup` API and extend it to include a third
argument for the explicit namespace. The arguments to `exapandLocalLookup` will
be:

* `specifier` - the Ember specifier for this lookup (similar to a partial
  specifier)
* `source` - the source of the lookup. For entries in the `app/` directory this
  can continue to be a string based on `moduleName` (maintining backwards
  compatibility). For all entries in the `src/` directory this should be an
  absolute specifier. Ember's resolver will only perform local lookup if an
  absolute specifier is passed.
* `namespace` - the explicit namespaces of a lookup (the part before `::` if
   present)

For example:

```
{{x-inner}} in src/ui/components/x-outer/template.hbs
resolver.expandLocalLookup('template:x-inner', 'template:/my-app/components/x-outer');

{{gadgets-lib::x-inner}} in src/ui/components/x-outer/template.hbs
resolver.expandLocalLookup('template:x-inner', 'template:/my-app/components/x-outer', 'gadgets-lib');
```

The return value from Ember's resolver implementation will be:

* `null` if there is no matching resolution
* `null` is there is a matching resolution that is top level (non local, non namespaced)
* An absolute specifier if there is a matching local or namespaced resolution

### Planned Deprecations

This RFC does not suggest any specific deprecations to be added in the near
term. We've committed to making the transition path for applications and addons
as smooth as possible via the following steps:

* Implementing a codemod for applications to adopt the new filesystem layout
* Ensuring applications can incremental adopt the new filesystem via the "fallback resolver".
* Ensuring Module Unification apps can continue to consume classic mode addons.
* Ensuring classic mode apps can continue to consume Module Unification addons. 

## How we teach this

The guides must be updated to document namespace invocations for component,
templates, and addons.

A page should be added to the section "Addons and Dependencies" documenting the
API from the perspective of an addon author.

## Drawbacks / Alternatives

Continuing the current addon implementation where the `src/` directory of an
addon is simply merged with an app's `src/` directory would be a alternative. In
general we've found that system clumsy though, with addon's competing to claim
helper and component names needlessly.

The initial Module Unification RFC had this to say about invocation rules:

> Addons should use the same namespacing that will be used by consuming apps
> when invoking their own components and helpers from templates. For instance,
> if the ember-power-select addon has a date-picker component that invokes
> multiple main components, it should also invoke them in a template as
> {{power-select::main}} or more simply as {{power-select}}  

This expresses a subset of this RFC I'll call "Mandatory Namespace". In this
approach we would force addons to explicitly use `{{my-addon-name::` to invoke
anything from their own `src` directory at the top level. The advantage to this
approach is that it does not require an AST transform to be nice to use, it is
easier for us to implement, and it is more explicit. However it has notable
disadvantages: It is verbose and would make forking or refactoring and addon
difficult, and it is a different set of invocation rules than application
developers use. This would make it more complex to teach and learn.

Finally there is effort underway to convert Ember's container to use Glimmer DI
directly and the Glimmer resolver. This approach is largely an incremental one
that limits breaking changes for app and addon code. We would can this whole
process and wait on a bigger refactor to be complete and clearly presented.

#### Explicit namespaces based on NPM scopes: Alternatives

Alternatively, namespaces with an NPM scope could be used for invocation. For example
`{{@scope/package::component-name}}`.

This introduces some ambiguity. The syntax `{{@foo}}` already denotes accessing
a component's `foo` argument.

An NPM scope alone could not be invoked. To reference a package, a `/` is
required. To eliminate the ambiguity between component arguments and scopes, any
template literal matching the pattern `/^@.*\/.*/` would be treated as an
invocation. This eliminates the ambiguity technically, and shows one way the two
cases are not overly visually ambiguous.

For example:

* `{{@foo/bar::baz}}` would be an invocation because it starts with an `@` and
  contains a `/`.
* `{{@foo/bar}}` would be an invocation because it starts with an `@` and
  contains a `/`.

A further alternative: Instead of using the pattern `/^@.*\/.*/` to disambiguate
NPM scoped invocations from component arguments, we could drop the requirement
of using `@` at the beginning and simply use the presence of a  `/`  as a
disambiguation. I think this would be confusing since some templates use `/` for
paths in Ember today.
