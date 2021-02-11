# KEP-2309: An IDL for Kubernetes

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [A bit of background](#a-bit-of-background)
- [Who interacts with our types?](#who-interacts-with-our-types)
- [What problems do we have now...](#what-problems-do-we-have-now)
  - [...as API Authors?](#as-api-authors)
    - [Pseudo-IDL is hard to write properly](#pseudo-idl-is-hard-to-write-properly)
    - [Pseudo-IDL is unfriendly to non-Go programmers](#pseudo-idl-is-unfriendly-to-non-go-programmers)
    - [Pseudo-IDL is not one-to-one with the k8s type system](#pseudo-idl-is-not-one-to-one-with-the-k8s-type-system)
  - [...as API consumers](#as-api-consumers)
    - [Generation of CRD YAML is unfriendly to non-Go environments](#generation-of-crd-yaml-is-unfriendly-to-non-go-environments)
    - [Wire formats are strongly tied to source formats](#wire-formats-are-strongly-tied-to-source-formats)
    - [Pseudo-IDL is hard to review](#pseudo-idl-is-hard-to-review)
  - [...as tooling maintainers?](#as-tooling-maintainers)
- [Ideal Properties of a Kubernetes IDL](#ideal-properties-of-a-kubernetes-idl)
- [Goals](#goals)
- [Non-Goals](#non-goals)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Addendum: Why do we need to do this in k/k](#addendum-why-do-we-need-to-do-this-in-kk)
- [Addendum: Footnotes](#addendum-footnotes)
  - [1. core/v1 Mistakes](#1-corev1-mistakes)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] (R) Graduation criteria is in place
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## A bit of background

Through our API conventions & best practices, API mechanics, and implicit
contracts, Kubernetes defines a type system for its API.  This type system
includes a specific set of primitives (integers, quantity, string, etc),
as well as several forms of "algebraic" data types (kinds, structs,
list-maps, etc).  Some of these concepts commonly exist elsewhere, with
slight restrictions (e.g. struct), others (e.g. kind, list-map, quantity)
are largely specific to Kubernetes.

Historically, we've encoded these types via a combination of Go types,
conventions, special comments (sometimes called "comment tags" or
"markers"), Go struct tags, and hand-written bespoke validation
& defaulting code (we'll refer to these things as the Go Pseudo-IDL, or
just pseudo-IDL for short).

The exact methods for making use of pseudo-IDL have varried between parts
of the Kubernetes ecosystem:

- "core" kubernetes/kubernetes uses [gengo][gengo]
  & [code-generator][code-generator]

- some SIG-sponsored projects, as well as the majority of the broader k8s
  ecosystem, use [controller-tools][controller-tools], part of the
  [KubeBuilder][kb] subproject of SIG API-Machinery.

- A few projects use their own toolchains due to specific
  constraints (e.g. Istio), but these are relatively rare.

Each of these projects uses slightly different "dialects" of the
pseudo-IDL, varying in the exact names an syntax of the marker comments
and such due to somewhat organic branching evolution.

[gengo]: https://git.k8s.io/gengo

[code-generator]: https://git.k8s.io/code-generator

[controller-tools]: https://sigs.k8s.io/controller-tools.

[kb]: https://book.kubebuilder.io

## Who interacts with our types?

* **API Authors** produce API definitions, commonly embedding other API
  types within their own (think PodTemplateSpec within a custom workloads
  controller).

  * **Kubernetes contributors** define built-in APIs for Kubernetes, as
    well as *reviewing* them for correctness.

  * **Go extension authors** write CRDs and custom controllers/operators
    in Go (often with [KubeBuilder][kb])

  * **Non-Go extension authors** write CRDs for systems in a variety of
    other languages, including both imperative programming languages (e.g.
    Python) and as abstractions over existing templatating systems (e.g.
    to port Helm charts to full operators).

* **API consumers** consume API definitions from elsewhere.  They write
  custom controllers that interact with core APIs and custom resources,
  using both Go and non-Go languages (e.g. Python, Java).

* **Tooling maintainers** maintain tooling that compiles or translates API
  definitions into other forms (CRD YAML, API docs, Go deepcopy code,
  etc).

## What problems do we have now...

### ...as API Authors?

#### Pseudo-IDL is hard to write properly

It's pretty easy to make mistakes when writing markers, JSON, and proto
tags.  Consider:

<details>
<summary>an example</summary>

```go
type MyType struct {
    ANumber int `json:"thenumber"`

    // +kubebuilder:validtion:Maximum=11

    // TODO: this should really be a pointer

    // Volume controls the volume
    Volume int `json:"Volume"`
}
```

There are three mistakes in the above example:

```go
type MyType struct {
    // - probably accidentally refactored and
    //   forgot to change the JSON tag

    ANumber int `json:"anumber"`

    // - markers should be in the second-closest comment
    //   block, or closer.  Probably added a TODO without
    //   realizing it would disrupt things
    // - "validation" was misspelled
    // - probably did a find-replace that was case-insensitive
    //   by accident, and therefor added an uppercase json field

    // +kubebuilder:validation:Maximum=11
    // TODO: this really should be a pointer

    // Volume controls the volume
    Volume int `json:"volume"`
}
```

</details>

The root issue in of each of these mistakes can be found in the following
causes, each of which can be seen in our own codebase
[<sup>1</sup>](#1-corev1-mistakes):

1. There are 3-4 different places & syntaxes that affect any given type or
   field (raw Go type information, JSON & proto struct tags, markers)

2. JSON & proto tags are largely redundant, yet must be specified,
   otherwise serialization won't work properly

3. It's easy to typo or misplace markers and struct tags in ways that
   don't get caught until much later.

4. These things are often simply copied from place to place without full
   understanding of them or the implications of getting them wrong, due to
   their complicated & redundant manner.

#### Pseudo-IDL is unfriendly to non-Go programmers

* Requiring a full Go toolchain for non-Go projects is cumbersome
  & potentially blocking for certain environments

* Non-Go programmers are often hesitant to learn an entirely separate
  programming language, but are comfortable with learning an IDL

#### Pseudo-IDL is not one-to-one with the k8s type system

* Go expresses concepts that we don't want or need in our "platonic" k8s
  IDL (e.g. interfaces, floats, aliasing string to int, complex maps)

  * This makes it harder to teach, since folks used to Go will reach for
    those things, and then become frustrated when they learn that those
    concepts aren't allowed.

* Go needs augmentation with comments to describe concepts that are
  fundamental to the type system (e.g. group-versions, kinds,
  enumerations, volumesource-esque things, ordered maps)

  * This makes it harder to teach because there's no direct representation
    for these concepts natively.

### ...as API consumers

#### Generation of CRD YAML is unfriendly to non-Go environments

* Writing Go code to generate CRDs makes little sense in environments
  without any pre-existing Go code

* Generation is in-binary, making custom extensions (even those in Go)
  more difficult to write without requiring duplicate parsing passes

#### Wire formats are strongly tied to source formats

Because we rely on Go types to serialize to JSON, experimenting with
different structures in Go is difficult (e.g. it might rely on
hand-writing custom JSON serialization)

While it's not *remotely* in scope for this proposal to change the Go
types representation, it has occasionally come up that it'd be nice to
experiment (e.g. types the preserve unknown fields round-trip, interfaces
instead of giant structs where you have to set only one field).

#### Pseudo-IDL is hard to review

* For many of the reasons pseudo-IDL is [hard to
  write](#pseudo-idl-is-hard-to-write-properly), it's also hard to review:
  reviewers have to look at many different pieces of information, commonly
  with slightly different values.  This results in issues like [bad struct
  tags][upper-field-name].

* Different parts of the pseudo-IDL are split across different files
  (types vs validation vs defaulting), making it more difficult to review
  holistically.

### ...as tooling maintainers?

* Duplicated efforts: [k8s.io/code-generator][code-generator] and
  [controller-gen][controller-tools] use different tooling to generate
  code & artifacts from markers.  This results in:

  * duplicated maintenance effort
  * slightly differing semantics
  * generation modules written in the less flexible k8s.io/code-generator
    cannot easily compatible with controller-gen

* Difficulty parsing:

  * a full Go parser & type-checker are required (due to aliasing over
    primitives, dot imports, etc), as well as a full Go standard library

  * Parsing is slow, since it requires traversing non-IDL resources in the
    same package (hacks can speed this up, but only so much, and at the
    expense of good error messages)

  * Properly parsing markers is clumsy & easy to mess up between different
    implementations, since it relies on wobbly definitions of which comments
    are associated with a given field or type

  * It's hard to distribute your pseudo-IDL without distributing the rest
    of your types package (this is a problem for orgs that have
    restrictions on which source code they can deliver to end users)

* Pseudo-IDL is hard to validate: since markers are just comment, and are
  explicitly defined, namespaced, etc, it's hard to know if
  a non-conforming or unknown marker is actually wrong, or just intended
  to be consumed by a different piece of tooling, or just happens to look
  like a marker.

## Ideal Properties of a Kubernetes IDL

1. **Guide authors in the right direction**: it should be easy to write
   APIs that follow the Kubernetes API conventions and recommendations,
   and hard/impossible to write invalid APIs

2. **Be easy to consume by different pieces of tooling**: a serialized
   form (think proto descriptor files) should exist that’s easy to consume
   in the language of your choice.  This way, different tools/languages
   can do code generation, etc from their native environments

3. **Be a good code generation input**: it should be relatively
   straightforward to generate idiomatic types in Go.  While it’s
   a non-goal to actually implement output for other languages, it should
   be possible to do so and get idiomatic types (consider this "external
   work" from goal 5).

4. **Be a good OpenAPI generation input**: it should be relatively
   straightforward to compile our IDL into Kubernetes OpenAPI.

5. **Enable future/external work & experimentation**: there are many
   possible follow-up improvements or experiments that can be done with an
   IDL. While implementing these explicitly does not fall under the
   concrete success criteria for this proposal, the IDL pipeline should be
   flexible enough to support developing them as future work by us or
   external tooling by third-party projects.

## Goals

1. **Transparent IDL-to-Go generator**: we'll introduce a generator that
   generates *identical* Kubernetes Go pseudo-IDL from the new IDL.  This
   will mitigate review time required: the review should largely need to
   consist only of making sure that "types.go" isn't present in the
   changes list.

2. **Auto-migration tooling**: as part of the MVP, auto-migration tooling
   will be written to migrate from pseudo-IDL to IDL.  This will enable
   both community members and k/k to perform automated migration. Together
   with goal 1 (IDL-to-Go generator), this should minimize the work to
   migrate k/k.

3. **Auto-migrate k/k**: use goals 1 & 2 to automatically migrate
   kubernetes types.  Linting will be introduced that verifies that
   generated `types.go` files are in sync with the source IDL, similarly
   to any other generated file.  All other generation tooling
   (deepcopy-gen, client-gen, etc) will continue to work on Go pseudo-IDL
   (now generated) initially.

4. **CRD generation tooling**: controller-tools (part of KubeBuilder) will
   be migrated to work on the IDL in order to enable community usage of
   the IDL as well.

5. (post-MVP) **Incremental tooling migration**: after goals 1-4 are
   implemented and consensus is reached around the IDL implementation
   details, incremental migration of existing tooling from pseudo-IDL to
   IDL will occur.  This can be incremental because of goal 1, but should
   happen eventually to reduce overall maintenance burden.

## Non-Goals

1. **Be a custom serialization format**: unlike proto, we’re not
   interested in *directly* describing a serialization format.  Instead,
   we’re interested in being able to output types.go, proto IDL, code
   suitable for serializing in JSON, etc

2. **Support all features of OpenAPI**: the Kubernetes API guidelines
   suggest avoiding certain OpenAPI constructs, while the structural
   schema requirements explicitly reject others.  We’re interested in an
   IDL that guides people down the right path, and that doesn’t support
   constructs that aren’t allowed

3. **Deprecate/replace OpenAPI everywhere**: it’s not currently intended
   that this replace all uses of OpenAPI -- OpenAPI is still part of the
   CRD specification, will still be published by the API server, etc.

4. **Change k/k’s Go type representation**: the existing codebase is
   large and intricate.  For the sake of compatibility, initial efforts
   will focus around generating 1-1 compatible Go with the code that
   exists currently.

### Risks and Mitigations

* **Learning curve**: any new setup has a potential learning curve.  That
  being said, terminology and theoretical constructs should be immediately
  familiar to anyone who works with kubernetes, which means only the
  syntax changes.

* **Tooling maintenance**: this introduces several new tools (KDL
  compiler, decompiler).  That being said, the intention is to supercede
  and simplify a variety of existing tools (code-generator/gengo,
  kubernetes-clients/gen, controller-tools), so this should eventually
  converge on a significantly defragmented set of tooling within the
  Kubernetes ecosystem and overall lower maintance burden across between
  API Machinery subprojects and k/k.

* **Code migration**: there’s a lot of types within the Kubernetes
  project.  To mitigate this, it *must* be possible to automatically
  translate the existing pseudo-IDL to the new form with tooling.

## Addendum: Why do we need to do this in k/k

While we could just do this as a self-contained part of the KubeBuilder
project (it would certainly be easier -- we wouldn't need a KEP!), such an
approach presents a significant downside:

Historically, a major source of frustration in using our existing
tooling has been the drift between k/k tooling and CRD tooling.  This
has largely taken three forms:

1. Differing ways of doing validation/defaulting/etc leading to validation
   being entirely absent in subsections of CRDs (think PodSpec embedded in
   a custom workload CRD).

2. k/k adding details that lead to unusuable type definitions (e.g.
   ["Default value not allowed for in
   x-kubernetes-list-map-keys"][unusable-types])

3. Type versioning being unwieldy to deal with due to Go versioning
   restrictions

While an IDL won't immediately solve issue 1 (although it can improve the
situation over time -- some is better than none!), it can certainly help
solve issues 2 and 3.

Stopping such drift between k/k and CRDs, and even reducing it, would be
a major win for the developer experience of writing CRDs for kubernetes.

[unusable-types]: https://github.com/kubernetes/kubernetes/issues/91395

## Addendum: Footnotes

### 1. core/v1 Mistakes

- We completely ignore the actual value of most proto tags data when
  generating (see [go-to-protobuf][ignored-proto]), leading to largely
  nonsensical proto tags in the actual codebase (e.g. [non-slice fields
  being marked as `rep`][proto-rep])

- We've published APIs with uppercase field names [because we missed
  a syntax error in code review][upper-field-name]

[ignored-proto]: https://github.com/kubernetes/code-generator/blob/8cc0d294774b931ef40bb2f1fb3a7bc06343ffa9/cmd/go-to-protobuf/protobuf/generator.go#L574

[proto-rep]: https://github.com/kubernetes/api/blob/master/core/v1/types.go#L35://github.com/kubernetes/api/blob/c873f2e8ab25481376d5b50b9d22719d6b2a1511/core/v1/types.go#L1597

[upper-field-name]: https://github.com/kubernetes/api/blob/c873f2e8ab25481376d5b50b9d22719d6b2a1511/core/v1/types.go#L1597
