---
layout: post
title: "Jenkins: Managing Metadata"
description: "Civilizing Data Artifacts in Jenkins"
category: 
tags: [Jenkins, workflow, data-sciences, analysis, bioinformatics, metadata]
author: Ioannis K. Moutsatsos
---
{% include JB/setup %}

In [Jenkins: Documenting Data with Metada]() I discussed the importance of metadata, established some nomenclature, and provided a rationale for classifying Jenkins analyses builds into data source and metadata-only builds. Finally, I discussed some of the challenges that we face trying to relate data source and metadata artifacts when these are processed through non-linear, ad-hoc Jenkins pipelines. In this second blog entry I will discuss a metadata management design that works effectively within the constraints of the Jenkins application model.

<!--more-->
All analyses pipelines start with an input dataset. If this dataset  originates as a Jenkins archived artifact, we consider the build a data source build. Therefore, **data source builds** provide input dataset(s), and/or additional metadata (information/insights about the data) for downstream analyses. Any metadata generated from a data source or metadata-only build is used to document the data structure, content, origin, quality and other characteristics of the data. Metadata assist the selection of the correct algorithm, analysis parameters and limits, and ultimately in modeling the underlying variables.
 
## Primary Data
Primary data is considered the initial input to an analysis pipeline whose goal is to perform additional meta-analysis, discovery and hypotheses generation. Primary data sets are typically introduced into the Jenkins artifact repository in either of two ways.

1. As a direct upload from the user
2. As an artifact of a previous Jenkins build
3. As a data stream from another system (database, API etc)

## Primary Metadata
As we've discussed in Part One of this series, a **data source build** will also generate some metadata for the data source artifact. This primary metadata can be generated in a variety of ways including:

1. User annotation
2. Automated annotation
3. Built-in metadata
4. Computed metadata
5. External metadata manifest accompanying the data or that can be generated

 
Each of these sources of metadata should be considered when a dataset is first introduced into the Jenkins artifact repository.
 
### User Annotation
User annotation is the easiest to understand and the **hardest to collect**. Users attach annotation while preparing to upload data to Jenkins. In this case, the build form contains fields and user controls that facilitate the entry of  controlled annotation useful for giving meaning and context to the data. 
 
For life science data analysis, user annotation frequently includes **experimental metadata**  that cannot be easily obtained otherwise. These metadata become critical for downstream cross-experimental analyses when results from different experiments need to be compared. 

>Examples of experimental metadata include the organism or cell-line where the sample originated, treatment doses, control, or reference samples etc.
 
Unfortunately, experience has shown that no matter what self-populating and intelligent form controls we present to the user, much of the user annotation enters the system with default values such as:

```
ANALYSIS_NAME='My favorite Analysis'
DESCRIPTION= 'Enter a concise description'
```
 
This challenge, of course, does not mean that we should stop trying to capture this critical metadata. It just means that we need to streamline the metadata capture and make it as easy and painless for the user as possible.
 
User **annotation entered in a build form is captured as build parameters**  and is stored in the corresponding build.xml file together with other types of metadata. 
>Metadata stored in build parameters can be queried through the Jenkins API. This is the only mechanism provided by Jenkins to store and query metadata. 

We will present a **hybrid metadata storage strategy**  that presents additional advantages by storing and managing metadata in java properties files.

 
### Automated annotation
Well designed data processing systems automatically collect a variety of processing metadata.

Jenkins collects an abundance of 'low level' build metadata and stores it in the corresponding build.xml file. Additional metadata may be added to this file from Jenkins plugins. For example, the Build Environment Plugin stores a complete set of the environment variables in the build.xml file and allows you to later display and compare the environment of builds.

 
### Built-in Metadata
Instruments and analysis tools generate data sets in various formats. Some produce data in tabular delimited format, some in XML or JSON and others some in custom (hopefully structured) text or binary format. In each of these cases important acquisition and processing metadata may be embedded in a data column, in an XML element or in another parsable part of the document.
 
Understanding and extracting built-in metadata requires a great deal of familiarity with the scientific domain in which this data is generated. Nonetheless, equipped with the appropriate knowledge and parsing tools one can extract lots of useful information directly from the data.

### Computed Metadata
 
Another set of embedded metadata useful in downstream analysis comes from analyzing the data structure and its properties. For example, we frequently analyze datasets to understand:

* Number of rows
* Number of columns
* Number of observations
* Column typing (numeric, character, date, categorical etc.)
* Summary statistics for numeric columns (mean, median, sd, max, min etc.)

