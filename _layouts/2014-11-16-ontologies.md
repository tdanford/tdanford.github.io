---
layout: post
title:  "Using ADAM to Replace the BAM"
date:   2013-12-27 01:14:01
tags: adam bioinformatics
---

[ADAM](https://github.com/bigdatagenomics/adam) is the name of a new project which intends to re-create some of the core data structures and tools from bioinformatics in a system that can take advantage of new technologies like [Avro](http://avro.apache.org/), [Parquet](http://parquet.io/), and [Spark](http://spark.incubator.apache.org/).

In this post, I will try to explain some of the basics of how ADAM can be used (and how it might be extended).

Compiling and Running ADAM
==========================

Before we do anything else, we will need to download and compile ADAM itself. ADAM currently lacks a binary distribution, and you will need ``git`` and ``mvn`` in order to build it for the first time.

```bash
git clone https://github.com/bigdatagenomics/adam.git
cd adam
mvn clean install
```

When the build is finished, you should have a working ADAM jar file in ``adam-cli/target/adam-0.6.0-SNAPSHOT.jar``.  You can run the ADAM command-line tool by passing this jar to the jvm as a command-line argument, 

```bash
java -Xmx2g -jar adam-cli/target/adam-0.6.0-SNAPSHOT.jar
```

When you run the ADAM command, you should see the following output: 

<pre>
     e            888~-_              e                 e    e
    d8b           888   \            d8b               d8b  d8b
   /Y88b          888    |          /Y88b             d888bdY88b
  /  Y88b         888    |         /  Y88b           / Y88Y Y888b
 /____Y88b        888   /         /____Y88b         /   YY   Y888b
/      Y88b       888_-~         /      Y88b       /          Y888b

Choose one of the following commands:

            bam2adam : Converts a local BAM file to ADAM/Parquet and writes locally or to HDFS, S3, etc
           transform : Apply one of more transforms to an ADAM file and save the results to another ADAM file
            flagstat : Prints statistics for ADAM data similar to samtools flagstat
           reads2ref : Convert an ADAM read-oriented file to an ADAM reference-oriented file
             mpileup : Output the samtool mpileup text from ADAM reference- oriented data
               print : Print an ADAM formatted file
   aggregate_pileups : Aggregates pileups in an ADAM reference-oriented file
            listdict : Prints the contents of an ADAM sequence dictionary
             compare : Compares two ADAM files based on read name
     computeVariants : Compute variant data from genotypes.
            adam2vcf : Convert an ADAM variant to the VCF ADAM format
            vcf2adam : Convert a VCF file to the corresponding ADAM format
</pre>

Each of these entries, such as ``bam2adam`` and ``flagstat``, are ADAM utilities and tools that can be invoked.  The first that we'll use is the ``bam2adam`` command to convert a BAM file into an "ADAM record file" which provides a more efficient basis for many of the subsequent computations.

To test out the execution of ADAM on a local machine, we'll also need a small BAM.  I typically choose one of the pre-aligned BAMs from the [1000genomes project on S3](http://aws.amazon.com/1000genomes/), but any reasonable BAM should work just fine -- for example, I use the ``s3cmd`` command-line tool to grab a simple pre-processed BAM with an exome sample in it from S3: 

```bash 
s3cmd get \
s3://1000genomes/data/HG00096/exome_alignment/HG00096.mapped.ILLUMINA.bwa.GBR.exome.20111114.bam
```

Once you have the BAM file, the next step is to convert it into and "ADAM File": 

```bash
java -Xmx2g -jar adam-cli/target/adam-0.6.0-SNAPSHOT.jar bam2adam \ 
HG00096.mapped.ILLUMINA.bwa.GBR.exome.20111114.bam \
HG00096
```

The first argument is the location of the BAM file I downloaded; the second is the location (that will be created) of the ADAM record file.  As this command runs it should generate a lot of output, but when it has completed you should see a directory with the same name (``HG00096``) as the second command-line argument . This directory should have four files in it, such as ``part0`` through ``part3``, which are together the compressed binary ADAM representation of the contents of the BAM file.  

```bash
$ ls HG00096
part0  part1  part2  part3
```

It will be this directory, and its contents, that we will load and manipulate using the ADAM commands.

Before going any further, we should take a moment to discuss the structure and organization of the ADAM project itself.

Structure of the ADAM Project
=============================

The ADAM project is composed of three logically distinct components: 

1. formats
2. methods
3. a command-line wrapper

The _formats_ component, present in the ``adam-format`` sub-module of the ADAM repository, contains definitions of the central bioinformatics data structures which will be serialized to and from disk, distributed in memory across a cluster, and used as the basis for most of the computations and tools in the ADAM project.  

The _methods_ component, embodied in the ``adam-core`` sub-module, contains implementations of common tools for manipulating these data structures. For example, there are implementations of "local realignment," BQSR, and MarkDuplicates (from the GATK and Picard packages) in this module; there are also re-implementations of other non-standard tools for examining and comparing data between ADAM files. Each of these methods is currently written as Scala code which  

Finally, the _command-line wrapper_ is present in the ``adam-cli`` sub-module.  This contains the wrapper code necessary for running each of the tools as a command-line utility, executed either against an existing Spark cluster or locally if no Spark cluster is provided.  When we ran the ``bam2adam`` command, above, since we did not provide an explicit location for a Spark cluster, a "local cluster" (parallelized across the cores of the local machine only) was created and the command was executed in that framework instead.

Avro and Parquet for Formats
----------------------------

The formats themselves are written as Avro schemas in the [adam.avdl](https://github.com/bigdatagenomics/adam/blob/master/adam-format/src/main/resources/avro/adam.avdl) file.  This file defines as set of "record" types, which are similar to the table definitions in the DDL-sublanguage of a SQL relational database system.  Avro is a tool for reading these record definitions and generating Java class files which mirror their structure, as well as parsers and serializers to convert between objects of these classes and "over the wire" binary representations.  

Parquet defines a serialization for sets of these objects into multi-part file format which is suitable for use within a "column-store"-style data system &mdash; that is, it stores the columns (the individual attributes of each record) as a different part of the file, it uses dictionary- and run-length-encoding along with compression to reduce the size of these columns on disk, it maintains headers to allow chunking of the files themselves, and it provides support for projections and push-down predicates when reading the files.  

The ADAMRecord format
---------------------

At this point, if you have not already, you should read the [SAM/BAM file format specification](http://samtools.sourceforge.net/SAMv1.pdf).

<pre>
record ADAMRecord {
    union { null, string } referenceName = null;
    union { null, int }    referenceId = null;
    union { null, long }   referenceLength = null;
    union { null, string } referenceUrl = null;

    union { null, string } readName = null;
    union { null, string } sequence = null;
    union { null, int }    mapq = null;
    union { null, long }   start = null;
    union { null, string } cigar = null;
    union { null, string } qual = null;

    union { null, int }    mateReferenceId = null;
    union { null, string } mateReference = null;
    union { null, long }   mateReferenceLength = null;
    union { null, string } mateReferenceUrl = null;
    union { null, long }   mateAlignmentStart = null;

    union { boolean, null } readPaired = false;
    union { boolean, null } properPair = false;
    union { boolean, null } readMapped = false;
    union { boolean, null } mateMapped = false;
    union { boolean, null } readNegativeStrand = false;
    union { boolean, null } mateNegativeStrand = false;
    union { boolean, null } firstOfPair = false;
    union { boolean, null } secondOfPair = false;
    union { boolean, null } primaryAlignment = false;
    union { boolean, null } failedVendorQualityChecks = false;
    union { boolean, null } duplicateRead = false;

    union { null, string } mismatchingPositions = null;
    union { null, string } attributes = null;

    union { null, string } recordGroupId = null;
    union { null, string } recordGroupSequencingCenter = null;
    union { null, string } recordGroupDescription = null;
    union { null, long }   recordGroupRunDateEpoch = null;
    union { null, string } recordGroupFlowOrder = null;
    union { null, string } recordGroupKeySequence = null;
    union { null, string } recordGroupLibrary = null;
    union { null, int }    recordGroupPredictedMedianInsertSize = null;
    union { null, string } recordGroupPlatform = null;
    union { null, string } recordGroupPlatformUnit = null;
    union { null, string } recordGroupSample = null;
}
</pre>

Where'd The Header Go? 
----------------------

You might be asking yourself at this point, "these are the records from a BAM, but where is the information normally stored in the BAM or SAM _header_?" 


<span style="font-size: 150%; font-family: monospace;">RDD[ADAMRecord]</span> as a BAM Replacement
========================================

The core abstraction in Spark is the [RDD](http://spark.incubator.apache.org/docs/0.8.1/api/core/org/apache/spark/rdd/RDD.html).  RDD is an acronym for ["Resilient Distributed Dataset"](http://www.cs.berkeley.edu/~matei/papers/2011/tr_spark.pdf), and can essentially be thought of as an array of arbitrary objects, parallelized across the Spark cluster itself.  


Writing a new command-line tool
===============================

Use the [ListDict](https://github.com/bigdatagenomics/adam/blob/master/adam-cli/src/main/scala/edu/berkeley/cs/amplab/adam/cli/ListDict.scala) tool as an example CLI tool template.

```scala
class CommandClass(protected val args: CommandArgs) extends AdamSparkCommand[CommandArgs] {

  val companion: AdamCommandCompanion = CommandObject

  def run(sc: SparkContext, job: Job): Unit = {
	// ... spark code goes here ...
  }

}
```

Finally, in order to have your new utility accessible from the command-line itself (and to take advantage of the Spark "harness" that ADAM provides), you'll need to add the name of your ``CommandClass`` to the ``commands`` list of classes in the [``AdamMain``](https://github.com/bigdatagenomics/adam/blob/master/adam-cli/src/main/scala/edu/berkeley/cs/amplab/adam/cli/AdamMain.scala) object.

