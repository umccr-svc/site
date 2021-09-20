---
title: "GSoC 2021, GA4GH and htsget-rs"
authors: 
  - roman-valls-guimera
date: "2021-07-28"
slug: htsget-rs
layout: post
permalink: https://umccr.org/blog/2021/07/28/htsget-rs
categories:
  - rust
  - htsget
  - bioinformatics
  - infrastructure
tags:
  - rse
  - rust
  - ga4gh
  - bioinformatics
  - gsoc
  - google
summary: "Google Summer of Code 2021 with GA4GH: A pure Rust htsget implementation is born"
---

TL;DR: [https://github.com/umccr/htsget-rs][htsget_rs]

## GSoC 2021 and htsget-rs

The [Rust htsget server][htsget_rs] that was sketched [a few months ago at UMCCR][aws_rust_lambdas_bioinfo] has been put together thanks to GA4GH's [generous allocation of 2 students to work on this effort][GSoC2021] to nourish the Rust bioinformatics ecosystem while delivering a GA4GH driver technology implementation [such as the htsget protocol][htsget_protocol].

After a bit of a struggle finding suitable candidates:

<!-- https://twitter.com/braincode/status/1379773467604647936?s=20 -->
{{< tweet 1379773467604647936 >}}

The work carried by [Daniel Castillo de la Rosa][castillodel] and [Marko Malenic][mmalenic1] with the mentoring supervision of [Christian Pérez-Llamas][chris-zen], [Victor San Kho Lin][victorskl], [Michael Macias][zaeleus] and [Roman Valls Guimera][brainstorm] (myself) has [been published in this repo and it is ready to use][htsget_rs] 🎉

To sum up, this project has:

1. Implemented all major bioinformatic formats that htsget supports: VCF, BCF, BAM and CRAM... along with their corresponding indices. Non-bioinfo savvy readers: [get acquainted with some of those formats here](https://www.youtube.com/watch?v=MrVpn0vpIYU).
1. Tested and reported bugs to a crucially important underlying Rust library: [Noodles][noodles].
1. Added a [local htsget (http) server](https://github.com/umccr/htsget-rs/blob/main/htsget-http-actix/README.md) that can be spawned for testing or on-premise deployment purposes.
1. Created a [benchmark suite to spot performance regressions and compare against other third party implementations][benchmarks_pr].
1. Code reviewed implementations, documentation on architecture and operation, including functional and integration tests.
1. Prompted third party genome viewers to [properly support GA4GH's htsget specification in their implementations](https://github.com/igvteam/igv/pull/850).

Also, to be clear, there's work to be finished up:

1. Proper local testing and deployment **for AWS lambdas**. Unfortunately there's [a chain of upstream AWS dependencies that need to be fixed first][aws-sam-cli-local-lambda]. To clarify, Rust lambdas can be deployed with SAM but not tested locally, which makes local development/deploy iterations too cumbersome to be practical at the time of writing this.
1. [S3 storage backend, supporting "non-immediately" accessible storage tiers such as AWS Glacier or Deep Archive](https://github.com/umccr/htsget-rs/issues/9).
1. [DRS compatible object ID resolver](https://www.ga4gh.org/news/drs-api-enabling-cloud-based-data-access-and-retrieval/), to improve integration of other GA4GH-standardized information retrieval mechanisms.

And future avenues for improvement are (but not limited to):

1. Other cloud-native deployments to major public cloud vendors such as: Cloudflare, Google, Azure, etc...
1. Other (cloud-native?) storage backends such as: Minio.
1. Profiling and optimization: [cargo-flamegraphs](https://github.com/flamegraph-rs/flamegraph), [cargo-instruments](https://github.com/cmyr/cargo-instruments), etc...
1. Additional http server implementations such as [Warp](https://docs.rs/warp/0.3.1/warp/), now only [actix-web](https://actix.rs/) is implemented but others can be built on top of the [http-core abstraction](https://github.com/umccr/htsget-rs/tree/main/htsget-http-core).

Now, if you want to see more [nitty-gritty details and ways to contribute][architecture_md], keep reading and see how Marko and Daniel implemented several of htsget-rs underpinnings.

## Marko Malenic

The ["qualifying task"][qual_tasks] chosen by Marko was in the lines of the `class` attribute reflected in the htsget spec and [he did a good job implementing it][class_attr_pr].

Marko took the initial [BAM search module][blocking_bam_search] as a reference and implemented the [CRAM search backend][blocking_cram_search]. After encompassing some necessary changes in the local files storage backend [such as functionality to return object sizes](https://github.com/umccr/htsget-rs/commit/dc1b2950ba17b8bf1a898f3a27f8160368d8d246), we realised that there was a significant amount of code replication across formats: at the end of the day, we search bytes on every format, so [there's a fair share of commonalities](https://github.com/umccr/htsget-rs/commit/f4a23435345dfe4e12cf13150ae8f1364aafbc0a).

Last but not least, [as Noodles was getting support for async methods](https://github.com/zaeleus/noodles/issues?q=is%3Aissue+is%3Aclosed+author%3Ammalenic), so did [htsget-rs incorporate its async facets](https://github.com/umccr/htsget-rs/pull/56). This is the reason why Marko separated [async code from the blocking code](https://github.com/umccr/htsget-rs/tree/main/htsget-search/src/storage), so that Daniel could easily benchmark both implementations and measure its underlying impact. 

For a more detailed list of the particular work units and commits, [there's a gist for that](https://gist.github.com/mmalenic/b30550b216d6461ca24001d8cba9893b).

## Daniel Castillo de la Rosa

Daniel hit the ground running on the quals by filing a [PR with the remaining htsget query builder methods implemented](https://github.com/umccr/htsget-rs/pull/21) and then [some tests for the local storage backend](https://github.com/umccr/htsget-rs/pull/22).

Starting with the already existing BAM search support, Daniel gained greater understading about BAI (BAM index) VirtualOffsets and managed to [optimise htsget's returned ranges accordingly](https://github.com/umccr/htsget-rs/pull/35). On top of that, during his main work with [htsget-rs][htsget_rs], he [found several issues](https://github.com/zaeleus/noodles/issues?q=is:issue+is:closed+author:CastilloDel) in the underlying library, [Noodles][noodles], that will definitely help other bioinformatics developers using it in the future. We would like to thank our co-mentor [Michael Macias, a.k.a @zaeleus][zaeleus], for his light-speed fixing of the reported bugs and implementation of new features that made this project possible.

After BAM was established and well tested, Daniel moved into implementing [VCF](https://github.com/umccr/htsget-rs/pull/37) and its compressed/binary counterpart, [BCF](https://github.com/umccr/htsget-rs/pull/43) while its [support was implemented in Noodles](https://github.com/zaeleus/noodles/issues/13) and engaging with [other Noodle users along his coding path](https://github.com/zaeleus/noodles/issues/26).

With all major bioinformatics formats that htsget supports in its spec implemented, Daniel went through the HTTP side of htsget: [adding the http-actix-web HTTP server and an abstraction, http-core for other HTTP servers](https://github.com/umccr/htsget-rs/pull/45), the [GA4GH service-info endpoint](https://github.com/umccr/htsget-rs/pull/54), which gives a birds-eye-view on what an htsget endpoint provides and its documentation.

Furthermore, htsget needs some middleware that disambiguates names and locations of particular datasets, and that's what Daniel fixed with his [regex-based id-resolver](https://github.com/umccr/htsget-rs/pull/60), as the GA4GH reference implementation does. In the future, other data/id resolver mechanisms should be implemented for better interoperability with different research institutes, industry and other htsget users at large.

Lastly, Daniel is working on [adding benchmarks](https://github.com/umccr/htsget-rs/pull/59) with the excellent [criterion-rs crate](https://bheisler.github.io/criterion.rs/book/index.html), which provides a good base for comparing our implementation with itself (across code changes) and against other third party implementations.

<!-- TBD: Also, there's a gist for all of that described above ;) -->

## Future

Other than deploying htsget-rs on your own usecase, here's some unassorted TODOs and ideas that can be explored:

1. [Implement Crypt4GH gotchas](https://github.com/umccr/htsget-rs/issues/34).
1. Implement other ID resolvers, possibly integrating well with DRS.
1. Implement other Storage backends, ideally leading to a BioSerDe crate, where all common cloud native formats are supported out of the box: Parquet, ORC, json.gz... to be able for compare against more custom designed storage representations such as pVCF, SAV and other upcoming formats that are proposed at GA4GH but that have relatively poor compatibility with the rest of the (big data) ecosystem and tooling found in today's commercial cloud systems.

One thing that prevents wider cloud-native adoption is formats that are not cloud-native:

1. Big BAM/CRAM files throttle the througput of S3 object store transfers. Using more object-storage friendly storage backend could work much better with htsget and other bioinformatic analysis pipelines. 
1. Where does htsget fit in the greater picture as a middleware between DRS, clients and other third party programs and hardware such as Illumina's DRAGEN and ICA service?

[htsget_protocol]: https://samtools.github.io/hts-specs/htsget.html
[aws_rust_lambdas_bioinfo]: https://umccr.org/blog/aws-bioinformatics-rust/
[GSoC2021]: https://github.com/umccr/htsget-rs/projects/1
[htsget_rs]: https://github.com/umccr/htsget-rs
[noodles]: https://github.com/zaeleus/noodles
[noodles_bugreports_daniel]: https://github.com/zaeleus/noodles/issues?q=is%3Aissue+is%3Aclosed+label%3Abug+author%3Acastillodel
[noodles_bugreports_marko]: https://github.com/zaeleus/noodles/issues?q=is%3Aissue+is%3Aclosed+label%3Abug+author%3Ammalenic+
[bcf_support]: https://github.com/zaeleus/noodles/issues/13
[async_pr]: https://github.com/umccr/htsget-rs/issues/16
[benchmarks_pr]: https://github.com/umccr/htsget-rs/pull/59
[igv_htsget_fixes]: https://github.com/igvteam/igv/issues/983
[rusty_aws_sam_cli]: https://github.com/aws/aws-sam-cli/issues/3132
[crypt4gh_htsget]: https://github.com/umccr/htsget-rs/issues/34
[castillodel]: https://github.com/CastilloDel
[mmalenic1]: https://github.com/mmalenic
[victorskl]: https://github.com/victorskl
[chris-zen]: https://github.com/chris-zen
[zaeleus]: https://github.com/zaeleus/
[brainstorm]: https://github.com/brainstorm
[aws-sam-cli-local-lambda]: https://github.com/aws/aws-sam-cli/issues/3132
[blocking_bam_search]: https://github.com/umccr/htsget-rs/blob/e83f599a9a36b1cf3b17ab971cb240149c4f31a5/htsget-search/src/htsget/bam_search.rs
[blocking_cram_search]: https://github.com/umccr/htsget-rs/commit/021346f43f527857e81299cd99324564ed8939d2#diff-37f6403ba06f890e39243bc46ffa8317f48a59a715e4a6eec608e85bd8c13a39
[class_attr_pr]: https://github.com/umccr/htsget-rs/pull/19
[gsoc_outreach]: https://twitter.com/braincode/status/1379773467604647936?s=20
[qual_tasks]: https://twitter.com/braincode/status/1380487973645414405?s=20
[architecture_md]: https://github.com/umccr/htsget-rs/blob/main/doc/ARCHITECTURE.md
