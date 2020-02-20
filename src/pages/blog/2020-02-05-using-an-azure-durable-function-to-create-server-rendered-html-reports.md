---
templateKey: blog-post
title: Using an Azure Durable Function to Create Server Rendered HTML Reports
date: 2020-02-05T03:05:48.700Z
description: >-
  I have been changing my workflow to take advantage of serverless architecture
  in my projects. Working primarily in Microsoft technologies for most of my
  career I naturally drifted to Azure. Lately I have been working with Azure
  Durable Functions and find them a fantastic technology for solving scenarios
  that require bursts of computer power with long or multiple operations.
  Scenarios such as the problem that we have been tasked to solve.
featuredpost: true
featuredimage: /img/Me2018.jpg
tags:
  - Azure. Durable Function. Serverless.
---
# Introduction

I have been changing my workflow to take advantage of serverless architecture in my projects. Working primarily in Microsoft technologies for most of my career I naturally drifted to Azure. Lately I have been working with Azure Durable Functions and find them a fantastic technology for solving scenarios that require bursts of computer power with long or multiple operations. Scenarios such as the problem that we have been tasked to solve.

# Scenario

What is this problem that we need to solve? Converting data from an old Lotus Notes database (yes, *really*) into a format compatible with our new cloud vendor's database format. The current databases are being exported into a series of Excel spreadsheets. Our job is to transform the data for importing, including formatted HTML reports. For each row in the Excel spreadsheet the project requires choosing one of one hundred different report templates written in HTML. Each report consists of unique `body`, markup, as well as shared `style` and `footer` sections. Each Excel Worksheet consists of~1200 rows.

# Technology

* Javascript
* Azure Durable Functions
* PowerShell

# Why Azure Durable Functions?

For each row in the Excel spreadsheet the project requires choosing one of one hundred different report templates written in HTML. Each report consists of unique `body`, markup, as well as shared `style` and `footer` sections. Each Excel Worksheet consists of~1200 rows. 

So why Azure Durable Functions?

* **Simplicity**: We only need to call two endpoints, not 100.
* **Speed**: Queries will be run asynchronously. For just one Excel Worksheet this the difference between the operation running for 15 minutes or 4.5 minutes !
* **Power**: That speed requires a lot of processing power. Anyone have the budget for that temporary hardware? Yeah, us either.

# How is an Azure Durable Function structured?

Despite the name an Azure Durable Function is not just ***one*** function. A Durable Function is made up of three or more functions: *Trigger*, *Orchestrator* and one or more *Activities*.

## Trigger Function

This is the endpoint that is called by a client. Receives any request payload and forwards the request and payload to the *Orchestrator*, then immediately returns a JSON-formatted message to the caller - including a `statusCode` and `statusQueryGetUri`. 

```javascript
const df = require("durable-functions");

module.exports = async function (context, req) {
    const client = df.getClient(context);
    const record = JSON.stringify({"query": req.query, "body": req.body})
    const instanceId = await client.startNew("TOrchestrator", undefined, record);

    return client.createCheckStatusResponse(context.bindingData.req, instanceId);
};
```

Our Trigger Function receives the template number as a `url` parameter and the JSON-formatted Excel Worksheet Row information. Creates a custom Javascript object and sends that Javascript (JSON) object to the *Orchestrator*. Returns a `statusCode` of *202 "Received"* and the `statusQueryGetUri` back to the calling client script. 

## Orchestrator Function

Calls one or more *Activity* Function(s), either synchronously or asynchronously. Tracks, collects and stores the returned outputs. Note that there should be no I/O operations here, meaning there should be no static strings, functions nor objects returned from the *Orchestrator* itself. Only the results from the *Activity* functions should be returned. 

