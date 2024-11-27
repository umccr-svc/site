---
title: "Using htsget-rs GA4GH's public endpoint"
authors:
  - marko-malenic
  - roman-valls-guimera
date: "2024-11-27"
slug: htsget-rs-complete
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

```
% curl "https://htsget.ga4gh-demo.org/reads/service-info"
```

Which should yield the following JSON, please open [a htsget spec](https://samtools.github.io/hts-specs/htsget.html) browser tab on the side while you follow up:

```json
{
  "id": "",
  "name": "GA4GH",
  "version": "0.1",
  "organization": {
    "name": "GA4GH",
    "url": "https://ga4gh.org/"
  },
  "type": {
    "group": "",
    "artifact": "",
    "version": ""
  },
  "htsget": {
    "datatype": "reads",
    "formats": [
      "BAM",
      "CRAM"
    ],
    "fieldsParametersEffective": false,
    "tagsParametersEffective": false
  },
  "contactUrl": "https://ga4gh.org/",
  "documentationUrl": "https://github.com/umccr/htsget-rs",
  "createdAt": "",
  "updatedAt": "",
  "environment": "dev"
}
```

Developers can now go and test the following endpoints with for BAM and other formats ([example files deployed from our htsget-рs data folder](https://github.com/umccr/htsget-rs/tree/main/data):

```sh
% curl "https://htsget.ga4gh-demo.org/reads/htsgetlambdastack-bucket83908e77-0bbbuwy4lrax/bam/htsnexus_test_NA12878"
```

Check out the Crypt4GH encrypted BAM payload example file:

```sh
% curl "https://htsget.ga4gh-demo.org/reads/htsgetlambdastack-bucket83908e77-0bbbuwy4lrax/c4gh/htsnexus_test_NA12878"
```

The payload can be decrypted with the test public/private keypair available on the official htsget-rs repo:

[https://github.com/umccr/htsget-rs/tree/main/data/c4gh][https://github.com/umccr/htsget-rs/tree/main/data/c4gh]

Please read the included `README.md` keygen/encrypt/decrypt walkthrough for more details on how to work with those files.

Our hope is that this public endpoint will facilitate much needed client implementations for Crypt4GH decryption, to name a few in no particular order:

* [htsjdk](https://github.com/umccr/igv/commit/c30e5a0aa7c5fc9cc914cb99dfcb28343995acb3)
* [IGV desktop](https://github.com/uio-bmi/crypt4gh/issues/85)
* [IGV.js](https://github.com/fathelen/crypt4ghJS)
* [JBrowse2](https://github.com/GMOD/jbrowse/issues/1142)

So this is a call library and client developers to test this htsget endpoint and [report back](https://github.com/umccr/htsget-rs/issues)!
