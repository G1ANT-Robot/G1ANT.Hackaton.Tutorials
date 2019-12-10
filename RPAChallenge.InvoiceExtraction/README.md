# RPAChallenge.com Invoice Extraction

You can find out this excersise on the website [RPAChallenge.com Invoice Extraction](https://rpachallengeocr.azurewebsites.net/):

Instructions
1. The goal of this challenge is to create a workflow that will read every table row and download the respective invoices.

2. From the invoices, you will have to extract the Invoice Number, Invoice Date, Company Name and Total Due.

3. You will have to build and upload a CSV file with the data extracted from each invoice, the ID and Due Date from the table, only for the invoices for which the Due Date has passed or is today.

4. The actual countdown of the challenge will begin once you click the Start button and will end once the CSV file is uploaded; until then, you may play around with the table on the right without receiving penalties.

5. Below you will find an example CSV file in order to see the required format for the end result and two sample invoices. The formats of the invoices will be exactly as in the samples and they will not change. The challenge expects the uploaded CSV to be in the exact same format as the example CSV, including the formatting of the cells, and the rows should be in the same order as they appear in the table. Any difference will result in a failed challenge.

Attachments:

* [Example CSV](https://rpachallengeocr.azurewebsites.net/invoices/example.csv)
* [Invoice 1](https://rpachallengeocr.azurewebsites.net/invoices/sample1.jpg)
* [Invoice 2](https://rpachallengeocr.azurewebsites.net/invoices/sample2.jpg)

## The solution

We can solve this challenge in two ways:

1. Because there are structured data (two type of invoices), we exactly know where are searched fields, so we can use just Google OCR.
2. The simpliest way is to use external service to invoice data extraction, which can deeply understand where are unstructured data.

## Rossum.AI

Let's use external [Rossum.AI](https://elis.rossum.ai/) service to extract all searched data in the simpliest way. 

There are some documentation files:
* [Extracting invoices using AI in a few lines of code](https://medium.com/@bzamecnik/extracting-invoices-using-ai-in-a-few-lines-of-code-96e412df7a7a)
* [Rossum Getting Started](https://developers.rossum.ai/docs)
* [Rossum API](https://api.elis.rossum.ai/docs/#getting-started)

Register on [that service](https://elis.rossum.ai/) and create queue for US invoices, because as we know, our invoices will come from US region.

![queue](queue.jpg)

Let's check your queue number on the url. In our case the number is 29838.

And setup fields we need to extract to CSV: 

CSV field name | Rossum field name
-------------- | -----------------
DueDate | Due Date
InvoiceNo | Invoice Number
InvoiceDate | Invoice Date
CompanyName | Customer name
TotalDue | Amount Due

![fields](fields1.jpg)

First of all, we should learn Rossum.AI to understand our two type of invoices, to make sure it will work properly.
That's why we have to implement that logic in the G1ANT.Language:

## Rossum.AI flow

1. Login to the account:

```
curl -s -H 'Content-Type: application/json' \
  -d '{"username": "east-west-trading-co@elis.rossum.ai", "password": "aCo2ohghBo8Oghai"}' \
  'https://api.elis.rossum.ai/v1/auth/login'
{"key": "db313f24f5738c8e04635e036ec8a45cdd6d6b03"}
```

2. Upload documents:

```
curl -s -H 'Authorization: token db313f24f5738c8e04635e036ec8a45cdd6d6b03'
  'https://api.elis.rossum.ai/v1/queues?page_size=1' | jq -r .results[0].url
https://api.elis.rossum.ai/v1/queues/8199
```

Then you can upload document to the queue. Alternatively, you can send documents to a queue-related inbox. See upload for more information about importing files.

```
curl -s -H 'Authorization: token db313f24f5738c8e04635e036ec8a45cdd6d6b03' \
  -F content=@sample1.jpg 'https://api.elis.rossum.ai/v1/queues/29838/upload/sample1.jpg' | jq -r .results[0].annotation
https://api.elis.rossum.ai/v1/annotations/319668
```

After that, we should open the Rossum web interface [elis.rossum.ai](https://elis.rossum.ai) 
to review and confirm extracted data.

3. Wait for document to be ready and review extracted data:

```
curl -s -H 'Authorization: token db313f24f5738c8e04635e036ec8a45cdd6d6b03' \
  'https://api.elis.rossum.ai/v1/annotations/319668' | jq .status
"to_review"
```

4. Download reviewed data:

```
curl -s -H 'Authorization: token db313f24f5738c8e04635e036ec8a45cdd6d6b03' \
  'https://api.elis.rossum.ai/v1/queues/8199/export?status=exported&format=csv&id=319668'
Invoice number,Invoice Date,Due date,Customer name,Customer ID,Total amount,
2183760194,2018-06-08,2018-06-08,Rossum,05222322,500.00
```

5. Logout:

```
curl -s -X POST -H 'Authorization: token db313f24f5738c8e04635e036ec8a45cdd6d6b03' \
  'https://api.elis.rossum.ai/v1/auth/logout'
{"detail":"Successfully logged out."}
```

## Rossum in G1ANT

Let's do this in the G1ANT language. We have to create procedure which will send all our queries to Rossum webservice.
First of all, we should login to that webservice. We can do this if variable `♥token` is empty. Otherwise, 
we just send `♥json` (standard query) or `♥filename` argument (upload invoice).

For security reasons, store your password in Credentials Container from menu Tools, and use that in G1ANT's 
code by special variable `♥credential⟦⟧`.

![Credential Container](credentialcontainer.jpg)

Let's check our script by retrieve organisation objects.

```
curl -H 'Authorization: token db313f24f5738c8e04635e036ec8a45cdd6d6b03' \
  'https://api.elis.rossum.ai/v1/organizations'
```

Example in G1ANT below.

```G1ANT
♥token = ‴‴
call Rossum query /v1/organizations method GET
dialog ♥result

procedure Rossum query ‴‴ json ‴‴ filename ‴‴ method ‴POST‴
    if ♥token==""
        ♥authorisation = ⟦text⟧{"username": "chris@g1ant.com", "password": "{password}"}
        ♥url = https://api.elis.rossum.ai/v1/auth/login
        ⊂
            System.Net.ServicePointManager.Expect100Continue = true;
            System.Net.ServicePointManager.SecurityProtocol = System.Net.SecurityProtocolType.Tls12;
            System.Net.WebClient client = new System.Net.WebClient();
            client.Encoding = System.Text.Encoding.UTF8;
            client.Headers.Add("Content-Type", "application/json");
            string json = ♥authorisation.Replace("{password}", ♥credential⟦Rossum:chris@g1ant.com⟧);
            return client.UploadString(♥url, "POST", json);
        ⊃
        ♥token = ♥result⟦.key⟧
        test ♥token!=""
    end
    ♥url = https://api.elis.rossum.ai♥query
    ⊂
        string url = "https://api.elis.rossum.ai" + ♥query;
        System.Net.ServicePointManager.Expect100Continue = true;
        System.Net.ServicePointManager.SecurityProtocol = System.Net.SecurityProtocolType.Tls12;
        if (♥filename != "")
        {
            System.IO.FileInfo file = new System.IO.FileInfo(♥filename);
            int fileLength = (int)file.Length;

            string boundary = "---------------------------" + DateTime.Now.Ticks.ToString("x");
            byte[] boundarybytes = System.Text.Encoding.ASCII.GetBytes("\r\n--" + boundary + "\r\n");

            System.Net.HttpWebRequest request = (System.Net.HttpWebRequest)System.Net.WebRequest.Create(url);
            request.ContentType = "multipart/form-data; boundary=" + boundary;
            request.Method = "POST";
            request.KeepAlive = true;
            request.Headers.Add(System.Net.HttpRequestHeader.Authorization, "token " + ♥token);

            using (System.IO.Stream stream = request.GetRequestStream())
            {
                stream.Write(boundarybytes, 0, boundarybytes.Length);

                byte[] headerbytes = System.Text.Encoding.UTF8.GetBytes(
                    "Content-Disposition: form-data; name=\"content\"; filename=\"" + file.Name + "\"\r\n\r\n");
                stream.Write(headerbytes, 0, headerbytes.Length);

                System.IO.FileStream fileStream = new System.IO.FileStream(♥filename, System.IO.FileMode.Open, System.IO.FileAccess.Read);
                stream.Write(System.IO.File.ReadAllBytes(♥filename), 0, fileLength);
                fileStream.Close();

                byte[] trailer = System.Text.Encoding.ASCII.GetBytes("\r\n--" + boundary + "--\r\n");
                stream.Write(trailer, 0, trailer.Length);
            }
            using (System.IO.Stream stream = request.GetResponse().GetResponseStream())
                return new System.IO.StreamReader(stream).ReadToEnd();
        }
        else if (♥method == "POST")
        {
            System.Net.WebClient client = new System.Net.WebClient();
            //client.Encoding = System.Text.Encoding.UTF8;
            client.Headers.Add("Authorization", "token " + ♥token);
            client.Headers.Add("Content-Type", "application/json");
            return client.UploadString(♥url, ♥method, ♥json);
        }
        else if (♥method == "GET")
        {
            System.Net.WebClient client = new System.Net.WebClient();
            //client.Encoding = System.Text.Encoding.UTF8;
            client.Headers.Add("Authorization", "token " + ♥token);
            return client.DownloadString(♥url);
        }
        else
            throw new NotImplementedException();
    ⊃
end
```

And the result:

![result json](resultjson.jpg)

Ok, it's working! So let's come back to our invoices, and send our two samples into Rossum API.

```G1ANT
♥token = ‴‴
call ProcessWebService query /v1/queues/29838/upload method POST filename c:\Users\Chris\Downloads\Sample1.jpg
dialog ♥result
```

![result json](resultjsonfile.jpg)

And take a look on Rossum's website. The file should appear in the US Invoices queue as exported, soon.

![exported](exported.jpg)

Let's check exported fields by clicking on the sample1.jpg name.

![exported fields](exportedfields.jpg)

## The full process

Everything looks fine. All fields were detected in the correct way. So we can write the rest of our script. 
The Rossum procedure is above. The example json which will be decoded by ReadFile procedure below:

```json
{
  "pagination": {
    "total": 1,
    "total_pages": 1,
    "next": null,
    "previous": null
  },
  "results": [
    {
      "url": "https://api.elis.rossum.ai/v1/annotations/1299441",
      "status": "exported",
      "arrived_at": "2019-12-06T10:29:41.680356Z",
      "exported_at": "2019-12-06T10:30:28.985607Z",
      "document": {
        "url": "https://api.elis.rossum.ai/v1/documents/1300633",
        "file_name": "Invoice1.jpg",
        "file": "https://api.elis.rossum.ai/v1/documents/1300633/content"
      },
      "modifier": null,
      "schema": {
        "url": "https://api.elis.rossum.ai/v1/schemas/249012"
      },
      "metadata": {},
      "content": [
        {
          "category": "section",
          "schema_id": "invoice_details_section",
          "children": [
            {
              "category": "datapoint",
              "schema_id": "invoice_id",
              "rir_confidence": 0.88315943292202548,
              "value": "16996",
              "type": "string"
            },
            {
              "category": "datapoint",
              "schema_id": "date_issue",
              "rir_confidence": 0.99074839277057047,
              "value": "2019-06-20",
              "type": "date"
            },
            {
              "category": "datapoint",
              "schema_id": "date_due",
              "rir_confidence": null,
              "value": "",
              "type": "date"
            }
          ]
        },
        {
          "category": "section",
          "schema_id": "payment_info_section",
          "children": []
        },
        {
          "category": "section",
          "schema_id": "totals_section",
          "children": [
            {
              "category": "datapoint",
              "schema_id": "amount_due",
              "rir_confidence": 0.958716846037662,
              "value": "2100.00",
              "type": "number"
            },
            {
              "category": "datapoint",
              "schema_id": "currency",
              "rir_confidence": null,
              "value": "usd",
              "type": "enum"
            },
            {
              "category": "multivalue",
              "schema_id": "tax_details",
              "children": [
                {
                  "category": "tuple",
                  "schema_id": "tax_detail",
                  "children": []
                }
              ]
            }
          ]
        },
        {
          "category": "section",
          "schema_id": "vendor_section",
          "children": [
            {
              "category": "datapoint",
              "schema_id": "recipient_name",
              "rir_confidence": 0.84859950622828428,
              "value": "ACME Inc.",
              "type": "string"
            }
          ]
        },
        {
          "category": "section",
          "schema_id": "other_section",
          "children": []
        },
        {
          "category": "section",
          "schema_id": "line_items_section",
          "children": [
            {
              "category": "multivalue",
              "schema_id": "line_items",
              "children": [
                {
                  "category": "tuple",
                  "schema_id": "line_item",
                  "children": []
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

Put the code below on the top of your script. 

```G1ANT
♥token = ‴‴
♥queue = 29838

selenium.open chrome url https://rpachallengeocr.azurewebsites.net/
window ‴Automation Challenge - OCR - Google Chrome‴ style maximize timeout 30000
selenium.click search //button[@id="start"] by xpath
for ♥page from 1 to 3
    call ReadTable page ♥page
end
call Rossum query /v1/auth/logout method POST


procedure ReadTable page 1
    selenium.click search //div[@id="tableSandbox_paginate"]/span/a[@data-dt-idx="♥page"] by xpath
    for ♥number from 1 to 4
        selenium.getinnerhtml search //tbody/tr[♥number]/td[3] by xpath result ♥duedate
        if ⊂System.DateTime.Parse(♥duedate) <= System.DateTime.Today⊃
            selenium.getinnerhtml search //tbody/tr[♥number]/td[1] by xpath result ♥index
            selenium.getinnerhtml search //tbody/tr[♥number]/td[2] by xpath result ♥id
            selenium.getattribute href search //tbody/tr[♥number]/td[4]/a by xpath result ♥url
            ♥filename = ♥environment⟦USERPROFILE⟧\Downloads\Invoice♥index.jpg
            call DownloadFile url ♥url filename ♥filename
            call ReadFile index ♥index id ♥id duedate ♥duedate filename ♥filename
        end
    end
end

procedure DownloadFile url ‴‴ filename ‴‴
    ⊂
        if (System.IO.File.Exists(♥filename))
            System.IO.File.Delete(♥filename);
        System.Net.ServicePointManager.Expect100Continue = true;
        System.Net.ServicePointManager.SecurityProtocol = System.Net.SecurityProtocolType.Tls12;
        System.Net.WebClient client = new System.Net.WebClient();
        client.DownloadFile(♥url, ♥filename);
    ⊃
end

procedure ReadFile index 1 id ‴‴ duedate ‴‴ filename ♥filename
    -dialog ‴♥index ♥id ♥duedate ♥url‴
    call Rossum query /v1/queues/♥queue/upload method POST filename ♥filename
    ♥annotation = ⊂♥result⟦annotation⟧.Replace("https://api.elis.rossum.ai", "")⊃
    ♥processing = true
    while ♥processing
        delay 1
        call Rossum query ♥annotation method GET
        ♥rossumid = ♥result⟦id⟧
        ♥processing = ♥result⟦status⟧!="exported"
    end
    call Rossum query /v1/queues/♥queue/export?status=exported&format=json&id=♥rossumid method GET
    
    ♥invoiceno = ♥result⟦results[0].content[0].children[0].value⟧
    test ♥result⟦results[0].content[0].children[0].schema_id⟧=="invoice_id"
    ♥invoicedate = ♥result⟦results[0].content[0].children[1].value⟧
    test ♥result⟦results[0].content[0].children[1].schema_id⟧=="date_issue"
    ♥date_due = ♥result⟦results[0].content[0].children[1].value⟧
    test ♥result⟦results[0].content[0].children[2].schema_id⟧=="date_due"
    ♥totaldue = ♥result⟦results[0].content[2].children[0].value⟧
    test ♥result⟦results[0].content[2].children[0].schema_id⟧=="amount_due"
    ♥companyname = ♥result⟦results[0].content[3].children[0].value⟧
    test ♥result⟦results[0].content[3].children[0].schema_id⟧=="sender_name"
        
    call AppendCsv csvname ♥environment⟦USERPROFILE⟧\Downloads\invoices.csv id ♥id duedate ♥duedate invoiceno ♥invoiceno invoicedate ♥invoicedate companyname ♥companyname totaldue ♥totaldue
end

procedure AppendCsv csvname ‴‴ id ‴‴ duedate ‴‴ invoiceno ‴‴ invoicedate ‴‴ companyname ‴‴ totaldue ‴‴
    ⊂
        if (!System.IO.File.Exists(♥csvname))
            System.IO.File.WriteAllText(♥csvname,
                "ID,DueDate,InvoiceNo,InvoiceDate,CompanyName,TotalDue\r\n");
        System.IO.File.AppendAllText(♥csvname,
            ♥id + "," + ♥duedate + "," + ♥invoiceno + "," + ♥invoicedate + "," + 
            ♥companyname + "," + ♥totaldue + "\r\n"); 
    ⊃
end
```

## Hotfixes

After the script execution, we will have that csvfile:

![csv in excel](excelcsv.jpg)

As we can see, everything working fine this time and all the data was detected properly. 
But remember - this is not the best solution for production environment.
Better way is to use [Generic AI Engine](https://rossum.ai/help/article/do-you-need-generic-or-dedicated-ai-engine/)
which can be trained by our data.

Second thing, all the dates should be in the format dd-MM-yyyy (for example 27-11-2019), but 
some of them, exported from invoices have format yyyy-MM-dd (for exampple 2019-11-27). 
So, whe should put some code to correct that issue.

```G1ANT
    if ⊂♥invoicedate.Substring(2, 1) != "-"⊃
        - yyyy-MM-dd
        ♥year = ⟦text⟧♥invoicedate.Substring(0, 4)
        ♥month = ⟦text⟧♥invoicedate.Substring(5, 2)
        ♥day = ⟦text⟧♥invoicedate.Substring(8, 2)
        ♥invoicedate = ‴♥day-♥month-♥year‴
    end
    test ⊂♥invoicedate.Length == 10 && ♥invoicedate.Substring(2, 1) == "-" && ♥invoicedate.Substring(5, 1) == "-" ⊃
```

## Upload CSV

The last thing is to upload generated file into csv file:

```G1ANT
procedure AppendCsv csvname ‴‴ id ‴‴ duedate ‴‴ invoiceno ‴‴ invoicedate ‴‴ companyname ‴‴ totaldue ‴‴
    ⊂
        if (!System.IO.File.Exists(♥csvname))
            System.IO.File.WriteAllText(♥csvname,
                "ID,DueDate,InvoiceNo,InvoiceDate,CompanyName,TotalDue\r\n");
        System.IO.File.AppendAllText(♥csvname,
            ♥id + "," + ♥duedate + "," + ♥invoiceno + "," + ♥invoicedate + "," + 
            ♥companyname + "," + ♥totaldue + "\r\n"); 
    ⊃
end
```

And we should click submit button and sent all the data into rpachallenge.com.

```G1ANT
selenium.click search //input[@type="file"] by xpath
window open timeout 30000
keyboard ♥environment⟦USERPROFILE⟧\Downloads\invoices.csv
keyboard ⋘enter⋙
excel.open ♥environment⟦USERPROFILE⟧\Downloads\invoices.csv
```

Execute script. After short time, you should receive message:

![congrats](congrats.jpg)

## Faster way

The script will work much faster if we put all the data into Rossum first, and on the end 
just download all the result. This is the job for you.

## The full script

```G1ANT
♥token = ‴‴
♥queue = 29838

file.delete ♥environment⟦USERPROFILE⟧\Downloads\invoices.csv errorjump next
label next
selenium.open chrome url https://rpachallengeocr.azurewebsites.net/
window ‴Automation Challenge - OCR - Google Chrome‴ style maximize timeout 30000
selenium.click search //button[@id="start"] by xpath
for ♥page from 1 to 3
    call ReadTable page ♥page
end
call Rossum query /v1/auth/logout method POST
selenium.click search //input[@type="file"] by xpath
window open timeout 30000
keyboard ♥environment⟦USERPROFILE⟧\Downloads\invoices.csv
keyboard ⋘enter⋙
excel.open ♥environment⟦USERPROFILE⟧\Downloads\invoices.csv

-selenium.click //input[@type=file] by xpath


procedure Rossum query ‴‴ json ‴‴ filename ‴‴ method ‴POST‴
    if ♥token==""
        ♥authorisation = ⟦text⟧{"username": "chris@g1ant.com", "password": "{password}"}
        ♥url = https://api.elis.rossum.ai/v1/auth/login
        ⊂
            System.Net.ServicePointManager.Expect100Continue = true;
            System.Net.ServicePointManager.SecurityProtocol = System.Net.SecurityProtocolType.Tls12;
            System.Net.WebClient client = new System.Net.WebClient();
            client.Encoding = System.Text.Encoding.UTF8;
            client.Headers.Add("Content-Type", "application/json");
            string json = ♥authorisation.Replace("{password}", ♥credential⟦Rossum:chris@g1ant.com⟧);
            return client.UploadString(♥url, "POST", json);
        ⊃
        ♥token = ♥result⟦.key⟧
        test ♥token!=""
    end
    ♥url = https://api.elis.rossum.ai♥query
    ⊂
        string url = "https://api.elis.rossum.ai" + ♥query;
        System.Net.ServicePointManager.Expect100Continue = true;
        System.Net.ServicePointManager.SecurityProtocol = System.Net.SecurityProtocolType.Tls12;
        if (♥filename != "")
        {
            System.IO.FileInfo file = new System.IO.FileInfo(♥filename);
            int fileLength = (int)file.Length;

            string boundary = "---------------------------" + DateTime.Now.Ticks.ToString("x");
            byte[] boundarybytes = System.Text.Encoding.ASCII.GetBytes("\r\n--" + boundary + "\r\n");

            System.Net.HttpWebRequest request = (System.Net.HttpWebRequest)System.Net.WebRequest.Create(url);
            request.ContentType = "multipart/form-data; boundary=" + boundary;
            request.Method = "POST";
            request.KeepAlive = true;
            request.Headers.Add(System.Net.HttpRequestHeader.Authorization, "token " + ♥token);

            using (System.IO.Stream stream = request.GetRequestStream())
            {
                stream.Write(boundarybytes, 0, boundarybytes.Length);

                byte[] headerbytes = System.Text.Encoding.UTF8.GetBytes(
                    "Content-Disposition: form-data; name=\"content\"; filename=\"" + file.Name + "\"\r\n\r\n");
                stream.Write(headerbytes, 0, headerbytes.Length);

                System.IO.FileStream fileStream = new System.IO.FileStream(♥filename, System.IO.FileMode.Open, System.IO.FileAccess.Read);
                stream.Write(System.IO.File.ReadAllBytes(♥filename), 0, fileLength);
                fileStream.Close();

                byte[] trailer = System.Text.Encoding.ASCII.GetBytes("\r\n--" + boundary + "--\r\n");
                stream.Write(trailer, 0, trailer.Length);
            }
            using (System.IO.Stream stream = request.GetResponse().GetResponseStream())
                return new System.IO.StreamReader(stream).ReadToEnd();
        }
        else if (♥method == "POST")
        {
            System.Net.WebClient client = new System.Net.WebClient();
            //client.Encoding = System.Text.Encoding.UTF8;
            client.Headers.Add("Authorization", "token " + ♥token);
            client.Headers.Add("Content-Type", "application/json");
            return client.UploadString(♥url, ♥method, ♥json);
        }
        else if (♥method == "GET")
        {
            System.Net.WebClient client = new System.Net.WebClient();
            //client.Encoding = System.Text.Encoding.UTF8;
            client.Headers.Add("Authorization", "token " + ♥token);
            return client.DownloadString(♥url);
        }
        else
            throw new NotImplementedException();
    ⊃
end

procedure ReadTable page 1
    selenium.click search //div[@id="tableSandbox_paginate"]/span/a[@data-dt-idx="♥page"] by xpath
    for ♥number from 1 to 4
        selenium.getinnerhtml search //tbody/tr[♥number]/td[3] by xpath result ♥duedate
        if ⊂System.DateTime.Parse(♥duedate) <= System.DateTime.Today⊃
            selenium.getinnerhtml search //tbody/tr[♥number]/td[1] by xpath result ♥index
            selenium.getinnerhtml search //tbody/tr[♥number]/td[2] by xpath result ♥id
            selenium.getattribute href search //tbody/tr[♥number]/td[4]/a by xpath result ♥url
            ♥filename = ♥environment⟦USERPROFILE⟧\Downloads\Invoice♥index.jpg
            call DownloadFile url ♥url filename ♥filename
            call ReadFile index ♥index id ♥id duedate ♥duedate filename ♥filename
        end
    end
end

procedure DownloadFile url ‴‴ filename ‴‴
    ⊂
        if (System.IO.File.Exists(♥filename))
            System.IO.File.Delete(♥filename);
        System.Net.ServicePointManager.Expect100Continue = true;
        System.Net.ServicePointManager.SecurityProtocol = System.Net.SecurityProtocolType.Tls12;
        System.Net.WebClient client = new System.Net.WebClient();
        client.DownloadFile(♥url, ♥filename);
    ⊃
end

procedure ReadFile index 1 id ‴‴ duedate ‴‴ filename ♥filename
    call Rossum query /v1/queues/♥queue/upload method POST filename ♥filename
    ♥annotation = ⊂♥result⟦annotation⟧.Replace("https://api.elis.rossum.ai", "")⊃
    ♥processing = true
    while ♥processing
        delay 1
        call Rossum query ♥annotation method GET
        ♥rossumid = ♥result⟦id⟧
        ♥processing = ♥result⟦status⟧!="exported"
    end
    call Rossum query /v1/queues/♥queue/export?status=exported&format=json&id=♥rossumid method GET
    
    ♥invoiceno = ♥result⟦results[0].content[0].children[0].value⟧
    test ♥result⟦results[0].content[0].children[0].schema_id⟧=="invoice_id"
    ♥invoicedate = ♥result⟦results[0].content[0].children[1].value⟧
    if ⊂♥invoicedate.Substring(2, 1) != "-"⊃
        ♥year = ⟦text⟧♥invoicedate.Substring(0, 4)
        ♥month = ⟦text⟧♥invoicedate.Substring(5, 2)
        ♥day = ⟦text⟧♥invoicedate.Substring(8, 2)
        ♥invoicedate = ‴♥day-♥month-♥year‴
    end
    test ⊂♥invoicedate.Length == 10 && ♥invoicedate.Substring(2, 1) == "-" && ♥invoicedate.Substring(5, 1) == "-" ⊃
    test ♥result⟦results[0].content[0].children[1].schema_id⟧=="date_issue"
    ♥date_due = ♥result⟦results[0].content[0].children[1].value⟧.Replace("/", "-")
    test ♥result⟦results[0].content[0].children[2].schema_id⟧=="date_due"
    ♥totaldue = ♥result⟦results[0].content[2].children[0].value⟧
    test ♥result⟦results[0].content[2].children[0].schema_id⟧=="amount_due"
    ♥companyname = ♥result⟦results[0].content[3].children[0].value⟧
    test ♥result⟦results[0].content[3].children[0].schema_id⟧=="sender_name"
  
    call AppendCsv csvname ♥environment⟦USERPROFILE⟧\Downloads\invoices.csv id ♥id duedate ♥duedate invoiceno ♥invoiceno invoicedate ♥invoicedate companyname ♥companyname totaldue ♥totaldue
end

procedure AppendCsv csvname ‴‴ id ‴‴ duedate ‴‴ invoiceno ‴‴ invoicedate ‴‴ companyname ‴‴ totaldue ‴‴
    ⊂
        if (!System.IO.File.Exists(♥csvname))
            System.IO.File.WriteAllText(♥csvname,
                "ID,DueDate,InvoiceNo,InvoiceDate,CompanyName,TotalDue\r\n");
        System.IO.File.AppendAllText(♥csvname,
            ♥id + "," + ♥duedate + "," + ♥invoiceno + "," + ♥invoicedate + "," + 
            ♥companyname + "," + ♥totaldue + "\r\n"); 
    ⊃
end
```