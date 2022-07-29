---
title: "GSoC 2022, GA4GH and BioSerDe"
authors:
  - gabriel-frank-simonetto
  - roman-valls-guimera
date: "2022-07-28"
slug: BioSerDe
layout: post
categories:
  - rust
  - BioSerDe
  - bioinformatics
  - serialization
tags:
  - serialization
  - rust
  - ga4gh
  - bioinformatics
  - gsoc
  - google
summary: "Google Summer of Code 2022 with GA4GH"
---

TL;DR: Check out [BioSerDe][bioserde] and its accompanying [SerDe experiments with noodles-bed][umccr-noodles].

![a_dna_sequencing_format_screaming_for_help_and_much_needed_change_by_salvador_dall_e](/img/2022/a_dna_sequencing_format_screaming_for_help_and_much_needed_change_by_salvador_dall-e.png)

# GSoC 2022 and BioSerDe

The BioSerDe project, initially idealized in a [noodles issue](https://github.com/zaeleus/noodles/issues/53) is [finally being worked on][bioserde] by UMMCR in partnership with GA4GH's proposal's on GSOC.

It's being currently worked on by [Gabriel Simonetto](https://github.com/GabrielSimonetto) with the mentorship of [Roman Valls Guimera](https://github.com/brainstorm), [Michael Milton](https://github.com/multimeric) and [Marko Malenic](https://github.com/mmalenic).

The mission is to have a safe and performant system to convert bioinformatics formats into alternative data representations.

## Deciding on this "Bioinformatics Rosetta Stone" rough shape

How do we execute on that goal? Is it possible to build a rosetta stone that transforms, say, a [BAM file into Arrow data stream relatively effortlessly][discussion-output-formats]?

In the 2 months of duration of the project, various experiments have been made, we started with [initial attempts using protobuf files](https://github.com/umccr/BioSerDe/issues/2), but then quickly realized we would be wasting a lot of potential, since this approach would force all formats to conform to a row-based representation.

This then initiated [a discussion](https://github.com/umccr/BioSerDe/discussions/8) targeting which was the correct way to define said representation. From this discussion, we explored many different options ([1](https://github.com/umccr/BioSerDe/discussions/8#discussioncomment-2958136) [2](https://github.com/umccr/BioSerDe/pull/10) [3](https://github.com/umccr/BioSerDe/pull/11) [4](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=255a3600d1bf7b6935f5fe35a4354ccf)).

Ultimately, the current conclusion from all of these experiments was that it was necessary to modify noodles to accommodate some necessities of BioSerDe. This brought us back to [the issue that started it all on noodles](https://github.com/zaeleus/noodles/issues/53#issuecomment-1165222293). In the current status of said discussion, [Michael Macias](https://github.com/zaeleus/), author of Noodles, asks for a demonstration of how BioSerDe would use said changes in the noodles-bed crate as the simplest starting point among the VCF/BCF/SAM/BAM/CRAM bioinformatics format salad.

## Present

And that is the step the project currently is in: for the last couple of weeks we have [forked noodles][umccr-noodles] on the ummcr organization in order to create the changes needed for this demonstration, with the first bits of code already being worked on ([5](https://github.com/umccr/noodles/pull/1) [6](https://github.com/umccr/BioSerDe/pull/14)). 

## Future

The intention of this fork is to experiment how an **ergonomic SerDe implementation will fit within the intersection of Rust users and the bioinformatics community**.

We think that this approach is worth exploring because SerDe is a very well understood crate within the Rust ecosystem and so (growingly) does Noodles as a Rust alternative to htslib. We are also aware of the risks and upcoming challenges of taking this approach, namely: 

1. Not being merged nor supported upstream by Noodles: To be fully clear, **this is not a "hostile" type of fork** by any stretch of the imagination. We aim at BioSerDe being used by bioinformaticians and data scientists at large and this is done by building community not dividing it. Exploring the feasibility of embedding SerDe into Noodles helps us figure out limitations and drawbacks we can solve, refactor or reconsider later on: **Perhaps [serde-remote](https://serde.rs/remote-derive.html) is all we need for our usecase at the end?**. Or maybe we'll see a [fitting trait architecture at the end of this journey][chris-zen-traits], paving the way for future contributions?
1. When not used appropriatedly, [SerDe loads all bytes into memory][serde-streaming]. We know this is does not scale well within the multi-gigabyte bioinformatics file formats ecosystem. Instead, we need to convert between formats by streaming bytes from the underlying noodles structures. This could be eased by [serde-transcoding capabilities][serde-transcode] or any other attempts by third parties such as [tokio-serde][tokio-serde] or experiments with [postcard and async by James Munns][postcard-async] a format primarily designed for embedded targets.
1. Unknown unknowns: we'll probably fill this up at the end of this GSoC edition with our learnings and insight.

Tons to explore and implement! We're excited to see the outcomes of the second GSoC 2022 term.

## Join us

Interested in having your say? Reach us out at [BioSerDe] or [umccr-noodles][umccr-noodles] repos for discussion, issues and contributions!

[serde-transcode]: https://docs.rs/serde-transcode/latest/serde_transcode/index.html
[tokio-serde]: https://twitter.com/braincode/status/1540662917749583872
[postcard-async]: https://twitter.com/bitshiftmask/status/1540683373554700288
[umccr-noodles]: https://github.com/umccr/noodles
[serde-streaming]: https://github.com/serde-rs/json/issues/345#issuecomment-323578757
[discussion-output-formats]: https://github.com/umccr/BioSerDe/discussions/5
[chris-zen-traits]: https://github.com/umccr/BioSerDe/discussions/8#discussioncomment-3031107
[bioserde]: https://github.com/umccr/BioSerDe