```javascript
const df = require("durable-functions");

module.exports = df.orchestrator(function* (context) {

    const record = JSON.parse(context.df.getInput());
    var tasks = [];

    tasks.push(yield context.df.callActivity("THeader", record.query));//HEADER
    tasks.push(yield context.df.callActivity(`${(record.query.template).toString()}`, record.body));//BODY
    tasks.push(yield context.df.callActivity("TFooter", record.body));//FOOTER

    let page = `<!DOCTYPE html><html>${tasks[0]}<body>${tasks[1]}${tasks[2]}</body></html>`;

    return page;
});
```

Our *Orchestrator* function receives the JSON object from the *Trigger* Functions. We are using a ***function chaining pattern*** because we need the constructed returned HTML-template in a specific order: the `Header`, a **`Body`** template (based on the JSON's object template property), and **`Footer`** *Activity* functions. 

## Activity Functions

The actual ***meat*** of the *Durable* Function where the actual work happens. Analyze, manipulate, shape and return data to the *Orchestrator*. 

### Activity Function - Header

Returns a standard CSS-styled header string for formatting. This is so we do not have to copy/paste this section in each *`Body`* Function and to have one place to update the style if needed. 

```javascript
module.exports = async function (context) {
    const record = context.bindings.record;

    var head = `<head>    
            <title>record.template : </title>
            <style>
                .bold {font-weight: bold;}
                .underline {border-bottom: 1px solid #000000;}
                .right {text-align: right;}
                .top {vertical-align: top;}
                .center {text-align: center;}
                .grid {
                    border: 5px solid #000000;
                    width: 100%;
                }
                .grid th {
                    border: 1px solid #000000;
                    text-align: left;
                    vertical-align: top;
                }
                .grid input {text-align: center;}
                .grid td {border: 1px solid #000000;}
                .shadedrow {background-color: #b4b4b4;}
                .indent1 {margin-left: 5%;}
                .indent2 {margin-left: 10%;}
            </style>
        </head>`

    return head;
};
```

### Activity Function - Body

Each body template, all 100 of them, are their own function. Then creates an HTML-formatted string that describes the format of the template body combined with the passed JSON-formatted object. Returns the combined HTML-formatted string. 

**NOTES**: 

* We are using Moment.js to properly format the dates.
* This is a just a few lines of one of these hundred templates.

```javascript
  var moment = require('moment');

function Format_Date($date){
    return new Date(moment($date)).toLocaleDateString('en-US')
};

module.exports = async function (context) {
    const record = context.bindings.record;

    var body = ` 
  ...
     </li>
        <li><p><b>Medications Ordered on Day of Admission:</b> SEE MEDICATION SUBFORM</p></li>
        <li><p><b>Allergies:</b> ${record.Allergies}</p></li>
        <li><p><b>Relevant Social Histor:</b> ${record.FamSocHist}</p></li>
        <li>
            <b>Mental Status Exam</b>
            <ol style='list-style-type: lower-alpha;'>
                <li><p>Appearance / Behaviour: ${record.MentalStatusExam_a}</p></li>
                <li><p>Speech: ${record.MentalStatusExam_b}</p></li>
                <li><p>Mood and Affect: ${record.MentalStatusExam_c}</p></li>
                <li>
                    <p>Thought Process and Content</p>
                    <ol style="list-style-type: lower-roman;">
                        <table>
                            <tr>
                                <td><li>Thought disordered:</li></td>
                                <td><input type='radio'`;
                                if(record.MentalStatusExam_d_1 == 'Y'){body += ` checked`};
                                body += `>Y</td>
                                <td><input type='radio'`;
                                if(record.MentalStatusExam_d_1 == 'N'){body += ` checked`};
                                body += `>N</td>
                            </tr>
                            <tr>
                                <td><li>Suicidal ideation:</li></td>
                                <td><input type='radio'`;
                                if(record.MentalStatusExam_d_2 == 'Y'){body += ` checked`};
                                body += `>Y</td>
                                <td><input type='radio'`;
                                if(record.MentalStatusExam_d_2 == 'N'){body += ` checked`};
                                body += `>N</td>
                            </tr>
      ...

    </ol>
    `;

    return body;
```

### Activity Function - Footer

Returns an HTM-formatted string that has a standard structure describing, based on a passed JSON-formatted object, the date and time that the report was generated in Lotus Notes. **NOTE**: We are using Moment.js to properly format the dates.

```javascript
var moment = require('moment');

function Format_Date($date){
    return new Date(moment($date)).toLocaleDateString('en-US')
};

function Format_Time($time){
    return new Date(moment($time)).toLocaleTimeString();
}

module.exports = async function (context) {
    const record = context.bindings.record;
   
    var footer = `    <footer>
            <table>
                <tr><td class='bold'>Finalized by</td<td>record.Therapist_FN , record.MD_FN
                <tr><td class='bold'>on</td><td>`
                    if(record.FinalizeDate != null){ footer += Format_Date(record.FinalizeDate)};
                footer += `</td></tr>
                <tr><td class='bold'>at</td><td>`
                    if(record.FinalizeTime != null){ footer += Format_Time(record.FinalizeTime) };
                footer += `</td></tr>
            </table>
        </footer>`

    return footer;
};
```

### *A Note about Parameters*

Durable Functions support a single, but not multiple, parameters between functions. What if we have more than one parameter to send? We will have to wrap them into a JSON object and consume the JSON properties in the receiving function. 

# Client Technology

We need a client technology to provide input to the HTTP-triggered *Durable* Function, monitor the status of the *Durable* Function's *Activity* Functions, and record the results of the *Durable* Function as finished HTML-formatted templates. We can use pretty much language and technology that can call an API and store information into an array. Since we needed this quick we will use PowerShell. 

## Our Client script looks like this

1. Read in Excel worksheet information.

```powershell
[string]$excel = (Read-Host -Prompt "Excel worksheet name"),
[string]$worksheet = (Read-Host -Prompt "Which Worksheet?")

$info = Import-Excel -Path ".\Results\$num\$excel" -WorksheetName $worksheet
```

2. For each row convert the row information into a JSON object and create a unique HTML file name based on JSON information fields.

```powershell
$i = 0;
$jobs = @();

$info | ForEach-Object {
    $i++;
    
    $props = @{
        "num" = $num
        "localId" = "$($num)_$i";
        "uri" = $uri;
        "outfile" = "";
        "azureId" = "";
        "statusQueryGetUri" = "" ;
    };

    $job = New-Object -TypeName psobject -Property $props;
```

3. Send the template number, HTML file name and JSON object to an asynchronous operation. 
```powershell
Start-WFTemplate -job $job -body $_ -AsJob -JobName $job.localId | Out-Null;
```
4. Asynchronous operation sends the template number, as a URL query parameter, and the JSON payload as the body to the HTTP Durable Function *Trigger* Function. 
5. Asynchronous operation creates an object to hold the template number, output file name and returned `statusQueryGetUri`. 
6. Client script stores the returned object into an array.
7. Client script waits for all rows to be read and all asynchronous operations to completed. 
8. Client script then queries each `statusQueryGetUri` from the array objects and, when status is ***200 "Completed"***, converts to the content/output from a JSON object and records the result into a uniquely named HTML file. 

# statusQueryGetUri

Notice that we only call two HTTP endpoints: *Trigger* Function and `statusQueryGetUri`. The *Trigger* Function we have already gone over but what is up with the `statusQueryGetUri`address? A *Trigger* Function asynchronously calls an *Orchestrator* function and immediately returns an ***HTTP 202 "Received"*** code with several URLs that can be queried, including a `statusQueryGetUri`. Querying the `statusQueryGetUri` will return the status of the Durable Function's *Activity* function(s). Querying the `statusQueryGetUri` later will give the *Orchestrator* Function time to run *Activity* Function(s) and return the `statusCode` and results. When the `statusCode` is **200 "Complete"** the result of the Durable Function can be used as input into another operation, such as exporting the returned HTML-formatted string to an HTML file on the file system.
