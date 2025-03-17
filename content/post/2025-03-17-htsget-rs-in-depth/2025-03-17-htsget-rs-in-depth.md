---
title: "htsget-rs in depth"
authors:
  - marko-malenic
  - roman-valls-guimera
date: "2025-03-17"
slug: htsget-rs-in-depth
layout: post
categories:
  - rust
  - bioinformatics
  - htsget
tags:
  - rust
  - ga4gh
  - bioinformatics
summary: "In depth htsget-rs and the htsget protocol"
---
# The details of the htsget protocol using htsget-rs and Crypt4GH

Following on from the [first] blog post about htsget-rs and Crypt4GH, this post goes into further details about how htsget works and illustrates more complex use cases.

Let's start by querying an example file. The deployed GA4GH htsget instance has access to [example files][example-files] from the htsget-rs
repository. Recall from the first htsget-rs blog post that the reads endpoint can serve BAM files, for example:

```sh
curl "https://htsget.ga4gh-demo.org/reads/htsnexus_test_NA12878"
```

This will return a set of URL "tickets" inside the "urls" field of the JSON response. These "tickets" contain URLs that
should be fetched and concatenated to produce the response. Additionally, there is a "headers" field that contains HTTP
headers that should included when requesting the url in the ticket. Take a look at the [htsget] spec for more details.

To simplify fetching and concatenating URL tickets, use a htsget client, such as the [GA4GH client][client].

## Querying the header

As a simple example, query the header of the file example file by passing `class=Header`:

```sh
htsget "https://htsget.ga4gh-demo.org/reads/htsnexus_test_NA12878?class=header" > out.bam
```

Internally, this yields a JSON with a URL that can be fetched along with a "Range" header:

```json
{
  "htsget": {
    "format": "BAM",
    "urls": [
      {
        "url": "...",
        "headers": {
          "Range": "bytes=0-4667"
        },
        "class": "header"
      }
    ]
  }
}
```

The client takes care of fetching the URLs and concatenating bytes.

A strength of the htsget protocol is that the output represents a small part of the full file, allowing the user to
query specific regions of a file without needing to obtain the entire file.

In this case, the output represents the BAM header of the file:

```sh
samtools view -H out
```

Note that "..." inside the JSON example responses represents some data or a URL. This will be different when executing
the query.

## Querying reference names with start and end ranges

A more interesting query would involve selecting a specific region, for example chr11. This can be accomplished by
using the `referenceName` parameter. Viewing the output will show data for that specific region:

```sh
htsget "https://htsget.ga4gh-demo.org/reads/htsnexus_test_NA12878?referenceName=11" | samtools view
```

Similarly, the query can be refined further by specifying specific start and end ranges, so that only those regions
are returned:

```sh
htsget "https://htsget.ga4gh-demo.org/reads/htsnexus_test_NA12878?referenceName=11&start=500000&end=5001000" | samtools view
```

Internally, the output from htsget-rs will contain multiple URL tickets that represent the specific data queried:

```json
{
  "htsget": {
    "format": "BAM",
    "urls": [
      {
        "url": "...",
        "headers": {
          "Range": "bytes=0-273085"
        }
      },
      {
        "url": "...",
        "headers": {
          "Range": "bytes=499249-574358"
        }
      },
      {
        "url": "...",
        "headers": {
          "Range": "bytes=627987-647345"
        }
      },
      {
        "url": "...",
        "headers": {
          "Range": "bytes=824361-842100"
        }
      },
      {
        "url": "...",
        "headers": {
          "Range": "bytes=977196-996014"
        }
      },
      {
        "url": "...",
        "headers": {
          "Range": "bytes=2596771-2596798"
        }
      }
    ]
  }
}
```

## Querying Crypt4GH files

Moving on to a more complex example, we will now incorporate querying [Crypt4GH][c4gh] encrypted files from htsget-rs.
To decrypt Crypt4GH files, install the Crypt4GH [CLI][c4gh-cli] and get the [keys] from the htsget-rs repository.

Then, query like before, except add the `encryptionScheme` parameter:

```sh
curl "https://htsget.ga4gh-demo.org/reads/htsnexus_test_NA12878?class=header&encryptionScheme=C4GH"
```

This is an experimental parameter that htsget-rs supports.

This will return a JSON that contains encrypted data when concatenated. Here, there are additional URLs that are base64
encoded. These URLs represent inline data to the JSON ticket, and just need to be decoded to obtain the bytes. They
follow the same semantics as the other URLs and should be concatenated after decoding.

```json
{
  "htsget": {
    "format": "BAM",
    "urls": [
      {
        "url": "data:;base64,..."
      },
      {
        "url": "...",
        "headers": {
          "Range": "bytes=16-123"
        }
      },
      {
        "url": "data:;base64,..."
      },
      {
        "url": "...",
        "headers": {
          "Range": "bytes=124-65687"
        }
      }
    ]
  }
}
```

Putting it all together, and using the keys from the htsget-rs repo, the data can be accessed by running:

```sh
htsget "https://htsget.ga4gh-demo.org/reads/htsnexus_test_NA12878?class=header&encryptionScheme=C4GH" | crypt4gh decrypt --sk bob.sec | samtools view -H
```

## Running the htsget-compliance test suite

As an extra section, the htsget protocol has a compliance suite that can be run on htsget-rs. This contains tests that
ensure that htsget-rs runs as expected. 

In order to run the compliance tests, follow the installation instructions in the [htsget-compliance] repository and
then run the following on the deployed htsget-rs instance:

```sh
htsget-compliance https://htsget.ga4gh-demo.org | jq '.["summary"]'
```

[first]: https://umccr.org/blog/htsget-rs-crypt4gh/
[example-files]: https://github.com/umccr/htsget-rs/tree/main/data
[client]: https://htsget.readthedocs.io/en/latest/quickstart.html#installation
[htsget]: https://samtools.github.io/hts-specs/htsget.html
[c4gh]: https://samtools.github.io/hts-specs/crypt4gh.pdf
[c4gh-cli]: https://github.com/EGA-archive/crypt4gh-rust
[keys]: https://github.com/umccr/htsget-rs/tree/main/data/c4gh/keys
[htsget-compliance]: https://github.com/ga4gh/htsget-compliance?tab=readme-ov-file#installation