### External Dataset Manifest

If an external manifest file containing metadata is available it can be used (similar to the built-in metadata) to assist in the annotation of the dataset. Such files are rather rare unless the data is already curated and stored in a database. 

## Jenkins and Generated Metadata

Unfortunately there is no default Jenkins mechanism for storing, managing and querying build-generated metadata. Ideally, this information would be structured and stored in an appropriate data store, but such a mechanism is not currently available in Jenkins.

However, we can still use Jenkins file-based archiving  for a simpler yet quite effective metadata management strategy. Below I present this strategy with the understanding that although effective, it still has some drawbacks and can be further improved by the inclusion of a dedicated data store for metadata management.   
 
## Java Properties File for Storing Metadata
We have developed a strategy for storing, accessing and reusing metadata using Java property files. Java key-value property files can be read, written and managed in standardized ways via Java and Groovy APIs. This file based strategy is based on the following conventions.
 
### Each Dataset has an Associated Metadata File
Each build that generates a new dataset also generates and associated set of metadata.  dataset metadata is typically a combination of:
1. User annotation
2. Jenkins build annotation
3. Built-In Metadata
 
### Metadata are Stored as Java properties files
The data generating build is responsible for writing metadata to a standard Java properties file (key=value format) . This properties file is handled as a standard build artifact. In other words, it is archived with the build and can be easily referenced with a build artifact URL.
 
### Metadata File Names are Standardized
The Java properties file should have a standardized name that makes its identification unique amongst the build artifacts. We have chosen a naming convention that allows us to match the name of the properties file to its corresponding data set.
 
For example, if the generated data set is called JData_Normalized.csv, the corresponding metadata file is called JData_Normalized.properties.

This naming convention allows us to have multiple pairs of data and corresponding metadata in the same archive. However, any other similar convention is acceptable. For example, you may choose to name the file metadata.properties and insure that no other such file exists in the build artifact collection.
 
By having this standardized naming convention you will be able to easily identify and retrieve metadata for a dataset for use in build form UI controls (using the Active Choices plugin for example) as well as during the analysis build.

As a result of this approach, analysis scripts first load the dataset to be analyzed and then load the corresponding metadata that contains information such as sample controls, excluded samples, relevant experimental metadata etc. Having this information allows the analysis of the dataset to use the appropriate logic and algorithmic workflow for a particular dataset.

 
## Active Metadata Management
After its original creation metadata is reused and actively maintained. Properties may be added, updated or deleted. Below are the conventions for this active metadata management.
 
### Downstream Builds Update the Original Metadata
As we have mentioned some analysis do not create or transform a dataset, but simply generate new information about it. These builds simply generate new metadata that need to be added or merged into the original metadata properties.
 
Metadata generating builds typically perform the following steps:

1. Read the original metadata properties
2. Update properties as needed (create, update, delete)
3. Write the properties file back into the original build archive
 
This approach successively accumulates relevant metadata as a datasets is processed through various quality control and analysis builds. 
>Downstream analyses can utilize the metadata created from upstream analyses. This allows users to perform ad hoc analyses and the analysis pipeline to branch as needed.
 
### A Special Property: LAST Job Build
It is possible that a user may want to explore the same type of analysis under different conditions or with different parameters. In this case, the analysis may add a special property identified by 'propertyName.JOB_NAME.LAST' to indicate that this property was updated by the last successful run of a particular job type.
 
### A Special Property: History
It is desirable to allow users to perform ad hoc analysis, but this creates a challenge in Jenkins as there are no clear downstream upstream build paths. How do we track the analysis path of a dataset?
 
We can address some of these challenges by requiring that each build records some logging information in the metadata file itself. Then there is a single file that we can review to understand all of the analyses performed on a dataset.
 
We create a history property and record the build tags of each of the downstream analysis as in the example:
`history=JOB_NAME1-BUILD_NUMBER,JOB_NAME2-BUILD_NUMBER,JOB_NAME4-BUILD_NUMBER,JOB_NAME3-BUILD_NUMBER`
 
## Derived Data Source Metadata
A data source build that generates a derived dataset has an additional source of metadata, the metadata of the input dataset (a.k.a as primary dataset).
 
Although not all of the original metadata will be applicable for the new dataset, it is expected that some of the metadata, and especially experimental metadata and user annotations will be trasferable to the derived dataset. In this case, the derived datasource build is responsible for transferring some of the input datasource metadata and merging it with the new metadata.
 
