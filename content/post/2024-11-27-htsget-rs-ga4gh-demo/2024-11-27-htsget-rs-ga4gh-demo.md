---
title: "Using htsget-rs GA4GH's public endpoint"
authors:
  - marko-malenic
  - roman-valls-guimera
date: "2024-11-27"
lastmod: "2025-03-17"
slug: htsget-rs-crypt4gh
layout: post
categories:
  - rust
  - bioinformatics
  - htsget
tags:
  - rust
  - ga4gh
  - bioinformatics
summary: "htsget-rs demo up and running at GA4GH"
---
# GA4GH htsget+Crypt4GH demo

Welcome to the first [htsget-rs](https://github.com/umccr/htsget-rs) public instance with Crypt4GH support, hosted by GA4GH proper. Let's begin with a simple `curl` command:

```sh
curl "https://htsget.ga4gh-demo.org/reads/service-info"
```

Which should yield the following JSON, please open [a htsget spec](https://samtools.github.io/hts-specs/htsget.html) browser tab on the side while you follow up:

```json
{
  "createdAt": "2025-02-13T21:50:40.515058363+00:00",
  "version": "0.6.0",
  "organization": {
    "name": "",
    "url": ""
  },
  "updatedAt": "2025-02-13T21:50:40.515060644+00:00",
  "documentationUrl": "https://github.com/umccr/htsget-rs",
  "id": "htsget-lambda/0.6.0",
  "name": "htsget-lambda",
  "description": "A cloud-based instance of htsget-rs using AWS Lambda, which serves data according to the htsget protocol.",
  "type": {
    "group": "org.ga4gh",
    "artifact": "htsget",
    "version": "1.3.0"
  },
  "htsget": {
    "datatype": "reads",
    "formats": [
      "BAM",
      "CRAM"
    ],
    "fieldsParametersEffective": false,
    "tagsParametersEffective": false
  }
}
```

Developers can now go and test the following endpoints with BAM and other formats ([example files deployed from our htsget-rs data folder](https://github.com/umccr/htsget-rs/tree/main/data)):

```sh
curl "https://htsget.ga4gh-demo.org/reads/htsnexus_test_NA12878"
```

Check out the Crypt4GH encrypted BAM payload example file, note the **experimental** `encryptionScheme=` parameter, [subject to change and in active discussion by the GA4GH htsget committee](https://github.com/samtools/hts-specs/pull/808):

```sh
curl "https://htsget.ga4gh-demo.org/reads/htsnexus_test_NA12878?encryptionScheme=C4GH"
```

The payload can be decrypted with the test public/private keypair available on the official htsget-rs repo:

[https://github.com/umccr/htsget-rs/tree/main/data/c4gh](https://github.com/umccr/htsget-rs/tree/main/data/c4gh)

Please read the included `README.md` keygen/encrypt/decrypt walkthrough for more details on how to work with those files.

CRAM files can be queried by specifying `?format=CRAM` on the reads endpoint:

```sh
curl "https://htsget.ga4gh-demo.org/reads/htsnexus_test_NA12878?format=CRAM"
```

And the equivalent Crypt4GH encrypted CRAM:

```sh
curl "https://htsget.ga4gh-demo.org/reads/htsnexus_test_NA12878?format=CRAM&encryptionScheme=C4GH"
```

Similar to the `/reads` endpoint, `/variants` is also supported for VCF and BCF files:

```sh
curl "https://htsget.ga4gh-demo.org/variants/service-info"
```

For example, to fetch the VCF example file, try:

```sh
curl "https://htsget.ga4gh-demo.org/variants/spec-v4.3"
```

Or, the same Crypt4GH encrypted VCF can be fetched with:

```sh
curl "https://htsget.ga4gh-demo.org/variants/spec-v4.3?encryptionScheme=C4GH"
```

BCF files can be obtained by specifying `?format=BCF` on the variants endpoint. To get the example BCF:

```sh
curl "https://htsget.ga4gh-demo.org/variants/sample1-bcbio-cancer?format=BCF"
```

Or, the Crypt4GH encrypted BCF:

```sh
curl "https://htsget.ga4gh-demo.org/variants/sample1-bcbio-cancer?format=BCF&encryptionScheme=C4GH"
```

All the queries above that fetch data (excluding the service-info endpoint) support parameters which can restrict the
regions returned.

For example, to query a specific reference name with start and end positions, try:

```sh
curl "https://htsget.ga4gh-demo.org/reads/htsnexus_test_NA12878?referenceName=1&start=1000&end=2000"
```

Or, with the variants endpoint:

```sh
curl "https://htsget.ga4gh-demo.org/variants/spec-v4.3?referenceName=20&start=0&end=100"
```

This also works with Crypt4GH files, and it means that only specific regions that match the query will be returned:

```sh
curl "https://htsget.ga4gh-demo.org/reads/htsnexus_test_NA12878?referenceName=1&start=1000&end=2000"
```

Or, with the Crypt4GH VCF file:

```sh
curl "https://htsget.ga4gh-demo.org/variants/spec-v4.3?referenceName=20&start=0&end=100&encryptionScheme=C4GH"
```

Our hope is that this public endpoint will facilitate much needed client implementations for Crypt4GH decryption, to name a few in no particular order:

* [htsjdk](https://github.com/umccr/igv/commit/c30e5a0aa7c5fc9cc914cb99dfcb28343995acb3)
* [IGV desktop](https://github.com/uio-bmi/crypt4gh/issues/85)
* [IGV.js](https://github.com/fathelen/crypt4ghJS)
* [JBrowse2](https://github.com/GMOD/jbrowse/issues/1142)

So this is a call for library and client developers to test this htsget endpoint and [report back](https://github.com/umccr/htsget-rs/issues)!
