--
title: "AWS Rust Lambdas and Bioinformatics"
author: Roman Valls Guimera 
date: "2021-03-31"
slug: aws-bioinformatics-rust 
layout: post
permalink: https://umccr.org/blog/2020/12/20/aws-bioinformatics-rust
categories:
  - rust
tags:
  - rse
  - rust
  - ga4gh
  - bioinformatics
  - gsoc
  - google
summary: "Google Summer of Code 2021 with the Global Alliance for Genomics and Health, implementing htsget in Rust, Noodles and AWS Lambdas"
---

# Yet another AWS Lambda and Rust blogpost?

At this point in time, much has been written about AWS and Rust Lambdas, here's a personal selection in no particular order:
[0][aws_lambda_rust_article_0]
[1][aws_lambda_rust_article_1]
[2][aws_lambda_rust_article_2]
[3][aws_lambda_rust_article_3]
[4][aws_lambda_rust_article_4]
[5][aws_lambda_rust_article_5]
[6][aws_lambda_rust_article_6]
[7][aws_lambda_rust_article_7]

This writeup patches most of those blogposts with a couple of recent developments and proposes a cool (paid) project to work on if the reader is keen:

1. The latest, first officially released on crates.io, [aws-lambda-rust-runtime][aws_lambda_rust_runtime_github] **version 0.3.0** and why you don't need MUSL in your lambda builds anymore. **If you are interested in quickly adopting those changes, read the "New lambda runtime" section below.**
1. Proposes an interesting BioIT project **for students**, that uses the tech described in this article. This is **paid by Google** if you are accepted on this year's [GSoC htsget-rs idea under the GA4GH organization][ga4gh_project_idea].
1. On the bioinformatics front, we outline a serialization-deserialization (BioSerDe?) future where file formats get abstracted to allow easier experimentation with different storage and compute backends.

So, without further ado, what's new in the AWS Rust serverless space?

## New AWS lambda runtime

Released just a few weeks ago by a relentless effort from (mostly, among others) [@bahildebrand][bahildebrand] and [@coltonweaver][coltonweaver], [release 0.3.0][lambda_rust_release_0_3] most likely breaks some of the examples from the blogposts listed above in the introduction.

While the official documentation for this crate was updated accordingly, you might want to clone [s3-rust-noodles-bam as a testbed, since it includes the improvements described in this blogpost][s3-rust-noodles-bam] **and** the application's payload is hopefully more interesting [than a hello world][time4tea_hello_world].

### Letting go of MUSL static builds

One of the ugly truths of AWS lambdas is that the [underlying "provided" Amazon Linux v1 build is too old][glibc_errors_rust_lambda] to link against more recent GLIBCs, often yielding errors like:

