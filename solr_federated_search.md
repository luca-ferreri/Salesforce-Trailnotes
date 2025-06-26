# Salesforce Federated Search (with OpenSearch protocol) to query Apache Solr
## The Business Use Case
Our challenge was to log thousands of relevant email messages per day in Salesforce — without exhausting its limited and valuable storage. As a first option, we turned to Salesforce Federated Search, which is [included (i.e., already covered by the license) in the Enterprise, Professional, Unlimited, and Developer Editions](https://help.salesforce.com/s/articleView?id=platform.external_data_source_define.htm&type=5).

Federated Search allows us to create an external object backed by a database (in our case Apache Solr) exposed through the [**OpenSearch** protocol](https://github.com/dewitt/opensearch). However, it comes with some [limitations](https://help.salesforce.com/s/articleView?id=ai.search_federated_considerations.htm&type=5) stricter than those applied to Salesforce Connect adapters. One key constraint is that 
> Federated Search supports only external lookup relationships, where the external object must always be the parent in the relationship.

## Introduction

The goal of this guide is to enable querying external data hosted in **Apache Solr** using the **OpenSearch** protocol, specifically to integrate with **Salesforce Federated Search**. While this integration is entirely possible, we were surprised—especially as members of the open-source community—by how scarce comprehensive end-to-end resources are on the topic. From deploying an Apache Solr server, to configuring the schema, exposing a compliant OpenSearch description, and even writing the required XSL transformation file, the journey involves several steps that are often only partially documented or scattered across different sources.

This document aims to fill that gap by offering a practical, example-driven walkthrough of the entire process. We hope it serves as a helpful reference for others taking on the same challenge.

As always, we welcome feedback and suggestions—this is a living document, and we’re happy to keep improving it!

## Deploy your Apache Solr
The following is not intended as a comprehensive guide to deploying Apache Solr. There are several options available for deploying Solr — from using binaries (for Unix-compatible systems or Windows) to containerized solutions such as Docker and Kubernetes. Additionally, Solr can operate in different modes, such as a single-node server or a distributed multi-node cluster managed with ZooKeeper. For more detailed information, we recommend consulting the excellent official [Apache Solr documentation](https://solr.apache.org/resources.html).

For the illustrative purposes of this guide, we will use the simplest deployment method: running Apache Solr in Docker as a single-node server.

The docker-compose.yml file is as follows:
```yaml
services:
    # Define a service named "solr"
  solr:
    # Use the official Solr Docker image from Docker Hub
    image: solr
    # Map port 8983 on the host to port 8983 in the container
    # This allows you to access the Solr web interface at http://localhost:8983
    ports:
      - "8983:8983"
    # Mount the local ./data/ directory to /var/solr in the container
    # This ensures Solr data is persisted on the host between container restarts
    volumes:
      - ./data/:/var/solr
    # The following two commands are optionals.
    # "solr-precreate gettingstarted" tells Solr to create a new core named "gettingstarted"
    command:
      - solr-precreate
      - gettingstarted
    # Set environment variables for the container
    # Here, it enables the "scripting" module which allows use of scripting features like the XSLTResponseWriter
    environment:
      - SOLR_MODULES=scripting
```
Once you have Apache Solr up and running, take some time to familiarize yourself with the Solr Dashboard - available at [http://localhost:8983](http://localhost:8983) if you are running it locally, or at [http://<some_ip>:8983](http://<some_ip>:8983) if you are deploying it remotly. From this point forward, we will refer to this base URL as `<SOLR_URL>`. 

You should also explore the Solr API and the command-line interface. For a helpful introduction, consider followign the official [Solr tutorial](https://solr.apache.org/guide/solr/latest/getting-started/solr-tutorial.html).
## Your data in Apache Solr
### Create a Core
Let’s say you want to create a core (i.e. a collection of document) called `amazingCore`. Inside the container, e.g. `docker exec -it solr_tests-solr-1 /bin/bash`, run the following command:
```bash
solr create -c amazingCore
```
This will create the necessary folder structure for the new core. You can also observe the corresponding folders locally, thanks to the mounted volume that syncs the container’s data with your local filesystem.
### Configure the Core
Now, let’s say that each document represents an email message, and we want to store both its content and metadata—such as the message ID, sender, receivers, and subject. Specifically, we want to define:
+ a custom type field called `text_normalized` to perform a case-insensitive full-text search
+ the following fields
    + `email_id`: type `string`, single-valued, retrivable, searchable, and **required** in each document
    + `sender`: type `string`, single-valued, retrivable, searchable, and **required**
    + `receivers`: type `string`, multi-valued, retrivable, searchable, and **required**
    + `subject`: custom type `text_normalized`, single-valued, retrivable, searchable and **required**
    + `body`: custom type `text_normalized`, single-valued, retrivable, searchable, and **required**
    + `date`: custom type `date`, single-valued, retrivable, searchable, and **required**
    + `tags`: type `string`, multi-valued, retrivable, searchable, and **required**
+ a "catchall filed"  that collects the contents of all other fields and indexes them into a special field named _text_. This enables general full-text search across all document fields using a single query input.

To define the following schema, you can use the `curl` command as shown below:
```bash
curl <SOLR_URL>/solr/amaizingCore/schema -H 'Content-type:application/json' -d '<json here!>'
```
with the following JSON:

```json
{
    "add-field-type": [
        {
            "name": "text_normalized",
            "class": "solr.TextField",
            "positionIncrementGap": "100",
            "analyzer": {
                "tokenizer": {"class": "solr.StandardTokenizerFactory"},
                "filters": [{"class": "solr.LowerCaseFilterFactory"}],
            },
        },
        {
            "name": "date",
            "class": "solr.DatePointField"
        },
    ],
    "add-field": [
        {
            "name": "email_id",
            "type": "string",
            "multiValued": false,
            "stored": true,
            "indexed": true,
            "required": true,
        },
        {
            "name": "sender",
            "type": "string",
            "multiValued": false,
            "stored": true,
            "indexed": true,
        },
        {
            "name": "receivers",
            "type": "string",
            "multiValued": true,
            "stored": true,
            "indexed": true,
        },
        {
            "name": "subject",
            "type": "text_normalized",
            "multiValued": false,
            "stored": true,
            "indexed": true,
        },
        {
            "name": "body",
            "type": "text_normalized",
            "multiValued": false,
            "stored": true,
            "indexed": true,
        },
        {
            "name": "date",
            "type": "date",
            "multiValued": false,
            "stored": true,
            "indexed": true,
        },
        {
            "name": "tags",
            "type": "string",
            "multiValued": true,
            "stored": true,
            "indexed": true,
        }
    ],
    "add-copy-field": [{"source": "*", "dest": "_text_"}],
}
```
### Load data
We are now ready to load documents into the core. Once again, we can do this using the following `curl` command:
```bash
curl <SOLR_URL>/solr/amaizingCore/update?commit=true -H 'Content-type:application/json' -d '<json here!>'
```
and the json is like the following
```json
[
    {
        "email_id": "2c84ea3d-e8e5-40cd-a3b3-1b822134e432",
        "sender": "james70@example.net",
        "receivers": [
            "josecox@example.net",
            "bbarber@example.net",
            "allisonbrown@example.net",
            "james04@example.com",
            "christopher44@example.org"
        ],
        "subject": "Congress dog determine relate admit win trade.",
        "body": "Speak record drop scientist full current. Ground find policy action heavy gas. Perform whole memory traditional word listen culture. Lay commercial road drive school popular week. Wear country practice.",
        "date": "2024-09-14T23:16:18Z",
        "tags": [
            "reflect",
            "tree",
            "wear",
            "career",
            "discover"
        ]
    },
    {
        "email_id": "875ce94f-794c-4cb6-a047-a9dd1e5f40ca",
        "sender": "annehill@example.org",
        "receivers": [
            "stephanielee@example.com",
            "brittanyharris@example.net",
            "larrygilbert@example.org",
            "spatterson@example.net"
        ],
        "subject": "Easy although wonder son between group discuss.",
        "body": "Ability family toward notice glass year. Cut particular ever skin campaign reality. Fact black structure whether result conference safe pretty. I job site claim. Prepare walk pass court. Bring evidence ready firm name.",
        "date": "2024-10-22T06:12:41Z",
        "tags": [
            "mother",
            "night",
            "matter",
            "discussion",
            "hope",
            "concern",
            "today",
            "cut",
            "employee",
            "home",
            "present",
            "near",
            "if",
            "during",
            "professor",
            "story",
            "care",
            "back",
            "record"
        ]
    },
    ...
]
```

### Query the docs
Let’s say you want to retrieve all documents that contain the word “scientist”. You can do this by sending a request to the following URL: `<SOLR_URL>/solr/amazingCore/query?q=scientist` You can either `curl` this URL or open it directly in your browser. By default, Solr returns the response in JSON format. If you prefer the output in XML, simply add the `wt=xml` (writer type) parameter like so: `<SOLR_URL>/solr/amazingCore/query?q=scientist&wt=xml`. 

Now, Salesforce **Federated Search** to work properly expects as input an XML Atom with the OpenSearch specs/constraints (reference [one](https://help.salesforce.com/s/articleView?id=ai.search_configure_solr_federated_search.htm&type=5) and [two](https://developer.salesforce.com/docs/atlas.en-us.federated_search.meta/federated_search/federated_search_intro.htm)). Therefore, we have to enable the `xslt` query response writer and to configure it. The steps are:
+ modify the `solrconfig.xml` file in `data/amazingCore/conf/solrconfig.xml` adding the following lines
    ```xml
    <queryResponseWriter name="xslt"
        class="solr.scripting.xslt.XSLTResponseWriter">
    </queryResponseWriter>
    ```
    and restart the service.
+ create the folder `xslt` in the path `data/amazingCore/` and create the following file `atom.xsl`:
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!-- XSLT stylesheet to transform a custom XML search response into an Atom feed with OpenSearch and Salesforce-specific metadata -->
    <xsl:stylesheet version="1.0"
        xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
        xmlns:atom="http://www.w3.org/2005/Atom"
        xmlns:opensearch="http://a9.com/-/spec/opensearch/1.1/"
        xmlns="http://www.w3.org/2005/Atom"
        xmlns:sfdc="http://salesforce.com/2016/federatedsearch/1.0"
        exclude-result-prefixes="xsl atom opensearch">

        <!-- Output settings: XML with indentation, UTF-8 encoding, and correct media type -->
        <xsl:output method="xml" indent="yes" encoding="UTF-8" media-type="application/xml" />
        <!-- Strip whitespace-only text nodes from the input to avoid unnecessary spaces -->
        <xsl:strip-space elements="*" />

        <xsl:template match="/response">
            <feed xmlns="http://www.w3.org/2005/Atom"
                xmlns:opensearch="http://a9.com/-/spec/opensearch/1.1/">
                
                <title>emailmessage</title>
                <!-- OpenSearch metadata for pagination and query tracking -->
                <opensearch:totalResults>
                    <xsl:value-of select="//result/@numFound" />
                </opensearch:totalResults>
                <opensearch:startIndex>
                            <xsl:value-of select="//result/@start" />
                </opensearch:startIndex>
                <opensearch:itemsPerPage>
                    <xsl:value-of
                        select="//lst[@name='responseHeader']/lst[@name='params']/int[@name='rows']" />
                </opensearch:itemsPerPage>
                <!-- OpenSearch query details -->
                <opensearch:Query role="request">
                    <!-- Search term -->
                    <xsl:attribute name="searchTerms">
                        <xsl:value-of
                            select="//lst[@name='responseHeader']/lst[@name='params']/str[@name='q']" />
                    </xsl:attribute>
                    <!-- Index of first item in this page -->
                    <xsl:attribute name="startIndex">
                                <xsl:value-of select="//result/@start" />
                    </xsl:attribute>
                    <!-- Number of results returned -->
                    <xsl:attribute name="count">
                        <xsl:value-of
                            select="//result/@numFound" />
                    </xsl:attribute>
                </opensearch:Query>

                <!-- Atom <entry> elements for each result document -->
                <xsl:for-each select="result/doc">
                    <entry>
                        <!-- Email subject as the title -->
                        <title>
                            <xsl:value-of select="str[@name='subject']" />
                        </title>
                        <!-- Custom metadata -->
                        <recordType name="emailmessage" />
                        <!-- Unique ID for the entry -->
                        <id>
                            <xsl:value-of select="str[@name='id']" />
                        </id>
                        <!-- Static fake link (could be made dynamic if needed) -->
                        <link href="https://www.example.com" />
                        <!-- Last updated date -->
                        <updated>
                            <xsl:value-of select="date[@name='date']" />
                        </updated>
                        <!-- Summary (subject again) -->
                        <summary>
                            <xsl:value-of select="str[@name='subject']" />
                        </summary>
                        <!-- Salesforce recordType -->
                        <sfdc:recordType>emailmessage</sfdc:recordType>
                        <!-- Email sender -->
                        <sfdc:sender>
                            <xsl:value-of select="str[@name='sender']" />
                        </sfdc:sender>
                        <!-- Email recipients (list of) -->
                        <sfdc:receivers>
                            <xsl:for-each select="arr[@name='receivers']/str">
                                <xsl:value-of select="." />
                                <xsl:if test="position() != last()">
                                </xsl:if>
                            </xsl:for-each>
                        </sfdc:receivers>
                        <!-- Email subject -->
                        <sfdc:subject>
                            <xsl:value-of select="str[@name='subject']" />
                        </sfdc:subject>
                        <!-- Email body -->
                        <sfdc:body>
                            <xsl:value-of select="str[@name='body']" />
                        </sfdc:body>
                        <!-- Email date -->
                        <sfdc:emaildate>
                            <xsl:value-of select="date[@name='date']" />
                        </sfdc:emaildate>
                        <!-- Email tags (list of) -->
                        <sfdc:tags>
                            <xsl:for-each select="arr[@name='tags']/str">
                                <xsl:value-of select="." />
                                <xsl:if test="position() != last()">
                                </xsl:if>
                            </xsl:for-each>
                        </sfdc:tags>
                    </entry>
                </xsl:for-each>
            </feed>
        </xsl:template>
    </xsl:stylesheet>
    ```
now by querying `<SOLR_URL>/solr/amazingCore/query?q=scientist&wt=xsl&tr=atom.xsl` the output is the one expected by Salesforce. In the specific case:
```xml
<feed xmlns:sfdc="http://salesforce.com/2016/federatedsearch/1.0"
    xmlns="http://www.w3.org/2005/Atom">
    <title>emailmessage</title>
    <opensearch:totalResults xmlns:opensearch="http://a9.com/-/spec/opensearch/1.1/">1</opensearch:totalResults>
    <opensearch:startIndex xmlns:opensearch="http://a9.com/-/spec/opensearch/1.1/">0</opensearch:startIndex>
    <opensearch:itemsPerPage xmlns:opensearch="http://a9.com/-/spec/opensearch/1.1/" />
    <opensearch:Query xmlns:opensearch="http://a9.com/-/spec/opensearch/1.1/" role="request"
        searchTerms="scientist" startIndex="0" count="1" />
    <entry>
        <title>Congress dog determine relate admit win trade.</title>
        <recordType name="emailmessage" />
        <id>b3249981-0b44-4ca2-9e94-1fb083b99d30</id>
        <link href="https://www.example.com" />
        <updated>2024-09-14T23:16:18Z</updated>
        <summary>Congress dog determine relate admit win trade.</summary>
        <sfdc:recordType>emailmessage</sfdc:recordType>
        <sfdc:sender>james70@example.net</sfdc:sender>
        <sfdc:receivers>
            josecox@example.netbbarber@example.netallisonbrown@example.netjames04@example.comchristopher44@example.org</sfdc:receivers>
        <sfdc:subject>Congress dog determine relate admit win trade.</sfdc:subject>
        <sfdc:body>Speak record drop scientist full current. Ground find policy action heavy gas.
            Perform whole memory traditional word listen culture. Lay commercial road drive school
            popular week. Wear country practice.</sfdc:body>
        <sfdc:emaildate>2024-09-14T23:16:18Z</sfdc:emaildate>
        <sfdc:tags>reflecttreewearcareerdiscover</sfdc:tags>
    </entry>
</feed>
```
## OpenSearch Description
According to Salesforce documentation, [reference one](https://developer.salesforce.com/docs/atlas.en-us.federated_search.meta/federated_search/federated_search_examples.htm) and [reference two](https://help.salesforce.com/s/articleView?id=ai.search_configure_publish_solr_federated_search.htm&type=5), Salesforce requires access to a public URL that hosts the OpenSearch XML description. This file defines how to query the external search engine and outlines its supported features. Therefore, we need to publish an `opensearch.xml` file with the following content:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<OpenSearchDescription xmlns="http://a9.com/-/spec/opensearch/1.1/"
    xmlns:sfdc="http://salesforce.com/2016/federatedsearch/1.0">
    <ShortName>Solr Search</ShortName>
    <Contact>mycontact@domain.com</Contact>
    <Url type="application/atom+xml" rel='results' sfdc:maxTotalResults='1000'
        sfdc:recordType='emailmessage' indexOffset='0'
        template="<SOLR_URL>/solr/amazingCore/select?q={searchTerms}&amp;type={sfdc:recordType}&amp;start={startIndex?}&amp;num={count?}&amp;wt=xslt&amp;tr=atom.xsl" />
    <sfdc:RecordTypes>
        <sfdc:RecordType name='emailmessage'>
            <sfdc:Field name="sender" type="string" sortable="true" />
            <sfdc:Field name="subject" type="string" sortable="true" />
            <sfdc:Field name="body" type="longstring" sortable="false" />
            <sfdc:Field name="emaildate" type="date" sortable="true" />
            <sfdc:Field name="tags" type="string" sortable="true" />
            <sfdc:Field name="receivers" type="longstring" sortable="true" />
        </sfdc:RecordType>
    </sfdc:RecordTypes>
    <InputEncoding>UTF-8</InputEncoding>
    <OutputEncoding>UTF-8</OutputEncoding>
    <sfdc:Version>1</sfdc:Version>
</OpenSearchDescription>
```
**Warning:**
    There’s an important detail we learned the hard way — setting `indexOffset='0'` is crucial. If omitted, OpenSearch defaults to one-based indexing, which causes it to skip the first document.
## SalesForce
We are now ready to set up the Salesforce External Data Source using Federated Search (OpenSearch). The steps are explained in the [official documentation](https://help.salesforce.com/s/articleView?id=ai.search_define_federated_search.htm&type=5), but keep the following key points in mind:
+ For the OpenSearch Description URL, make sure to provide the URL of the OpenSearch Description XML file—not the Solr endpoint itself.

+ Consider securing the connection using appropriate certificates and credentials.
