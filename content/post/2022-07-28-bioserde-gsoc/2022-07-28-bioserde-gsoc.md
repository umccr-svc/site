---
title: "GSoC 2022, GA4GH and BioSerDe"
authors: 
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

TL;DR: [https://github.com/umccr/BioSerDe][BioSerDe]

## GSoC 2022 and BioSerDe

The BioSerDe project, initially idealized in a [noodles issue](https://github.com/zaeleus/noodles/issues/53) is [finally being worked on](https://github.com/umccr/BioSerDe) by UMMCR in partnership with GA4GH's proposal's on GSOC.

It's being currently worked on by [Gabriel Simonetto](https://github.com/GabrielSimonetto) with the mentorship of [Roman Valls Guimera](https://github.com/brainstorm), [Michael Milton](https://github.com/multimeric) and [Marko Malenic](https://github.com/mmalenic).

In the 2 months of duration of the project, various experiments have been made, we started with [initial attempts using protobuf files](https://github.com/umccr/BioSerDe/issues/2), but then quickly realized we would be wasting a lot of potential, since this approach would force all formats to conform to a row-based representation.

This then initiated [a discussion](https://github.com/umccr/BioSerDe/discussions/8) targeting which was the correct way to define said representation. From this discussion, we explored many different options ([1](https://github.com/umccr/BioSerDe/discussions/8#discussioncomment-2958136) [2](https://github.com/umccr/BioSerDe/pull/10) [3](https://github.com/umccr/BioSerDe/pull/11) [4](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=255a3600d1bf7b6935f5fe35a4354ccf)).

Ultimately, the current conclusion from all of these experiments was that it was necessary to involve noodles modifications to accommodate some necessities of BioSerDe. This brought us back to [the issue that started it all on noodles](https://github.com/zaeleus/noodles/issues/53#issuecomment-1165222293). In the current status of said discussion, [Michael Macias](https://github.com/zaeleus/) asks for a demonstration of how BioSerDe would use said changes in the noodles crate

And that is the step the project currently is in, for the last couple of weeks we have [forked noodles](https://github.com/umccr/noodles) on the ummcr organization in order to create the changes needed for this demonstration, with the first bits of code already being worked on ([5](https://github.com/umccr/noodles/pull/1) [6](https://github.com/umccr/BioSerDe/pull/14)).