```shell
(...)
/var/task/bootstrap: /lib64/libc.so.6: version `GLIBC_2.18' not found (required by /var/task/bootstrap)
(...)
ms	Billed Duration: 100 ms 	Memory Size: 128 MB	Max Memory Used: 12 MB	
```

Until the [AWS lambda `provided.al2` was released][provided_al2_lambda], the workaround was to build the resulting `bootstrap` (lambda payload binary) with MUSL. This means statically linking your payload and all dependencies, inflating the resulting binary and [generating many headaches on the integration and deployment land][musl_lambda_nightmare].

No more.

Just use `provided.al2` and target `x86_64-unknown-linux-gnu` and [jemallocator][jemallocator]. The latter just requires a couple of simple changes on your `Cargo.toml` and `main.rs`, [refer to the docs][jemallocator].

So now we have dynamically linked GLIBC for `x86_64` with an optimized memory allocator. That's at least until AWS lambdas don't run on Graviton2 (arm64)? Also, speaking of Lambda deployment...

### Embrace AWS SAM...

... while AWS fixes its [reported Rust-specific developer experience issues][rust_aws_sam_dx].

Until then, [TMTOWTDI][TMTOWDI] for AWS Rust Lambdas: Manually upload the resulting zipped `bootstrap`, Terraform, [CDK][lambda_cdk_typescript], serverless.com and [AWS's SAM][aws_sam], so pick and choose.

After testing a few of those approaches on [s3-rust-htslib-bam][s3-rust-htslib-bam], we'll be [sticking with SAM since its tooling is meant to integrate well with AWS][aws_lockin].

Having the updates and gotchas covered, let's talk about how to use this great tech next?

## Motivation

Shortly after announcing that [this project idea was up for grabs][ga4gh_project_idea], a few voices pointed out some good questions:

> There is already a Golang implementation of htsget IIRC, and so one for C4GH: So building this in go could be easier, although perhaps does not live up to performance expectations?

> I would like to know the reasons for choosing rust for this task primarily. I am a fan of the language but given what’s out there now it feels a bit of an overkill?

Here's a few reasons we think that justify the time and effort to implement this project:

1. Multiple independent implementations help establish standards and allow us to demonstrate interoperability; it also shows where standards/references are ambiguous or underspecified.
1. While optimizing the implemention of this project we will most probably also help related Rust (crates), nourishing its ecosystem as a result. **Diverse software options are important and sorely needed in our field**, htslib has been central to Bioinformatics for many years now [and htsjdk has its own issues too][htsjdk_no_cram_threading].
1. UMCCR's [@victorskl][victorskl] pull-requested an [Amazon-specific AWS-Go-SDK integration][cors_htsget_refserver_pullrequest] for the GA4GH's Go implementation, which is currently under scrutiny. Perhaps that pullrequest will never be merged to stay on principle: **a reference implementation should be platform-agnostic**. On the other hand, this Rust project is highly opinionated and does not have that particular limitation.
1. The [reference implementation just shells out to `samtools` under the hood][htsget_go_refserver_samtools]. This means that it still uses `htslib`, with integration issues such as the one noted above.
1. Safety, [see comments about OpenSSL and htslib][openssl_htslib_safety]. many static analysis via fuzzers reveal memory leaks and other defects found routinely on htslib this way. A pure Rust implementation does not have to go through those audits because the compiler guarantees a series of safety measures against those defects.
1. Performance vs Go argument? See a well written, reality-based and pragmatic justification here: https://thenewstack.io/rust-vs-go-why-theyre-better-together/?s=09

To be fair, regarding performance, [there's both Fargate and Lambda supporting docker images now][aws_lambda_vs_fargate], which probably provides a more than good enough solution?

Since towards the end of this project **we will run some benchmarks and see what makes sense to use in different regimes**.


# Google Summer of Code 2021 and htsget-rs

The GA4GH (Genomics Alliance for Genomics and Health) was accepted as an organization [for this year's (GSoC) Google's Summer of Code][ga4gh_gsoc_landing_page].

## Expectations

Since Rust libraries in our field are currently developing, this project has a strong community component. In our proposal we will inevitably be touching a few crates from the Rust ecosystem:

1. Noodles
1. AWS Rust Lambda Runtime
1. Rust-S3 (as a **temporal** replacement to Rusoto).
1. RustLS (as a **permanent** replacement to OpenSSL).

[Unexpected issues might crop up on the AWS side][aws_tooling_flukes] or elsewhere. This means that **the student must be prepared to fix third party dependencies and thus interact with different crate authors and communities**, with some help from the mentors if need be.

There's also some minor onboarding particularities for this project, namely having access to an Amazon Web Services account: **Our team at UMCCR can provide temporal access to a limited AWS account for this particular project**. In addition, the successful student will be added to GA4GH's `#htsget` Slack channel as well as pointers to [the broader Rust community][rust-community].

## Architecture and implementation plan

As outlined in the official [htsget specification][htsget_spec], only two API endpoints are required: `reads` and `variants` (we are obviating ticket handling for simplicity in this writeup):

![htsget-user-interaction](/img/2021/htsget-rs-user-interaction.png)

[GA4GH already has a reference htsget implementation written in Go][ga4gh-refserver], partially based on an previous [older Google-specific implementation][google-htsget].

This presents itself as a huge advantage since we can examine parts of it like [the endpoints defined by the ga4gh-refserver implementation and build the aforementioned mock testing outlined above][ga4gh-refserver-endpoints]. Many other functional groups that conform a htsget implementation are already very well laid out:

![htsget-refserver-impl](/img/2021/htsget-go-refserver-classes.png)

That being said, Rust not only has different syntax, module structure and other primitives than Go, but it also has [its own supported ways to structure projects via Cargo workspaces][cargo-workspaces].

For instance, Noodles itself uses [cargo workspaces][cargo-workspaces] to modularise and compose different file format structures and their associated functions:

```
[workspace]
members = [
  "noodles",
  "noodles-bam",
  (...)
  "noodles-sam",
  "noodles-tabix",
  "noodles-vcf",
]
```

