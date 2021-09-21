---
title: Amazon for the Integrative Genomics Viewer (IGV)
authors: 
  - roman-valls-guimera
date: "2019-02-16"
slug: igv-amazon
layout: post
permalink: https://umccr.org/blog/2019/02/16/igv-amazon
categories:
  - bioinformatics
  - infrastructure
tags:
  - bioinformatics
  - igv
summary: "On expanding the Broad's Integrative Genomics Viewer (IGV) to read input directly from AWS S3."
---

TL;DR: https://github.com/igvteam/igv/pull/620

# Intro

In bioinformatics there's hardly anybody that hasn't come across the Broad Institute's venerable Java [Integrative Genomics Viewer][igv], if not, have [a peek at the original nature methods publication][igv_scihub]: it wouldn't be an overstatement to say that IGV is the samtools of genome visualization.

<!--
No resizing support for Markdown... sigh, I guess that a static image resizer/thumbnailer is needed here :/

![igv google search](/img/2019/02/igv_everywhere.png)
-->

<img src="/img/2019/02/igv_everywhere.png" alt="igv google search" width="640" height="300">


Even [Google's deepvariant][deepvariant] used [IGV to validate their algorithms and learn about variants][igv-deepvariant-test]:

> "The read and reference data are encoded as an image for each candidate variant site (...)  These labeled image + genotype pairs, along with an initial CNN, which can be a random
model, a CNN trained for other image classification tests (...) the reference and read bases, quality scores, and other read features are encoded into a red–green–blue (RGB) pileup image at a candidate variant"

Originally conceived as a locally-run desktop Java application IGV is, still today, the bread and butter of bioinformaticians and clinicians alike. With the arrival of whole genome sequencing (WGS), datasets became bigger so time consuming storage logistics burdens started to creep up.

{{<tweet 428616138252353536>}}

Analyzing several WGS datasets quickly turned into [a bioinformatics sweat shop pattern][bioinfo_sweatshops]. To solve that, IGV learned [new tricks to access remote data][igv_server], but how good those turned out to be in the research trenches?

# The typical IGV setup

The [implementation of such ~~tricks~~ hacks][igv_server_setup_umccr] require a backend with a traditional client/server model. The server is talking to IGV via [HTTP (range) requests][range_requests] and requesting only a handful of genomic reads that the user needs to see at a particular genomic region. Since 24/7 longrunning processes often clash with the traditional batch-oriented HPC culture, research software engineers have resorted to **suboptimal workarounds to expose the valued sequencing data to their colleagues**:

1. SSH+X11 forwarding: Low resolution and overall sluggish UI.
2. [SSHFS-related user level mounts][ssh-fuse]: Prone to filesystem stalling when your laptop briefly goes to sleep.
3. [S3 FUSE mount][s3-fuse] on AWS via [S3Proxy][s3_proxy] or similar: Majorly unused EC2 instance, low network throughput if that instance is not on the higher end (wasteful, expensive proposition).

Worth mentioning, the genomic viewer alternatives, [NGB][NGB], [jbrowse][jbrowse], [IGB][igb] and [IGV.js][igv-js] unfortunately have some pain points:

1. They currently implement a traditional client-server model which leads to the aforementioned waste of (backend) resources.
2. Some are under-mantained (funding got cut?).
3. Their feature set is "not there yet", or there's "that" missing feature that conservative clinicians trust and love.
4. Their software architecture has questionable choices that endanger its future sustainability.
5. All of the above or worse.

So what's left when, as Titus Brown puts it:

> ["Scientific culture is generational; change will come, but slowly"][titus_generational]

**There must be a better way for users to overcome those data access barriers while lowering CO2 emissions and keeping everything safe, right?**

Our answer is to adapt what's most used (IGV desktop) to a more modern and efficient paradigm, hopefully transitioning to a better situation in the future, incrementally.

**So we connected IGV desktop directly to our our main cloud provider. At UMCCR that is Amazon's AWS.**

Next section for the tech gory details if you are still here. Jump towards the end for new possibilities to awe for :)


# Getting genomic reads directly from S3 into IGV

Right after forking IGV from upstream into an `aws_support` branch, we tackled the backend, navigating the methods with [IDEA IntelliJ community edition][intellij_idea], unfortunately VSCode [is quite far behind with Java editor ergonomy][vscode-java-behind].

