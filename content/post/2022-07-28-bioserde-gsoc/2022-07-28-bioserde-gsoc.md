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

The BioSerDe project, initially idealized in a [noodles issue](https://github.com/zaeleus/noodles/issues/53) [has been worked on][bioserde] by UMMCR in partnership with GA4GH's proposal's on GSOC.

The project was developed by [Gabriel Simonetto](https://github.com/GabrielSimonetto) with the mentorship of [Roman Valls Guimera](https://github.com/brainstorm), [Michael Milton](https://github.com/multimeric) and [Marko Malenic](https://github.com/mmalenic).

The mission is to have a safe and performant system to convert bioinformatics formats into alternative data representations.

## Deciding on this "Bioinformatics Rosetta Stone" rough shape

How do we execute on that goal? Is it possible to build a rosetta stone that transforms, say, a [BAM file into Arrow data stream relatively effortlessly][discussion-output-formats]?

In the first 2 months of duration of the project, various experiments have been made, we started with [initial attempts using protobuf files](https://github.com/umccr/BioSerDe/issues/2), but then quickly realized we would be wasting a lot of potential, since this approach would force all formats to conform to a row-based representation.

This then initiated [a discussion](https://github.com/umccr/BioSerDe/discussions/8) targeting which was the correct way to define said representation. From this discussion, we explored many different options ([1](https://github.com/umccr/BioSerDe/discussions/8#discussioncomment-2958136) [2](https://github.com/umccr/BioSerDe/pull/10) [3](https://github.com/umccr/BioSerDe/pull/11) [4](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=255a3600d1bf7b6935f5fe35a4354ccf)).

Ultimately, the current conclusion from all of these experiments was that it was necessary to modify noodles to accommodate some necessities of BioSerDe. This brought us back to [the issue that started it all on noodles](https://github.com/zaeleus/noodles/issues/53#issuecomment-1165222293). In the current status of said discussion, [Michael Macias](https://github.com/zaeleus/), author of Noodles, asks for a demonstration of how BioSerDe would use said changes in the noodles-bed crate as the simplest starting point among the VCF/BCF/SAM/BAM/CRAM bioinformatics format salad.

## Working with formats inside of noodles
Once we concluded we needed to fork noodles in order to unlock progress, we developed a plan to showcase how serde functionalities could be added to the crate.

The idea here is to create a `serde-json` representation of BED, as well as having a `Serializer` and a `Deserializer` for the regular BED formats, in order to take advantage of the serde ecosystem, which would allow for a [serde-transcode][serde-transcode] usage as a conversion between the 2 formats without needing to collect the entire input into an intermediate form in memory.

After some time [testing the possibilities](https://github.com/umccr/noodles/pull/1) with how the API and code should be defined, we arrived at some [serde solutions](https://github.com/umccr/noodles/pull/1#pullrequestreview-1048929529) that would help us to produce the desired behaviors in a clean way.

At this point, the usage of [derive macros](https://github.com/umccr/noodles/compare/07bd3d0b984157e39e3f09c2b40996f1f1725ff4...e0a892e55824d108e2e627b9b2a04bf008604d94#diff-59cc615206fcd8309d6a70dc24ad799cddab1b775ef3dc2d9945eb7db3786e63R28-R53) on top of the BED record struct [were already enough](https://github.com/umccr/noodles/compare/07bd3d0b984157e39e3f09c2b40996f1f1725ff4...e0a892e55824d108e2e627b9b2a04bf008604d94#diff-734a71c8c64780abff78dc27a3f49d2fff77e1b8d08270742e406c10b64d4979R7) to have a basic representation of BED in the json format.

All that was left to do, was to make a Serializer and Deserializer for the regular BED format. From this phase of the project onwards, there is one concern that will always be present in our decisions: how to reduce the duplication of code that comes from having parts of the behavior of the serialization and deserialization process already present in parts of the noodles codebase, namely, the [`Display`](https://github.com/zaeleus/noodles/blob/master/noodles-bed/src/record.rs#L468) and [`FromStr`](https://github.com/zaeleus/noodles/blob/master/noodles-bed/src/record.rs#L750) traits.

This gives us 2 options: using the noodles functionality inside the Serializer and Deserializer, or, recreating the functionality on the Serializer and Deserializer, and then, calling it on the original noodles entrypoints (Display calls the Serializer, instead of Serializer calling the Display). We initially went with the first idea, but later, some developtments will reveal that maybe we have to change our perspective.

Our initial research eventually found the [serde-with][serde-with] crate which has many functionalities, but most important to us, allows for a struct to [use its Display and FromStr](https://docs.rs/serde_with/2.0.1/serde_with/struct.DisplayFromStr.html) traits as serde behavior. This usage was very promising: it was suddenly very quick to implement serde [the plethora of BED definitions](https://github.com/umccr/noodles/blob/459015573c1c538f8b75cfa736f2876036f1c3d4/noodles-bed/src/de.rs#L126-L298) as each BED version can take care of it's own particularities. Even better is that `Display` and `FromStr` are well defined for the many noodles formats, which makes it so that this architecture will work well for them as well.

[At this point](https://github.com/umccr/noodles/pull/2), we had serde functionality for all BED formats, as well as the serde_json custom representation. With these basic behavior working, it was time to look into how would a conversion between 2 formats look like.

## Experimenting on conversions between two formats.
As mentioned before, we had the goal of using [serde-transcode][serde-transcode] as our main tool of conversion, since the BioSerDe project has an interest in being memory efficient between conversions. 

However, a quick inspection made us realize that [serde-transcode works by receiving only an `Serializer` and a `Deserializer`](https://github.com/umccr/noodles/pull/3/commits/67bbe84a2bb946796e464891e91e24346379d78f#diff-734a71c8c64780abff78dc27a3f49d2fff77e1b8d08270742e406c10b64d4979R40-R45) object, which is a problem, since our previous solution was reliant on passing a type annotation for serde-with to know which `Display` and `FromStr` implementation to use.

At this point, we had a decision to make: either, we drop serde-transcode, and make a custom function which uses internal memory allocation, in order to define a type that will be used in the process, or we need to go back on our architecture decision and find a way for the serializer and deserializer to produce BED valid entries inside the serialization and deserialization process, without the help of serde-with.

An extensive discussion on this problem and decision can be found [here](https://github.com/umccr/noodles/pull/3)

## Present
And this is how the project is going at the moment, we are currently searching for ways to either call upon noodles functionalities from within the serializers. Ideally, all parsing should be done using already existing code, and that should be possible by [tracking the state of serialization](https://github.com/umccr/noodles/pull/2#discussion_r949969398), and calling the appropriate functions at the right times.

## Future

The intention of this fork is to experiment how an **ergonomic SerDe implementation will fit within the intersection of Rust users and the bioinformatics community**.

We think that this approach is worth exploring because SerDe is a very well understood crate within the Rust ecosystem and so (growingly) does Noodles as a Rust alternative to htslib. We are also aware of the risks and upcoming challenges of taking this approach, namely: 

1. Not being merged nor supported upstream by Noodles: To be fully clear, **this is not a "hostile" type of fork** by any stretch of the imagination. We aim at BioSerDe being used by bioinformaticians and data scientists at large and this is done by building community not dividing it. Exploring the feasibility of embedding SerDe into Noodles helps us figure out limitations and drawbacks we can solve, refactor or reconsider later on: **Perhaps [serde-remote](https://serde.rs/remote-derive.html) is all we need for our usecase at the end?**. Or maybe we'll see a [fitting trait architecture at the end of this journey][chris-zen-traits], paving the way for future contributions?
1. When not used appropriatedly, [SerDe loads all bytes into memory][serde-streaming]. We know this is does not scale well within the multi-gigabyte bioinformatics file formats ecosystem. Instead, we need to convert between formats by streaming bytes from the underlying noodles structures. This could be eased by [serde-transcoding capabilities][serde-transcode] or any other attempts by third parties such as [tokio-serde][tokio-serde] or experiments with [postcard and async by James Munns][postcard-async] a format primarily designed for embedded targets.
1. It's still needed some experimentation as to which is the best way to merge noodles functionality with serde functionality, maybe there is still a way to use serde-with from inside the serializers. Or maybe even with the hardships of implementing more intricate Serializers, it's still the best way to end up with performatic code. After this architecture is defined, discussions around the future of the crate should be resumed.

## Join us

Interested in having your say? Reach us out at [BioSerDe] or [umccr-noodles][umccr-noodles] repos for discussion, issues and contributions!

[serde-transcode]: https://docs.rs/serde-transcode/latest/serde_transcode/index.html
[serde-with]: https://docs.rs/serde_with/latest/serde_with/
[tokio-serde]: https://twitter.com/braincode/status/1540662917749583872
[postcard-async]: https://twitter.com/bitshiftmask/status/1540683373554700288
[umccr-noodles]: https://github.com/umccr/noodles
[serde-streaming]: https://github.com/serde-rs/json/issues/345#issuecomment-323578757
[discussion-output-formats]: https://github.com/umccr/BioSerDe/discussions/5
[chris-zen-traits]: https://github.com/umccr/BioSerDe/discussions/8#discussioncomment-3031107
[bioserde]: https://github.com/umccr/BioSerDe