Furthermore, since [Michael][zaeleus] had in mind a Rust implementation of htsget when he started Noodles, there are functions worth keeping in mind that will prove useful while implementing the **low level details of parsing a BAI file and optimizing byte ranges**, see [optimize_chunks][noodles_optimize_chunks]. This detail will come handy later on during the implementation, but it's not important right now, so don't worry if it's not clear at the moment.

When a local implementation works it'll be time to leverage the work already done on the newest [Noodles+Rust-S3+AWS+Lambda example][s3-rust-noodles-bam]. Once this stage is reached, it is important to remember the system description graphic above and realise that the scope for our implementation deals with **S3 public buckets** since all the authZ/authN happens on the AWS API Gateway. In other words, **we will not be writing code that validates JWT tokens ourselves since that is already managed by AWS's API Gateway**.

[The Go htsget reference implementation decouples security from core application functionality][cors_htsget_refserver_pullrequest]. In this implementation, we'll also do that but assume that data is fronted by a managed service such as API Gateway.

Last but not least, [benchmarking or "benches"][firestorm] needs to be run to assess bottlenecks [and aim at future improvements iteratively][cargo_bench_cmp].

Ready to start writing some Rust with us for GA4GH's GSoC? Jump to the next section!

## Qualifying task

Implementing the following task, **while not mandatory, gives extra points** to candidates since it helps mentors with the student selection process.

The first step the student should take is implementing a local (no AWS involved) htsget interface. This step is important to accomplish while we review proposals since it gives mentors and indication of:

1. How resourceful the incumbent is when carrying on API development.
1. Which code structure is followed to attain this goal? Do they constitute good use of abstractions, data structures and overall code quality?
1. [Which questions are being asked][smart_questions] to mentors and how thoughtful, precise are those?
1. Does the student manage version control and collaborative development successfully?

The task would be to understand, code and pullrequest a substantial constribution to this skeleton repo:

https://github.com/chris-zen/htsget-mvp

Good luck and see you soon! :)

# Future

TL;DR: [EVERYTHING IS SERIALIZATION][everything_serialization]

**This section is out of scope for GSoC**, but outlines a few exciting followup experiments that could materialize in the future:

1. Generalize Robert's Aboukhalil [WASM+CloudFlare][wasm_noodles_cloudflare] experiment. Use htsget as comms layer?
1. Think forward about (de)serializing our niche bioinformatics formats to other [supported formats by the Rust SerDe crate][custom_serializers]. There is an ongoing approach by Mike Linn [SQLite Genomics][genomics-sqlite] which is a particular case of a more general model, as in BAM to: Arrow, [Presto UDFs][presto-udf] or any destination format used by data analysis frameworks outside bioinformatics.
1. Use those serializers mentioned above in [other tooling such as more genome visualization apps][awesome_genome_viz] or even into [Radare2][radare2_bioinformatics] so that other communitites and fields can easily interact with genomics data.
1. Interface with the [ADAM ecosystem][ADAM], perhaps combining with [vega][vega], formerly known as [native-spark][fastspark].


Thanks to [@chris-zen][chris-zen], [@victorskl][victorskl], [@zaeleus][zaeleus] and the UMCCR team for proofreading and suggesting improvements for this blogpost.

