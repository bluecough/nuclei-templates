# Templating Guide

**Nuclei** is based on the concepts of `YAML` based template files that define how the requests will be sent and processed. This allows easy extensibility capabilities to nuclei.

The templates are written in `YAML` which specifies a simple human readable format to quickly define the execution process. 

Let's start with the basics and define our own workflow file for detecting the presence of a `.git/config` file on a webserver and take it from there.

Table of Contents
=================

   * [Templating Guide](#templating-guide)
      * [Template Details](#template-details)
         * [<strong>Info</strong>](#info)
         * [<strong>HTTP Requests</strong>](#http-requests)
            * [<strong>Method</strong>](#method)
            * [<strong>Path</strong>](#path)
            * [<strong>Headers</strong>](#headers)
            * [<strong>Body</strong>](#body)
            * [<strong>Matchers</strong>](#matchers)
               * [<strong>Types</strong>](#types)
               * [<strong>Conditions</strong>](#conditions)
               * [<strong>Matched Parts</strong>](#matched-parts)
               * [<strong>Multiple Matchers</strong>](#multiple-matchers)
            * [Extractors](#extractors)
            * [<strong>Example HTTP Template</strong>](#example-http-template)
         * [<strong>DNS Requests</strong>](#dns-requests)
            * [<strong>Type</strong>](#type)
            * [<strong>Name</strong>](#name)
            * [<strong>Class</strong>](#class)
            * [<strong>Recursion</strong>](#recursion)
            * [<strong>Retries</strong>](#retries)
            * [<strong>Matchers</strong>](#matchers)
            * [<strong>Example DNS Template</strong>](#example-dns-template)

## Template Details

Each template has a unique ID which is used during output writing to specify the template name for an output line.

The template file ends with **.yaml** extension. The template files can be created any text editor of your choice.

```yaml
# id contains the unique identifier for the template.
id: git-config
```

ID must not contain spaces. This is done to allow easier output parsing.

### **Info**


Next important piece of information about a template is the <u>**info**</u> block. Info block provides more context on the purpose of the template and the **author**. It can also contain a **severity** field which indicates the severity of the template.

Let's add an **info** block to our template as well.

```yaml
info:
  # name is the name of the template
  name: Git Config File Detection Template
  # author is the name of the author for the template
  author: Ice3man
  # severity is the severity for the template.
  severity: medium
```

Actual requests and corresponding matchers are placed below the info block and they perform the task of making requests to target servers and finding if the template request was succesful.

Each template file can contain multiple requests to be made. The template is iterated and one by one the desired HTTP/DNS requests are made to the target sites.

### **HTTP Requests**

Requests start with a request block which specifies the start of the requests for the template. 

```yaml
# start the requests for the template right here
requests:
```

Requests can be fine tuned to perform the exact tasks as desired. Nuclei requests are fully configurable meaning you can configure and define each and every single thing about the requests that will be sent to the target servers.

#### **Method**

First thing in the request is <u>**method**</u>. Request method can be **GET**, **POST**, **PUT**, **DELETE**, etc depending on the needs. 

```yaml
# method is the method for the request
method: GET
```

#### **Path**

The next part of the requests is the **path** of the request path. Dynamic variables can be placed in the path to modify its behaviour on runtime. Variables start with `{{` and end with `}}` and are case-sensitive. 

1. **BaseURL** - Placing BaseURL as a variable in the path will lead to it being replaced on runtime in the request by the original URL as specified in the target file.
2. **Hostname** - Hostname variable is replaced by the hostname of the target on runtime.

Some sample dynamic variable replacement examples - 

```yaml
path: {{BaseURL}}/.git/config
# this path will be replaced on execution with BaseURL
# When BaseURL = https://abc.com
# path will get replaced to the following - 
https://abc.com/.git/config
```

Multiple paths can also be specified in one request which will be requested for the target.

#### **Headers**

Headers can also be specified to be sent along with the requests. Headers are placed in form of Key-Value pairs. An example header configuration is - 

```yaml
# headers contains the headers for the request
headers:
  # Custom user-agent header
  User-Agent: Some-Random-User-Agent
  # Custom request origin
  Origin: https://google.com
```

#### **Body** 

Body specifies a body to be sent along with the request. For instance - 

```yaml
# body is a string sent along with the request
body: "{\"some random json\"}"
```

#### **Matchers** 

Matchers are the core of nuclei. They are what make the tool so powerful. Multiple type of combinations and checks can be added to ensure that the results you get are free from false positives.

##### **Types**

Multiple matchers can be specified in a request. There are basically 5 types of matchers - 

| Matcher Type | Part Matched               |
| ------------ | -------------------------- |
| status       | Status Code of Response    |
| size         | Content Length of Response |
| word         | Response body or headers   |
| regex        | Response body or headers   |
| binary       | Response body              |

To match status codes for responses, you can use the following syntax.

```yaml
matcher:
  # match the status codes
  - type: status
    # some status codes we want to match
    status:
      - 200
      - 302
```

To match binary for hexadecimal responses, you can use the following syntax.

```yaml
matchers:
      - type: binary
        binary:
        - "504B0304" # zip
        - "526172211A070100" # rar RAR archive version 5.0
        - "FD377A585A0000" # xz tar.xz
        condition: or
        part: body
```

To match size, similar structure can be followed. If the status code of response from the site matches any single one specified in the matcher, the request is marked as successful.

**Word** and **Regex** matchers can be further configured depending on the needs of the users. 

##### **Conditions**

Multiple words and regexes can be specified in a single matcher and can be configured with different conditions like **AND** and **OR**. 

1. **AND** - Using AND conditions allows matching of all the words from the list of words for the matcher. Only then will the request be marked as successful when all the words have been matched.
2. **OR** - Using OR conditions allows matching of a single word from the list of matcher. The request will be marked as successful when even one of the word is matched for the matcher.

##### **Matched Parts** 

Multiple parts of the response can also be matched for the request. 

| Part   | Matched Part                         |
| ------ | ------------------------------------ |
| body   | Body of the response                 |
| header | Header of the response               |
| all    | Both body and header of the response |

Example matchers for response body using AND condition - 

```yaml
matcher:
  # match the body word
  - type: word
   # some words we want to match
   words: 
     - "[core]"
     - "[config]"
   # both words must be found in the response body
   condition: and
   #  we want to match request body (default)
   part: body
```

Similarly, matchers can be written to match anything that you want to find in the response body allowing unlimited creativity and extensibility.

##### **Multiple Matchers** 

Multiple matchers can be used in a single template to fingerprint multiple conditions with a single request. 

Here is an example of syntax for multiple matchers.

```yaml
    matchers:
      - type: word
        name: php
        words:
          - "X-Powered-By: PHP"
          - "PHPSESSID"
        part: header
      - type: word
        name: node
        words:
          - "Server: NodeJS"
          - "X-Powered-By: nodejs"
        condition: or
        part: header
      - type: word
        name: python
        words:
          - "Python/2."
          - "Python/3."
        condition: or
        part: header
```

#### Extractors

Extractors are another important feature of nuclei. Extractors can be used to extract and display in results a match from the response body or headers based on a regular expression.

Currently only `regex` type extractors are supported. A sample extractor for extracting API keys from the response body is as follows - 

```yaml
# A list of extractors for text extraction
extractors:
  # type of the extractor, only regex for now.
  - type: regex
    # part of the response to extract (can be headers, all too)
    part: body
    # regex to use for extraction.
    regex:
      - "(A3T[A-Z0-9]|AKIA|AGPA|AROA|AIPA|ANPA|ANVA|ASIA)[A-Z0-9]{16}"
```

#### **Example HTTP Template**

The final template file for the `.git/config` file mentioned above is as follows - 

```yaml
id: git-config

info:
  name: Git Config File
  author: Ice3man
  severity: medium

requests:
  - method: GET
    path:
      - "{{BaseURL}}/.git/config"
    matchers:
      - type: word
        words:
          - "[core]"
```

### **DNS Requests**

Requests start with a dns block which specifies the start of the requests for the template. 

```yaml
# start the requests for the template right here
dns:
```

DNS requests can be fine tuned to perform the exact tasks as desired. Nuclei requests are fully configurable meaning you can configure and define each and every single thing about the requests that will be sent to the target servers.

#### **Type**

First thing in the request is <u>**type**</u>. Request type can be **A**, **NS**, **CNAME**, **SOA**, **PTR**, **MX**, **TXT**, **AAAA**. 

```yaml
# type is the type for the dns request
type: A
```

#### **Name**

The next part of the requests is the **name** of the request path. Dynamic variables can be placed in the path to modify its value on runtime. Variables start with `{{` and end with `}}` and are case-sensitive. 

1. **FQDN** - variable is replaced by the hostname/fqdn of the target on runtime.

Some sample dynamic variable replacement examples - 

```yaml
name: {{FQDN}}.com
# this value will be replaced on execution with FQDN
# When FQDN = https://this.is.an.example
# name will get replaced to the following - 
this.is.an.example.com
```

As of now the tool supports only one question per request.

#### **Class**

Class type can be **INET**, **CSNET**, **CHAOS**, **HESIOD**, **NONE**, **ANY**. Usually it's enough to just leave it as **INET**

```yaml
# method is the class for the dns request
class: inet
```

#### **Recursion** 

Recursion is a boolean value, and determines if the resolver should only return cached results, or traverse the whole dns root tree to retrieve fresh results. Generally it's better to leave it as **true** 

```yaml
# recursion is a boolean determining if the request is recursive
recursion: true
```

#### **Retries** 

Retries is the number of attempts a dns query is retried before giving up among different resolvers. It's recommended a reasonable value, like **3**. 

```yaml
# retries is a number of retries before giving up on dns resolution
retries: 3
```

#### **Matchers** 

Matchers are just equal to HTTP, but the search is performed on the whole dns response, therefore it's not necessary to specify the **part**. Multiple type of combinations and checks can be added to ensure that the results you get are free from false positives.

##### **Types**

Multiple matchers can be specified in a request. There are basically 5 types of matchers - 

| Matcher Type | Part Matched               |
| ------------ | -------------------------- |
| size         | Response size              |
| word         | Response                   |
| regex        | Response                   |
| binary       | Response                   |

## **Example DNS Template**

The final example template file for performing `A` query, and check if CNAME and A records are in the response is as follows - 

```yaml
id: dummy-cname-a

info:
  name: Dummy A dns request
  author: mzack9999
  severity: none

dns:
    - name: "{{FQDN}}"
      type: A
      class: inet
      recursion: true
      retries: 3
      matchers:
      - type: word
        words:
          # The response must contains a CNAME record
          - "IN\tCNAME"
          # and also at least 1 A record
          - "IN\tA"
        condition: and
```
