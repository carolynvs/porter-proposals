---
title: "PEP TITLE" # Limit 40 characters
number: "NNN" # Assigned by maintainer after you get approval to submit a proposal
status: "provisional"
owner: "@GITHUB_USERNAME"
authors:
  - "@GITHUB_USERNAME"
created: "YYYY-MM-DD"
type: "PROPOSAL_TYPE" # feature, process
---

TODO: View the markdown source and follow the directions in the `<!-- -->`
comments. The comments should not be included in your PEP.

## Abstract

<!--
A short (~200 word) description of the technical issue being addressed.
-->


## Motivation

<!--
It should clearly explain why Porter's existing functionality is inadequate to
address the problem that the PEP solves and identify the impacted audience(s) (mixin
developers, bundle authors, end-users). PEP submissions without sufficient
motivation may be rejected outright. This is the most important part at the
beginning and is required before moving forward.
-->


## Rationale

<!--
The rationale fleshes out the specification by describing why particular design
decisions were made. It should describe alternate designs that were considered
and related work.

The rationale should provide evidence of consensus within the community and
discuss important objections or concerns raised during discussion.
-->


## Specification

<!--
The technical specification should describe the command and/or configuration
syntax and semantics of any new feature.

* If this is a command, we are looking for what the `porter help` would look
  like: description of command, arguments, flags, default behavior and error
  handling.

* If this is a syntax change to a configuration file, define the allowed syntax,
  at least one example per use case, covering defaults and error handling.

* All PEPs will be reviewed for user experience. So make sure to think about the
  common use case, how people can accomplish more advanced scenarios, precedence
  from existing Porter features or other tools in the ecosystem, and how the
  change fits into Porter workflows and tasks.

The spec should be detailed enough that someone other than the PEP
authors can understand what needs to be implemented.
-->


## Implementation

<!--
After the PEP status is changed to implementable, when the PEP has been
implemented link to the pull request(s) here.
-->


## Backwards Compatibility

<!--
All PEPs that introduce backwards incompatibilities must include a section
describing these incompatibilities and their severity.  The PEP must explain how
to deal with these incompatibilities, possibly with defaulting or migrations.
PEP submissions without a sufficient backwards compatibility treatise may be
rejected outright.
-->


## Security Implications

<!--
If there are security concerns in relation to the PEP, those concerns should be
explicitly written out to make sure reviewers of the PEP are aware of them.
Mitigations should be included if possible.
-->


## Rejected Ideas

<!--
Throughout the discussion of a PEP, various ideas will be proposed which are not
accepted. Those rejected ideas should be recorded along with the reasoning as to
why they were rejected. This both helps record the thought process behind the
final version of the PEP as well as preventing people from bringing up the same
rejected idea again in subsequent discussions.
-->


## Open Questions

<!--
Before a PEP is implemented, questions can come up which warrant further discussion.
Those questions should be recorded here so people know that they are being
thought about but do not have a concrete resolution. This ensures all concerns
are addressed prior to accepting the PEP and reduces repeating prior discussion.
When possible, link the question to where it is being discussed, such as a
[forum] post/comment.
-->

__

Right now we are re-solving distribution mechanisms with how we distribute mixins.
Such as versioning, source and digests. In addition, our current mixin architecture
requires that all mixins be compatible with the base distribution of the invocation image.

What if the mixin binary remained the same, but they are all distributed in docker images,
where the entry point is the mixin binary. The invocation image becomes just a container
for bundle specific stuff, e.g. porter.yaml, helper scripts, and mixins are run
by mounting the invocation image as a volume. Mixins can then interact with the file system
and porter can step in to share environment variables between steps.

Bundle
- invocation image = porter DinD
- bundle filesystem = built during porter build
- mixins...

porter install
- collect params, resolve creds
- execute dependencies
- run bundle

Run bundle
  client:
    save referenced images as tar
    mount referenced images as a volume /cnab/images/
  k8s:
    podman? skopeo? (equivalent of docker save, must preserve images)
    volume for cached images
    (same as client)
  runtime: (runner)
    FROM docker:dind. Requires --privileged watch https://github.com/nestybox/sysbox/issues/64 to not need it
    https://github.com/containers/podman/issues/4131
    know it's in k8s and make pods?
    docker load from /cnab/images
    create volume from bundle filesystem image
    run each mixin as a container with the shared volume
    ASSUME WE CAN FIGURE IT OUT

mixin bundle:
The bundle interface is sh.porter.mixin, it's for communicating with porter, not the steps in teh bundle
  * mixin
  * whatever normally was in the dockerfile lines output by build
  * credentials
    * docker username/password
    * cloud credentials
  * parameters
    * mixin config
    * bundle step
    * /cnab/app directory ! parameters can mount volumes
  * outputs
    * how to do dynamic outputs? just write to /cnab/app/outputs per usual?
  * actions = install, upgrade, uninstall, invoke


---

## Use Cases

> When I install two bundles that use the same version of Porter, and the same mixins, the second installation should go faster.

> I want smaller bundles.

> I want to distribute bundles using someone elses infrastructure, instead of figuring out how to host the binaries myself.

> I want to be able to verify that the mixins in a bundle came from their stated source.

> I want to write mixins that are based on different linux distributions.

> I want to limit which mixins can access bundle credentials. For example, only the az mixin should have access to my Azure credentials, not the exec mixin.

## Proposed Solution
Distribute mixins via bundles, using them as referenced OCI artifacts in the invocation image. The mixin binary becomes the bundle entry point.

### Pros
* Reduce size of invocation image.
* Optimized for layer caching.
* Reuse infrastructure for distribution and security.
* Firmer assurances of mixin provenance.
* Improved performance for build, publish and run.
* Builds on top of existing mixins.
* Demonstrates novel application of bundles and that we believe in using them ourselves.

### Cons
* Requires more development on something that mostly works today.
* Running mixins in a bundle is more complex.
* Needs changes to the CNAB Spec to avoid creating "porter-only" bundles.