### Backend

All our cancer sequencing data is deposited in [S3 buckets with encryption at rest][s3_encryption_at_rest]. Furthermore, in order to access the data, **researchers have to authenticate and get authorized via [AWS Cognito][aws_cognito], which supports federated authentication via Google in our particular case but others are also supported out of the box**. Our IGV auth flow looks like this (see images in tweet):

{{<tweet 1090446488071630849>}}

Under the hood, and after the [OAuth/OIDC flow][oauth_oidc_flows], a temporary AWS STS token is issued to give IGV access to a particular AWS resource:

![aws cognito federated idp](/img/2019/02/scenario-cup-cib.png)


A policy to limit access to a particular bucket is defined in the AWS cognito "authenticated users" default IAM role:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "IGVCognitoAuthedUsers",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::your_bucket",
                "arn:aws:s3:::your_bucket/*"
            ]
        },
        {
            "Sid": "IGVListBuckets",
            "Effect": "Allow",
            "Action": [
                "s3:ListAllMyBuckets"
            ],
            "Resource": [
                "arn:aws:s3:::*"
            ]
        }
    ]
}
```

The default **unauthed cognito role is left empty** so those users do not get access to anything and since [IAM roles and policies have a default "deny all" policy][iam-policy-deny-all].

Using the [AWS SDK for Java][aws_sdk_java] we abstracted away **[the common misconception that AWS S3 is "like a filesystem"][s3_misconception]**. It is not, it's an object store, therefore this bit requires some attention so that the UI (and users) can deal with it:

> The Amazon S3 data model is a flat structure: you create a bucket, and the bucket stores objects. There is no hierarchy of subbuckets or subfolders; however, you can infer logical hierarchy using key name prefixes and delimiters as the Amazon S3 console does. The Amazon S3 console supports a concept of folders.

You can browse most of this implementation in the [`AmazonUtils.java` class on the ongoing IGV pullrequest][igv_amazonutils_class].

### UI

Since our growing data warehouse has many objects, [we were concerned that retrieving a s3 bucket tree listing recursively at once could become too slow from an UX/UI perspective][stackoverflow_dynamic_jtree_question]. Therefore, **we implemented JTree's `TreeWillExpand` listener so that S3 bucket listings could be obtained dynamically** as the user explored the potentially many levels of the hierarchy, wasting no API calls nor time.

The current [hardcoded/mock/prototype GA4GH IGV data model][igv_ga4gh_data_model] and corresponding [UI representation][igv_ga4gh_ui] gave us an idea of how to shoehorn our dynamic approach.


# About feeding giants and possible bright(er) futures

Now, we hear you say: why are we feeding Mr Bezos giant machine when we have local HPC produce? 

Well, in some situations, uptime is not counted [in 9's][hpc_hahaha] but perhaps in **8's or 0's if you are lucky**... true story:

<img src="/img/2019/02/hpc_uptime.png" alt="hpc uptimes and unscheduled maintenance windows" width="680" height="300">

We would like to think that bioinformatics is [transitioning towards consolidation, managed infrastructure that we can rely on][what_hplac_should_be] and paying the commercial cloud premium is a good way towards that reality. We would like to go forward where fighting ancient, barely funded tools and infrastructure are not the norm anymore, but instead modern, **integrated and robust workflows that gather metrics which allow increasing performance** enable or staff to aim higher in their science. As an end result, **tightening the loop between research question and results, iterating faster and overall having a better time as a researcher**.

This particular serverless solution will probably be forgotten as newer developments arise, but at least it allowed us to **very easily profile data access patterns and begin to decide which data to store on cheaper archival solutions** [(such as AWS Glacier)][aws_glacier], as opposed to rule of thumb, arbitrarily decided archival lifecycles. After this development **we are beginning to switch from "where to connect", "which data to host where" type of questions which plague core sequencing facilities to more interesting ones**:

1. Which genomic curators in our team make **queries in our custom cancer gene list against which samples**? Can we dynamically (re)generate that list based on IGV curator queries?
1. How many and which regions/variants/SVs/SNPs are examined and **how much time is spent** on them per curator? **Human fatigue can be a bias in cancer interpretation work**, can we build better tools to avoid this situation?
1. Shall we **integrate [variant sharing tools like VIPER][viper]** to have robust quorum on cancer interpretation work?

We know that the Broad institute had significant pushback in the past when implementing its [controversial metric-gathering GATK phone home feature][gatk_et_feature], and let us be clear that **none of those tracking/metric changes are present nor will be included [in the current pullrequest against IGV proper][igv_pr]** but having this system in place has already helped us make informed decisions on our core activity in many levels in the few days it has been in production.

Next possible stop to insure better interoperability: [htsget][htsget]. Any takers?


[NGB]: https://github.com/epam/NGB/
[jbrowse]: https://jbrowse.org/
[aws_sdk_java]: https://aws.amazon.com/sdk-for-java/
[aws_cognito]: https://aws.amazon.com/cognito/
[oauth_oidc_flows]: https://twitter.com/braincode/status/1088579033237880833
[igb]: https://bitbucket.org/lorainelab/integrated-genome-browser/src/master/
[igv]: https://software.broadinstitute.org/software/igv/
[igv-js]: https://github.com/igvteam/igv.js
[igv_pr]: https://github.com/igvteam/igv/pull/620
[igv_scihub]: https://sci-hub.tw/https://www.nature.com/articles/nbt.1754
[igv_server]: https://software.broadinstitute.org/software/igv/DataServer
[deepvariant]: https://sci-hub.tw/https://www.nature.com/articles/nbt.4235
[igv-deepvariant-test]: https://github.com/google/deepvariant/blob/c2167e7c90f016905f309f118eb3897935ee7c5f/third_party/nucleus/io/sam_reader_test.cc#L226
[s3_proxy]: https://github.com/nkrumm/s3proxy
[s3_encryption_at_rest]: https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingServerSideEncryption.html
[aws_accelerated_transfer]: https://s3-accelerate-speedtest.s3-accelerate.amazonaws.com/en/accelerate-speed-comparsion.html
[igv_amazonutils_class]: https://github.com/igvteam/igv/pull/620/files#diff-4eb026bcdcbd11039fe8e138f6614363
[titus_generational]: http://ivory.idyll.org/blog/2014-open-and-tenured.html
[cognito_federated_idp]: https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-integrating-user-pools-with-identity-pools.html
[igv_collaborative_curation]: https://software.broadinstitute.org/software/igv/
[gatk_et_feature]: https://software.broadinstitute.org/gatk/documentation/article.php?id=1250
[what_hplac_should_be]: https://www.dursi.ca/post/what-are-national-research-computing-platforms-for-now.html
[range_requests]: https://tools.ietf.org/html/rfc7233
[viper]: https://github.com/MarWoes/viper
[bioinfo_sweatshops]: https://image.slidesharecdn.com/2014-bosc-keynote-140711070550-phpapp01/95/2014-bosckeynote-13-638.jpg
[ssh-fuse]: https://github.com/libfuse/sshfs
[s3-fuse]: https://github.com/s3fs-fuse/s3fs-fuse
[s3_misconception]: https://www.reddit.com/r/aws/comments/2pkv2l/s3_misconceptions/
[intellij_idea]: https://www.jetbrains.com/idea/
[stackoverflow_dynamic_jtree_question]: https://stackoverflow.com/questions/54339957/simple-dynamic-jtree-based-s3-bucket-object-chooser-example/54433941/
[igv_ga4gh_data_model]: https://github.com/igvteam/igv/blob/d14f503314488b1059ea8bcea37d218560ea975d/src/main/java/org/broad/igv/ga4gh/Ga4ghAPIHelper.java#L58
[igv_ga4gh_ui]: https://github.com/igvteam/igv/blob/d14f503314488b1059ea8bcea37d218560ea975d/src/main/java/org/broad/igv/ga4gh/Ga4ghLoadDialog.java
[hpc_hahaha]: https://en.wikipedia.org/wiki/High_availability#%22Nines%22
[aws_glacier]: https://aws.amazon.com/glacier/
[vscode-java-behind]: https://code.visualstudio.com/docs/languages/java
[iam-policy-deny-all]: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html
[htsget]: https://samtools.github.io/hts-specs/htsget.html
[igv_server_setup_umccr]: https://gist.github.com/brainstorm/70e5c95620351b82bf44f94ba9efef28
