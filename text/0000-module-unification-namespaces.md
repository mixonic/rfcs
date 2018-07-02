# Module Unification Packages

- Start Date: 2018-03-02
- RFC PR:
- Ember Issue:

# Module Unification with Ember Addons

TODO:

* Component/template section should talk about `{{component}}`
* Component/template section should talk about `prelude.hbs`
* Service namespace section should have more examples.
* Section on registration APIs need to be reviewed
* Downsides need updating, see [scratchpad](https://paper.dropbox.com/doc/Module-Unification-Namespace-Scratchpad-wbwwqafYnhNxXqY7SNIpB#:uid=000286165742384854025306&h2=Downsides-of-this-solution)
* Ember resolver section needs updating, still has `::` examples
* Can codemods exist for ember-data injections, etc? Example of ember-data migration path
* hype blog post

## Summary

[RFC #143](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md)
introduced a new filesystem layout for Ember applications we've called "Module
Unification".

Ember addons can use the module unification filesystem layout (i.e. an addon
with a `src/` directory) in place of the legacy `app/` or `addon/` folders. The
`src/` folder encapsulates a module unification "package", and Ember
applications can access components, helpers, and services from that package.

This RFC rejects some parts of the Module Unification RFC, such as
`::` component invocation syntax and implicit invocation
of an addon's `main`. These changes are largely motivated by conflicts between
these designs and new features in Ember and NPM.

This RFC describes fully how packages work in Ember, and how apps and addons will migrate
from "classic" `app/` and `addon/` folders to the modern `src/` folder.
It introduces new APIs:

* Accessing a component or helper from a package requires the `{{use}}` helper.
* The service injection API gains a new argument `package`. It also gains
  an argument `source` which Ember CLI will add to all invocation with
  an AST transform.
* `lookup`, `factoryFor` and other owner APIs gain new `package` and `source` APIs.

## Motivation

* TODO: Promote the bold sentence to the top here?

[RFC #143](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md)
introduced a new filesystem layout for Ember applications we've called "Module
Unification". The RFC described semantics for ["Addon modules"](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md#addon-modules) and the concrete
example of components from an addon being
invoked as `{{my-addon::my-component}}`.

The syntax of `::` was intended to allow us to specify an **explicit** package for an
invocation. The lookup would only occur within the prefixed package name. The RFC
touches on these lookup rules and the syntax for explicit invocation only lightly.

Contemporaneous with the Module Unification discussion in ember, NPM announced a new feature: Scoped packages. Scoped
packages have a composite name in the format `@${scope}/${packageName}`.
For example:

```js
// import from a "traditional" NPM package.
//
// Import the default export of the moment package.
// Often this is node_modules/moment/index.js
import moment from 'moment';

// Import from a "scoped" NPM package.
//
// Import the default export of a scoped package.
// For example node_modules/@angular/router/index.js
import { RouterModule } from '@angular/router';
```

**The addition of `@npmscope/` as a valid part of package names is in conflict with
the original design of Module Unification package invocation syntax.** For
example here are several invocation examples which would be valid presuming
Module Unification's original package proposal, NPM scopes, and angle bracket
invocation syntax:

```hbs
{{#@npmscope/package-name::component-name}}
  Invoke a component from a scoped NPM package.
{{/@npmscope/package-name::component-name}}

{{#@npmscope/package-name}}
  Would invoke an implicit component "main" in the package.
{{/@npmscope/package-name}}

<@Npmscope/PackageName::Name>
  Invoke a component from a scoped NPM package.
</@Npmscope/PackageName::Name>

<@Npmscope/PackageName>
  Some content. Would invoke an implicit component "main" in the package.
</@Npmscope/PackageName>
```

Because RFC \#143 was a) not designed with NPM scopes in mind, b) was not
designed with `@var` syntax fully explored, and c) was not designed with
Ember's consensus `<AngleBracket>` syntax the examples above violate several design constraints:

- **Use of `@` when invoking a component from a scoped package should be avoided.**
  Recent versions of Ember and Ember RFCs have introduced the sigil `@` to
  templates with a specific meaning. `{{@someArg}}` reads an argument passed to
  a component, and arguments are themselves passed with the `@` prefix (for
  example `<MyComponent @someArg={{someObj}}>`). Arguments can also be invoked
  (`<@someArgWithClosureComponentValue>` per [RFC 226](https://github.com/emberjs/rfcs/blob/master/text/0226-named-blocks.md#unified-renderable-syntax)) or be created from a block (`<@someArg= >` per [RFC
  317](https://github.com/emberjs/rfcs/blob/named-block-syntax/text/317-named-block-syntax.md#summary)). As the meaning of `@` is very specific and important in Ember templates,
  we should not create visual ambiguity and confusion by using it for non-variable component lookup. `<@scope/package::name>` is a poor invocation syntax for a
  component from an NPM scoped package because it is visually ambiguous with
  Ember's existing `@` usage.
- **Use of `/` when invoking a component from a scoped package should be avoided.**
  When using angle brackets multiple closing tags are very difficult to
  disambiguate from multiple closing tags. We’ve already discouraged use of `/` in
  component names by disallowing arbitrary directories in Module Unification layout apps. For comparison here are two plausible strings in an Ember template which
  are difficult to disambiguate:
    - `</@Npmscope/PackageName::Name>`
    - `</@Npmscope></PackageName::Name>`
- **NPM package names should be treated as literal strings.** The issues addressed
  by this RFC arose
  because we made an assumption that NPM package names would be a single word or
  a dasherized series of words. That was incorrect, as both `@` and `/` are now
  valid parts of an NPM package name. In the future, we should assume that
  new features may be added to NPM with additional characters. So, to be
  future-safe our design should
  treat NPM package names as strings where any characters would be valid.

I suggest two further constraints:

- **Use of NPM scopes should not be discouraged.** It could be tempting to accept a design
  which works well for “normal” NPM packages but requires more effort for users of
  scoped packages. By accepting one of these options we would a) make Ember more
  difficult to learn since you must learn two different solutions for the same
  problem of addon component/helper/service invocation and b) by making un-scoped packages more
  ergonomic we would discourage addon authors from using the scoped packages
  feature. In practice, scoped packages seem like a good feature for authors to
  consider and in some corporate settings use of them may not be optional.
- **Resolution rules should be consistent across parts of Ember.** For example
  normalization rules for package names in templates and in service injections
  should be the same. Given that a developer is using Ember Data they should not
  refer to it in some places as EmberData and others as `ember-data`.

These are the problems and constraints that motivate this RFC.

## Detailed design

### Removal of RFC \#143 Module Unification features

* TODO: Link to examples of main usage

The following features from RFC \#143 are removed from the updated design:

- **Drop the single line package+name syntax (`::`)**. This syntax is the root
  cause of several issues in templates outlined in the motivation section. This RFC does not
  suggest any replacement "single-line" invocation syntax in templates. The
  `::` syntax was also implied to be used with service injections. As
  we're removing the micro-syntax from templates, we will also remove it from
  the service injection API.
- **Drop implicit "main" rules from module unification.** Because implicit main is a
  *resolution rule* and not an invocation syntax rule, there is no way for us to
  know if `<Foo />` is an app component or a package name + main without
  significant analysis of the program at build time. As a runtime
  resolver is sure to be part of common Ember usage for a long while, it seems
  beneficial not to introduce this ambiguity. Additionally, the whole-program knowledge
  needed to disambiguate `<Foo>` may make it difficult to write tooling
  for templates.
- **Remove the stripping of `ember-` or `ember-cli-` prefixes from
  package names.** [RFC #143: Module Unification - Main
  Modules](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md#main-modules).
  describes stripping these prefixes during build time, and so invocations
  referencing those package names can ignore the prefix.
  For example `{{ember-simple-auth::login}}` would be incorrect for a component
  in the ember-simple-auth addon, and `{{simple-auth::login}}` would be correct.
  As this RFC suggests removing single-line invocation entirely, this
  convenience is unnecessary. Ember should be conservative about how
  it changes the names of packages. A new developer installing a package should
  not need to internalize a bunch of naming rules. Knowing the name of the
  package itself should be sufficient.

The removal of these features of course requires that we provide alternatives.
These follow.

### New Package APIs

Without using the `owner` API, only **services**, **components**, and **helpers** from a module unification
package can be invoked by an application. The modules contained in an addon's
package are defined by the files in the addon's `src/` directory.

Both addons and app have a package. For this reason writing Module Unification
addon and app code should be very consistent. Modules in the
`src/` directory are invoked by their bare names, and invoking/injecting/looking up things from another package will require a special API.

A summary of new behaviors follows (specific detailed designs each have a section of this RFC):

* Invoking a component, helper, or service without an explicit package
  will always look to the **implicit** package of that lookup. For an addon
  or app, this means that first local lookup is considered for resolution then
  their top-level directory for that module under their own `src/`.
* To invoke a component or helper from another package, the `{{use}}` helper
  is defined by this RFC.
* To implicitly inject a service from a package the `source` argument
  is added to the injection API. To explicitly inject a service from another package the `package` argument is defined
  for the injection API.
* To look up or resolve an arbitrary module in local lookup the `source`
  argument is defined for `lookup`, `factoryFor`, and `resolveRegistration`. To explicitly look up or resolve an arbitrary module the `package` argument is added.

Each of these changes is described below in detail.

To help the RFC be approachable I'll describe behaviors using plausible
filenames. An actual implementation for any of these examples would be
based on Module Unification specifiers. Most filenames used in examples have alternative forms which are valid.

### Implicit packages for templates

An application using Module Unification already has an implicit package.
From an app you can invoke
`{{try-me}}` and it looks in the `src/ui/components/` directory. As an app author
you are not required to be
explicit and provide the app name.

This RFC suggest addon templates behave in the same manner. An invocation of
`{{try-me}}` in an addon will resolve:

* Local lookup `my-addon/src/ui/components/invoking-component/try-me/component.js`
* Top-level lookup `my-addon/src/ui/components/try-me/component.js`

Metadata on compiled Glimmer templates already contains the path on disk of
a template. This metadata should be expanded to also include an absolute
specifier for the template.

Using the `source` meta data on a template (or another appropriate build-time value), all lookups from that template will determine the package of the templates
and use it for any implicit invocations in the template itself.

### The `{{use}}` helper

This RFC describes a new helper `{{use}}` for explicit importing of a component or
helper from a package. This replaces the `::` syntax in templates.

The `{{use}}` helper's functionality is a subset of JavaScript's `import` keyword, but the syntax is
different. `{{use}}` is required to bring a component or helper from another
package (and not from a directory or file) into the template's scope as a symbol.

The `{{use}}`
helper has unique syntax compared to any keywords present in Glimmer today
because it introduces a symbol to the template without a block.

To start with an example: In this template the `{{use}}` helper imports a component `Widget` from the
`gadget` addon.

```hbs
{{! invokes node_modules/gadget/src/ui/components/Widget/component.js }}

{{use Widget from 'gadget'}}
<Widget @options=someOptions @value=someValue />
```

The semantics of `{{use}}` are the following:

- `{{use}}` introduces a symbol to template that corresponds to the
  component or helper (in the above example `Widget`) from the package (in the above example `gadget`). Normal
  invocation rules apply to the symbol: To be invoked with `{{` it must have a
`-`, to be invoked with a `<` it must start with a capital letter.
- There are no "default" imports. The component or helper being imported is
  always named.
- The symbol introduced by `{{use}}` is an invokable symbol. It has identical
  semantics to a symbol with a closure component value. The resolution of the
  component module is (semantically if not literally) performed at the time of
  `{{use}}`, not at the time of invocation. Because the symbol's value is
  already resolved when the component is invoked, there are no normalization
  rules (`CapCase` -> `caps-case`) applied to symbols introduces by `{{use}}`.
- `{{use}}` can use `as` to map a component or helper name to a differently
  named symbol.
- `{{use}}` can only be used to access absolute package names. There are no
  relative paths relating to file names or directories.
- `{{use}}` introduces a symbol for the entire template. The helper, like
  `import`, cannot be used dynamically. For example it cannot be invoked inside an `{{#if}}` block or
  component block.
- If a symbol introduced by `{{use}}` is already present in a template, the
  value from `{{use}}` will overwrite any previous value.

The syntax of `{{use}}` can be described using syntax spec language
comparable to the [ES `import`
spec](https://www.ecma-international.org/ecma-262/6.0/#sec-imports):

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
  * *StringLiteral* - The module unification package from which a components
    or helper is being imported. Usually
    an NPM package name.
* *ImportedBinding*
  * *BindingIdentifier* - The symbol used to reference the import.

And what follows are several examples of valid usage.

##### Using a component with mustaches

```hbs
{{! use package-name/src/ui/components/component-name/component.js }}

{{use component-name from 'package-name'}}

{{component-name}}

{{#component-name}}
  Some content
{{/component-name}}
```

```hbs
{{! use @npmscope/package-name/src/ui/components/component-name/component.js }}

{{use component-name from '@npmscope/package-name'}}

{{#component-name}}
  Some content
{{/component-name}}
```

##### Using a component with angle brackets

```hbs

{{! use package-name/src/ui/components/Name/component.js }}

{{use Name from 'package-name'}}

<Name />

<Name>
  Some content
</Name>
```

```hbs
{{! use @npmscope/package-name/src/ui/components/Name/component.js }}

{{use Name from '@npmscope/package-name'}}

<Name>
  Some content
</Name>
```

##### Re-assignment of the imported component to a new symbol

`{{use}}` introduces a symbol to the template, and can choose a local name.

```hbs
{{! use @npmscope/package-name/src/ui/components/Name/component.js }}

{{use Name as component-name from 'package-name'}}

{{component-name}}


{{use Name as component-name from '@npmscope/package-name'}}
{{component-name}}
```

```hbs
{{! use @npmscope/package-name/src/ui/components/component-name/component.js }}

{{use component-name as Name from 'package-name'}}

<Name />


{{use component-name as Name from '@npmscope/package-name'}}
<Name />
```


##### Multiple Imports

`{{use}}` can import multiple components separated by `,`.

```hbs
{{! use ember-power-select/src/ui/components/Option/component.js }}
{{! use ember-power-select/src/ui/components/Select/component.js }}

{{use
  Option,
  Select as PowerSelect
from 'ember-power-select'}}

<PowerSelect>
  <Option />
</PowerSelect>
```

##### `{{use}}` overwrites previous values for a symbol

```hbs
{{use ComponentName from 'package-name'}}
{{use ComponentName from 'other-package-name'}}

{{! invokes other-package-name/src/ui/components/ComponentName/component.js }}
<ComponentName />
```

### Implicit packages for services

Service injections do not currently have source information encoded. Here we
propose adding that information via an AST transform and argument to
`inject.service`. This additional information provided by Ember CLI at build
time mirrors how `source` is already a compile-time addition for templates.

For example given a service `main` in the addon `gadget`, an injection
will maintain the `gadget` package:

```js
// gadgets/src/services/main.js
import Service, { inject } from '@ember/service';

export default Service.extend({
  maguffin: inject()
});
```

This snippet injects a service `maguffin` from the addon while preserving the implicit
package of `gadgets` (`gadgets/src/services/maguffin.js`). The implicit
package is encoded at build time by an AST transform which adds a `source`
option to the injection. The post-transform file would look like:

```js
// gadgets/src/services/main.js
import Service, { inject } from '@ember/service';

export default Service.extend({
  maguffin: inject('maguffin', {
    source: 'gadgets/src/services/main'
  })
});
```

All uses of Service `inject` would have `source` added, including those in
components or helpers.

Note: Because services are not configured in the Ember module unification config to be
an available as a nested collection, services are never
subject for local lookup resolution despite using `source`.


### Explicit packages for service injections

The argument `package` is added to the `inject` API to specify an
explicit package. The packages provided by this argument is absolute.
Any implicit package is over-ruled.

For example:

```js
export default Ember.Component.extend({

  // inject node_modules/@ember/data/src/services/store.js
  store: inject('store', { package: '@ember/data' }),

  // inject node_modules/ember-simple-auth/src/services/session.js
  session: inject({ package: 'ember-simple-auth' })
});
```

### Explicit packages for `owner` APIs

Owner objects (the `ApplicationInstance` instance) in Ember should have new
options introduced that allow developers to programmatically interact with these
features.

##### `owner.lookup()`

```js
lookup(fullName, {
  source, // Optional source of the lookup
  package // Explicit package
});
```

##### `owner.factoryFor()`

```js
factoryFor(fullName, {
  source, // Optional source of the lookup
  package // Explicit package
});
```

##### `owner.resolveRegistration()`

```js
resolveRegistration(fullName, {
  source, // Optional source of the lookup
  package // Explicit package
});
```

##### `owner.hasRegistration()`

```js
hasRegistration(fullName, {
  source, // Optional source of the lookup
  package // Explicit package
});
```

#### Registration specific APIs

Several owner APIs are specific to registrations. Without the introduction of
absolute specifiers, several of these APIs have ambiguous cases that arise once
package and source are added features to the container. The following changes
are suggested:

**unregister**

* TODO: define absolute specifier

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
package features. For example "main" export from an addon can be referenced by
several different lookup paths (with an explicit package, with any of several
sources). In the future, we're considering absolution specifiers the path
forward.

Instead `unregister` should be changed to throw a deprecation warning if you
attempt to un-register a factory already looked up. This has the effect of
removing the cache question entirely and focusing this interface on the
registry.

##### `owner.inject()`

Injections by type will still pertain to source or package based lookups. This
APIs should not be extended to accept `source` and `package` since that is not
a canonical way to refer to a specific lookup.

##### `owner.register()`, `owner.registerOptions()`

As much as possible in Ember registry APIs should be isolated from the container
proper.  These APIs should not be extended to accept `source` and `package`
since that is not a canonical way to refer to a specific lookup.

##### `owner.registeredOption()`, `owner.registeredOptions()`

These APIs should not be extended to accept `source` and `package` since there
is no way to register options for those lookups by canonical name.

### Adding support to the Ember Resolver

The Ember resolver is responsible for accepting the `source` and `package`
arguments in addition to the `fullName` being requested and returning a resolved
path.

Merely passing these additional values to the resolver's `resolve` method is
inadequate. Ember's container manages a singleton cache which maps most closely
to the name of the resolved item, thus it needs not only a factory in response
to a lookup but also a key to manage that cache.

Instead we will use the `expandLocalLookup` API and extend it to include a third
argument for the explicit package. The arguments to `exapandLocalLookup` will
be:

* `specifier` - the Ember specifier for this lookup (similar to a partial
  specifier)
* `source` - the source of the lookup. For entries in the `app/` directory this
  can continue to be a string based on `moduleName` (maintining backwards
  compatibility). For all entries in the `src/` directory this should be an
  absolute specifier. Ember's resolver will only perform local lookup if an
  absolute specifier is passed.
* `package` - the explicit package of a lookup (the part before `::` if
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
* `null` is there is a matching resolution that is top level (non local, not from a package)
* An absolute specifier if there is a matching local or package-scoped resolution

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

```hbs
{{! gadgets/app/templates/components/try-me.hbs }}
Try me.
```

Would cause a file to be generated:

```js
// my-app/src/ui/components/try-me/template.js
export default from 'gadgets/app/templates/components/try-me';
```

In this way classic addon would still merge into the `src` of a module
unification app as a transition step, preserving its API during the transition
period.

These re-exports will be created for components, helpers, and services.

### Planned Deprecations

This RFC does not suggest any specific deprecations to be added in the near
term. We've committed to making the transition path for applications and addons
as smooth as possible via the following steps:

* Implementing a codemod for applications to adopt the new filesystem layout
* Ensuring applications can incremental adopt the new filesystem via the "fallback resolver".
* Ensuring Module Unification apps can continue to consume classic mode addons.
* Ensuring classic mode apps can continue to consume Module Unification addons.

## How we teach this

The guides must be updated to document package invocations for component,
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
