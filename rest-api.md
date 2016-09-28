## Overview

Islandora offers a REST API that allows external programs to query Solr (its text index) and to retrieve files and other content that make up Islandora objects. The REST API:

* is read-only (e.g., only supports HTTP GET)
* only returns JSON (there is no content negotiation), except where binary file content is requested
* uses standard HTTP response codes
* respects Islandora's access control mechanisms, including IP-based restrictions on access to certain collections.

This guide provides an overview of using the Islandora REST API. Many of the details covered here are specific to use of the API on digital.lib.sfu.ca, but [detailed information on Islandora's REST API](https://github.com/discoverygarden/islandora_rest) is also available.

Typically, an application consuming Islandora's REST API will issue a series of requests to accomplish a task. For example, an application may

1.  query the REST API's Solr endpoint to discover some matching objects, then
1.  issue a request for each object in the result set using the object's unique identifier to retrieve a list of properties pertaining to that object, then
1.  issue a final request to retrieve a specific file for processing or rendering to an end user

In many cases the third request is not necessary, since the second request may provide all the data required by the application.

## HTTP Responses

Response bodies (if present) will contain eIther JSON or binary file content. The HTTP response codes used by the Islandora REST API are:

* 200 OK
  * The response body will contain JSON or binary file content.
* 400 Bad Request
  *  The request contained some invalid information. The response body will contain a message saying the request was invalid.
* 401 Unauthorized
  * No response body. The request attempted an action and was denied.
* 403 Forbidden
  * No response body. The request attempted an action that required a higher level of permissions and was denied.
* 404 Not Found
  * No response body. The request attempted an action on a object or datastream (defined below) that does not exist or that is hidden from public view.
* 500 Internal Server Error
  * The response body will contain a detailed description of the error.

## Solr queries

Applications can query Islandora'a Solr index using the REST API. The Solr index contains a wide range of data, so it is useful not only for end-user "discovery" queries, but also for getting lists of pages in book objects and similar queries. Another use for Solr queries is to perform simple analysis on Islandora's content. Results retrieved from the REST API's Solr endpoint typically contain each matching object's unique identifier (called its "PID") so subsequent requests can link to the object or retrieve additional information about the object.

Solr queries take the form `field:string`, where 'field' is the Solr field to search in and 'string' is the string to search for. In addition to the field query, a number of parameters are available to indicate what fields to return in the results, or to indicate how to sort the results (but there are many other parameters you can include). An example of a query using the REST API's Solr endpoint is:

`http://digital.lib.sfu.ca/islandora/rest/v1/solr/dc.title:postcard+AND+dc.subject:vancouver?fl=PID`

This query searches for the keyword "down" in the `dc.title` Solr field combined with the keyword "vancouver" in the `dc.subject` field, and the `fl` parameter instructs Solr to retun the value of the `PID` field for each matching object.

Searching multiple Solr fields is expressed as ORing the same query in each field:

`http://digital.lib.sfu.ca/islandora/rest/v1/solr/dc.title:"british+columbia"+OR+dc.description:"british+columbia"?fl=PID`

Note that all spaces must be replaced by +, and phrases are enclosed in quotation marks. The colon is reserved for separating the field from the query string, so colons within query strings must be escaped with a backslash (`\`).

To sort Solr results, include the `sort` parameter with the value of the field to sort on followed by a `+` and either `asc` or `desc`:

`http://digital.lib.sfu.ca/islandora/rest/v1/solr/dc.title:"british+columbia"+OR+dc.description:"british+columbia"?fl=PID,dc.title&sort=mods_originInfo_encoding_w3cdtf_keyDate_yes_dateIssued_t+asc`

Sorting Solr results can be tricky, since sort fields cannot be multivalued, and it is difficult to tell whether or not a field is multivalued. For queries against SFU Library's Islandora Solr REST endpoint, sorting on `mods_originInfo_encoding_w3cdtf_keyDate_yes_dateIssued_t` is safe.

To return a "page" of results, include the `start` and `rows` parameters:

`http://digital.lib.sfu.ca/islandora/rest/v1/solr/dc.title:"british+columbia"+OR+dc.description:"british+columbia"?fl=PID,dc.title&sort=mods_originInfo_encoding_w3cdtf_keyDate_yes_dateIssued_t+asc&start=200&rows=10'`

Solr parameter are separated from the field query by a question mark (`?`) and are combined with an ampersand (`&`) as illustrated above. The most common parameters include:

* `fl`
  * Comma-separated list of fields to return in the results.
* `sort`
  * Field to sort on. The field cannot be multivalued, and the field name must be accompanied by a sort order, either `+asc` or `+desc`. 
* `start`
  * The offset (by default, 0) into the responses at which Solr should begin displaying content.
* `rows`
  * Number of rows to return at a time. The default is site-specific.
* `debug`
  * Show debugging information in the result.
* `fq`
  * Filter query. Applies a filter query to the search results. A useful way to limit search results to a specific collection on digital.lib.sfu.ca is to apply a filter query on the collection's namespace, e.g., `fq=PID:bcp\:*`.

Most queries against the Solr endpoint will be for discovering objects whose indexed metadata contain specific keywords. An example of a query that performs basic reporting or analysis via the REST API is used in the following command-line PHP script, which reads in a list of PIDs, queries the REST Solr enpoint for each object's content model, and prints out the result:

```php
<?php
/**
 * Script to get the content model for each object in a list of PIDs.
 * Uses Islandora's REST interface.
 */
// No trailing slash.
$islandora_host = 'http://digital.lib.sfu.ca';
// One PID per line.
$input_pids_file = 'input.txt';
$input_pids = file($input_pids_file);

foreach ($input_pids as $pid) {
  $pid = trim($pid);
  $pid_escaped = preg_replace('/:/', '\:', $pid);
  $request = $islandora_host . '/islandora/rest/v1/solr/PID:' . $pid_escaped . '?fl=PID,RELS_EXT_hasModel_uri_s';
  $result = file_get_contents($request);
  $result = json_decode($result, TRUE);
  print "Object $pid has content model " . $result['response']['docs'][0]['RELS_EXT_hasModel_uri_s'] . "\n";
}
?>
```

### Responses to Solr queries

The Islandora REST API returns JSON in response to a Solr query. The JSON contains two parts, a `responseHeader` and a `response`. The `responseHeader` provides information about the query and the `response` contains one `doc` per matching object. Each `doc` contains the fields requested in the `fl` query parameter. If the `fl` parameter is omitted, the entire Solr document for each matching object is returned.

The following example queries the Solr endpoint for the phrase "british columbia" in the `dc.title` field, and indicates that the values of the PID and dc.title fields are returned in the results. Only the first 5 rows (i.e., Solr documents) are to be included in the results. The query is:

`http://digital.lib.sfu.ca/islandora/rest/v1/solr/dc.title:"british+columbia"?fl=PID,dc.title&rows=5`

And the response is:

```json
{
   "responseHeader":{
      "status":0,
      "QTime":0,
      "params":{
         "q":"dc.title:\u0022british columbia\u0022",
         "json.nl":"map",
         "fl":"PID,dc.title",
         "start":"0",
         "fq":"RELS_EXT_isViewableByUser_literal_ms:\u0022anonymous\u0022 OR RELS_EXT_isViewableByRole_literal_ms:\u0022anonymous user\u0022 OR ((*:* -RELS_EXT_isViewableByUser_literal_ms:[* TO *]) AND (*:* -RELS_EXT_isViewableByRole_literal_ms:[* TO *]))",
         "rows":"5",
         "version":"1.2",
         "wt":"json"
      }
   },
   "response":{
      "numFound":1504,
      "start":0,
      "docs":[
         {
            "PID":"pfp:4457",
            "dc.title":[
               "Aged British Columbia Indians"
            ]
         },
         {
            "PID":"pfp:4640",
            "dc.title":[
               "British Columbia Tooth Picks"
            ]
         },
         {
            "PID":"pfp:6094",
            "dc.title":[
               "Indian Chiefs,  British Columbia"
            ]
         },
         {
            "PID":"pfp:6103",
            "dc.title":[
               "Totem Pole,  \/   British Columbia."
            ]
         },
         {
            "PID":"pfp:6109",
            "dc.title":[
               "Indian Maiden,  \/   British Columbia"
            ]
         }
      ]
   }
```

## Getting object properties, and video, audio, XML, and other content

An object's content models are listed in the JSON returned by queries like `http://digital.lib.sfu.ca/islandora/rest/v1/object/alping:756`, in the "models" field in the returned JSON. All objects have "fedora-system:FedoraObject-3.0" in this list; it can be ignored.

The binary content that makes up an Islandora object are known as "datastreams". Each datastream has its own unique identifier, called a datastream ID ("DSID"). You can think of datastreams as files, since they have `size`, `mimetype`, and `created` properties (among others). Most Islandora objects have a thumbnail image (which has a datastream ID of "TN"), an XML file that contains descriptive metadata about the object (every object has a "DC" datastream and most have a "MODS" datastream), and, if required by the object's content model, a datastream that contains its main content file. This datastream usually has a datastream ID of "OBJ" and could be a movie file, an audio file, or an image file. The particular set of datastreams that comprise a given object are determined largely by the object's content model, and are also listed in the JSON returned by the previous request: 

```json
{
   "pid":"alping:756",
   "label":"Mt. [Mount] Baker ice school, July 15, 1951",
   "owner":"admin",
   "models":[
      "islandora:sp_large_image_cmodel",
      "fedora-system:FedoraObject-3.0"
   ],
   "state":"A",
   "created":"2016-06-07T14:09:40.056Z",
   "modified":"2016-06-07T17:44:02.068Z",
   "datastreams":[
      {
         "dsid":"RELS-EXT",
         "label":"Fedora Object to Object Relationship Metadata.",
         "state":"A",
         "size":553,
         "mimeType":"application\/rdf+xml",
         "controlGroup":"X",
         "created":"2016-06-07T14:09:40.056Z",
         "versionable":true,
         "versions":[

         ]
      },
      {
         "dsid":"MODS",
         "label":"MODS Record",
         "state":"A",
         "size":4561,
         "mimeType":"application\/xml",
         "controlGroup":"M",
         "created":"2016-06-07T14:09:40.056Z",
         "versionable":true,
         "versions":[

         ]
      },
      {
         "dsid":"DC",
         "label":"DC Record",
         "state":"A",
         "size":2117,
         "mimeType":"application\/xml",
         "controlGroup":"M",
         "created":"2016-06-07T14:09:40.056Z",
         "versionable":true,
         "versions":[

         ]
      },
      {
         "dsid":"OBJ",
         "label":"OBJ Datastream",
         "state":"A",
         "size":1651496,
         "mimeType":"image\/jp2",
         "controlGroup":"M",
         "created":"2016-06-07T14:09:40.056Z",
         "versionable":true,
         "versions":[

         ]
      },
      {
         "dsid":"TECHMD",
         "label":"TECHMD",
         "state":"A",
         "size":6725,
         "mimeType":"application\/xml",
         "controlGroup":"M",
         "created":"2016-06-07T17:43:47.724Z",
         "versionable":true,
         "versions":[

         ]
      },
      {
         "dsid":"TN",
         "label":"Thumbnail",
         "state":"A",
         "size":5527,
         "mimeType":"image\/jpeg",
         "controlGroup":"M",
         "created":"2016-06-07T17:43:52.991Z",
         "versionable":true,
         "versions":[

         ]
      },
      {
         "dsid":"JPG",
         "label":"Medium sized JPEG",
         "state":"A",
         "size":33129,
         "mimeType":"image\/jpeg",
         "controlGroup":"M",
         "created":"2016-06-07T17:43:59.265Z",
         "versionable":true,
         "versions":[

         ]
      },
      {
         "dsid":"JP2",
         "label":"JPEG 2000",
         "state":"A",
         "size":1651496,
         "mimeType":"image\/jp2",
         "controlGroup":"M",
         "created":"2016-06-07T17:44:02.068Z",
         "versionable":true,
         "versions":[

         ]
      }
   ]
}
```

Each datastream can then be requested using a request in the form `http://digital.lib.sfu.ca/islandora/rest/v1/object/[PID]/datastream/[DSID]`, where `[PID]` is the object's PID and `[DSID]` is the datastream's DSID. Note that the REST API only returns the content of the datastream (or, optionally, the datastream's properties). If necessary, the requesting application must provide an embedded viewer or some other way of presenting it to the user. For example, the JP2 datastream, which is a JPEG2000 file, will not display natively in web browsers.

### Getting a thumbnail

As explained above, retrieving the content of a datastream, in this case the TN datastream, requires a request to the REST API that specifies the object's PID (in this example, "alping:756") and the datastream's DSID ("TN"):

`http://digital.lib.sfu.ca/islandora/rest/v1/object/alping:756/datastream/TN`

Here is a thumbnail retrieved using the URL above:

![Thumbnail for object alping:756](http://digital.lib.sfu.ca/islandora/rest/v1/object/alping:756/datastream/TN)

### Getting XML metadata

If you request the DC or MODS datastream, you will get back an XML file containing metadata describing the object. For example, `http://digital.lib.sfu.ca/islandora/rest/v1/object/alping:756/datastream/DC` will return this content:

```xml
<oai_dc:dc xmlns:oai_dc="http://www.openarchives.org/OAI/2.0/oai_dc/"
xmlns:dc="http://purl.org/dc/elements/1.1/"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.openarchives.org/OAI/2.0/oai_dc/ http://www.openarchives.org/OAI/2.0/oai_dc.xsd">
  <dc:title>Mt. [Mount] Baker ice school, July 15, 1951</dc:title>
  <dc:subject>People--Mountaineering</dc:subject>
  <dc:description></dc:description>
  <dc:contributor>[Chambers, Richard Harwood] (photographer)</dc:contributor>
  <dc:date>1951-07-15</dc:date>
  <dc:type>Image</dc:type>
  <dc:format></dc:format>
  <dc:format>1 photograph : b&amp;w ; 5.5 x 8 cm&gt;</dc:format>
  <dc:identifier>alping:756</dc:identifier>
  <dc:identifier>local: Alpine_790</dc:identifier>
  <dc:identifier>uuid: ea767fbf-d8ce-4e5a-b0d2-136252c42242</dc:identifier>
  <dc:identifier>http://content.lib.sfu.ca/cdm/ref/collection/alping/id/779</dc:identifier>
  <dc:language>English</dc:language>
  <dc:coverage>Baker, Mount, WA</dc:coverage>
  <dc:coverage>+48.77683, -121.81433</dc:coverage>
  <dc:rights>Images from the Richard Harwood (Dick) Chambers Alpine Photographs   Collection have been made available by Simon Fraser University Library   under a Creative Commons attribution, non-commercial license. A full legal   outline of the license can be viewed at:   http://creativecommons.org/licenses/by-nc/3.0/legalcode. We encourage the   appropriate open use of images from this collection for educational and   other not-for-profit purposes. Attribution/citation should be provided as   follows:  Image [insert image number here, eg. MSC130-2144-01] courtesy of   the Richard Harwood (Dick) Chambers Alpine Photographs Collection, a   digital initiative of Simon Fraser University Library. [Please include the   website url when images are used in an offline or print-based context].  Parties interested in using high-resolution versions of the images from   the Richard Harwood (Dick) Chambers Alpine Photographs Collection for   commercial purposes should contact Special Collections and Rare Books, SFU   Library.</dc:rights>
</oai_dc:dc>
```

### Getting information about compound objects: books, newspaper issues, and generic compound objects

Islandora supports many content models, including ones that combine sets of simple objects into more complex structures. The three most common types of complex objects are generic compound, books, and newspaper issues.

#### Generic compound

Objects with a content model of "islandora:compoundCModel" contain 0 or more child objects. To get the child objects, applications must query Solr for all objects that have a field with the name `RELS_EXT_isConstituentOf_uri_s:*[PID]`, where `[PID]` is the PID of the compound object. The compound object's children are ordered, so the results should include the `RELS_EXT_isSequenceNumberOf[PID]_literal_s` field, where `[PID]` is the compound object's PID with its colon replaced by an underscore.

For example, if a compound object's PID is "cfu:2968", we get a list of its children with the following Solr query:

`http://digital.lib.sfu.ca/islandora/rest/v1/solr/RELS_EXT_isConstituentOf_uri_s:*cfu\:2968?fl=PID,RELS_EXT_isSequenceNumberOfcfu_2968_literal_s&rows=1000`

The response will be:

```json
{
   "responseHeader":{
      "status":0,
      "QTime":1,
      "params":{
         "q":"RELS_EXT_isConstituentOf_uri_s:*cfu\\:2968",
         "json.nl":"map",
         "fl":"PID,RELS_EXT_isSequenceNumberOfcfu_2968_literal_s",
         "start":"0",
         "fq":"RELS_EXT_isViewableByUser_literal_ms:\u0022anonymous\u0022 OR RELS_EXT_isViewableByRole_literal_ms:\u0022anonymous user\u0022 OR ((*:* -RELS_EXT_isViewableByUser_literal_ms:[* TO *]) AND (*:* -RELS_EXT_isViewableByRole_literal_ms:[* TO *]))",
         "rows":"1000",
         "version":"1.2",
         "wt":"json"
      }
   },
   "response":{
      "numFound":12,
      "start":0,
      "docs":[
         {
            "PID":"cfu:2959",
            "RELS_EXT_isSequenceNumberOfcfu_2968_literal_s":"4"
         },
         {
            "PID":"cfu:2957",
            "RELS_EXT_isSequenceNumberOfcfu_2968_literal_s":"2"
         },
         {
            "PID":"cfu:2958",
            "RELS_EXT_isSequenceNumberOfcfu_2968_literal_s":"3"
         },
         {
            "PID":"cfu:2956",
            "RELS_EXT_isSequenceNumberOfcfu_2968_literal_s":"1"
         },
         {
            "PID":"cfu:2960",
            "RELS_EXT_isSequenceNumberOfcfu_2968_literal_s":"5"
         },
         {
            "PID":"cfu:2962",
            "RELS_EXT_isSequenceNumberOfcfu_2968_literal_s":"7"
         },
         {
            "PID":"cfu:2964",
            "RELS_EXT_isSequenceNumberOfcfu_2968_literal_s":"9"
         },
         {
            "PID":"cfu:2961",
            "RELS_EXT_isSequenceNumberOfcfu_2968_literal_s":"6"
         },
         {
            "PID":"cfu:2967",
            "RELS_EXT_isSequenceNumberOfcfu_2968_literal_s":"12"
         },
         {
            "PID":"cfu:2965",
            "RELS_EXT_isSequenceNumberOfcfu_2968_literal_s":"10"
         },
         {
            "PID":"cfu:2963",
            "RELS_EXT_isSequenceNumberOfcfu_2968_literal_s":"8"
         },
         {
            "PID":"cfu:2966",
            "RELS_EXT_isSequenceNumberOfcfu_2968_literal_s":"11"
         }
      ]
   }
}
```

Solr cannot return the list of children presorted in natural order  -- requesting applications must sort for themselves. It is also a good idea to add a `rows` parameter to the query with a value that is likely to be higher than the number of child objects, to ensure that the query returns records for all the children.

#### Books

Objects with a content model of "islandora:bookCModel" work similarly: we need to query Solr to get the list page objects. The specific fields in the query are different than those used in the compound object query above, and PIDs are not part of the field names. For example, if a book object's PID is "aldine:11302", we get a list of its pages with the following Solr query:

`http://digital.lib.sfu.ca/islandora/rest/v1/solr/RELS_EXT_isMemberOf_uri_s:*aldine\:11302?fl=PID,RELS_EXT_isSequenceNumber_literal_s&rows=1000`

#### Newspaper issues

Newspaper issues, which have a content model of "islandora:newspaperIssueCModel", are similar to books. Information about their pages is contained in the same Solr fields.

## Conclusion

As explained at the beginning of this guide, an application may need to make several requests to Islandora's REST API to get the data it requires. The most complicated queries are against the API's Solr endpoint, since Solr enforces its own rules about valid queries and because there are a number of optional parameters that can influence the results returned. Requests for information about a specific object, using the object's PID, are much simpler, since they return a simple structured list of properties. Requests for datastreams return binary file content such as image, video, and XML data. The datastreams that are available for a given object are determined largely by the object's content model.