[aws_lambda_rust_runtime_github]: https://github.com/awslabs/aws-lambda-rust-runtime
[custom_serializers]: https://serde.rs/custom-serialization.html
[presto-udf]: https://aws.amazon.com/about-aws/whats-new/2019/11/amazon-athena-adds-support-for-user-defined-functions-udf/.
[genomics-sqlite]: https://github.com/mlin/GenomicSQLite
[everything_serialization]: https://rustfest.global/session/9-everything-is-serialization/
[vega]: https://github.com/rajasekarv/vega
[fastspark]: https://medium.com/@rajasekar3eg/fastspark-a-new-fast-native-implementation-of-spark-from-scratch-368373a29a5c
[aws_lambda_rust_article_0]: https://aws.amazon.com/blogs/opensource/rust-runtime-for-aws-lambda/
[aws_lambda_rust_article_1]: http://jamesmcm.github.io/blog/2020/10/24/lambda-runtime/#en
[aws_lambda_rust_article_2]: https://beanseverywhere.xyz/blog/rust-lambda
[aws_lambda_rust_article_3]: https://adevait.com/rust/deploying-rust-functions-on-aws-lambda
[aws_lambda_rust_article_4]: https://andre.arko.net/2018/10/25/parsing-logs-230x-faster-with-rust/
[aws_lambda_rust_article_5]: https://silentbyte.com/writing-aws-lambda-functions-in-rust
[aws_lambda_rust_article_6]: https://blog.knoldus.com/aws-lambda-with-rust/
[aws_lambda_rust_article_7]: https://dev.to/rad_val_/aws-lambda-rust-292g
[lambda_rust_release_0_3]: https://github.com/awslabs/aws-lambda-rust-runtime/releases/tag/v0.3.0
[s3-rust-noodles-bam]: https://github.com/umccr/s3-rust-noodles-bam
[provided_al2_lambda]: https://aws.amazon.com/blogs/compute/migrating-aws-lambda-functions-to-al2/
[glibc_errors_rust_lambda]: https://github.com/awslabs/aws-lambda-rust-runtime/issues/17
[musl_lambda_nightmare]: https://github.com/rust-bio/rust-htslib/pulls?q=is%3Apr+MUSL
[TMTOWDI]: http://acronymsandslang.com/definition/217417/TMTOWDI-meaning.html
[bahildebrand]: https://github.com/bahildebrand
[coltonweaver]: https://github.com/coltonweaver
[jemallocator]: https://lib.rs/crates/jemallocator
[s3-rust-htslib-bam]: https://github.com/brainstorm/s3-rust-htslib-bam
[aws_lockin]: https://www.lastweekinaws.com/blog/the-lock-in-you-dont-see/
[rust_aws_sam_dx]: https://github.com/aws/aws-lambda-builders/pull/174#issuecomment-698110442
[lambda_cdk_typescript]: https://github.com/brainstorm/s3-rust-htslib-bam/pull/2/files#diff-6f87698966b1a30bfbd0189d7b58e0e1021f4299137d395a9f84dc0ef48a695b
[aws_sam]: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html
[ga4gh_project_idea]: https://twitter.com/braincode/status/1369851695442751492?s=20
[ga4gh_gsoc_landing_page]: https://summerofcode.withgoogle.com/organizations/5907083486035968/
[aws_tooling_flukes]: https://github.com/umccr/s3-rust-noodles-bam/blob/master/FLUKES.md
[cors_htsget_refserver_pullrequest]: https://github.com/ga4gh/htsget-refserver/pull/24
[rust-community]: https://www.rust-lang.org/community
[htsget_spec]: https://samtools.github.io/hts-specs/htsget.html
[smart_questions]: http://catb.org/~esr/faqs/smart-questions.html
[http_rust_mock]: https://github.com/search?q=rust+http+mock
[aws_lambda_vs_fargate]: https://stackoverflow.com/questions/65370498/aws-lambda-container-image-support-vs-fargate
[openssl_htslib_safety]: https://github.com/umccr/s3-rust-noodles-bam#read-bam-header-on-an-aws-lambda-with-noodles
[htsjdk_no_cram_threading]: https://github.com/hartwigmedical/gridss-purple-linx/issues/30#issuecomment-809819107
[victorskl]: https://github.com/victorskl
[chris-zen]: https://github.com/chris-zen
[zaeleus]: https://github.com/zaeleus/
[google-htsget]: https://github.com/googlegenomics/htsget
[ga4gh-refserver]: https://github.com/ga4gh/htsget-refserver
[ga4gh-refserver-endpoints]: https://github.com/ga4gh/htsget-refserver/blob/develop/internal/htsconstants/endpoints.go
[cargo-workspaces]: https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html
[noodles_optimize_chunks]: https://github.com/zaeleus/noodles/blob/0e5af40a786e9b23672552ae8b7ebfc6fd13c460/noodles-bam/src/bai.rs#L144
[radare2_bioinformatics]: https://radareorg.github.io/blog/posts/radare2-bioinformatics/
[bcl2fastr]: https://github.com/czbiohub/bcl2fastr/
[illumina_ora]: https://www.illumina.com/informatics/sequencing-data-analysis/sequence-file-formats.html
[cargo_bench_cmp]: https://github.com/BurntSushi/cargo-benchcmp
[firestorm]: https://github.com/That3Percent/firestorm
[wasm_noodles_cloudflare]: https://twitter.com/RobAboukhalil/status/1374067575173251075
[ADAM]: https://github.com/bigdatagenomics/adam
[awesome_genome_viz]: https://github.com/cmdcolin/awesome-genome-visualization
[htsget_go_refserver_samtools]: https://github.com/ga4gh/htsget-refserver/blob/bd3fa58b20377c5151cfa33e45d48ceeb49f28bb/internal/htsserver/getreadsdata.go#L153
[time4tea_hello_world]: https://github.com/time4tea/rust-lambda
