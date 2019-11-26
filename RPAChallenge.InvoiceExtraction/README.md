# RPAChallenge.com Movie Extraction

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

![queue](queues.jpg)

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
  -F content=@document.pdf 'https://api.elis.rossum.ai/v1/queues/8199/upload' | jq -r .results[0].annotation
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

For security purpose, store your password in Credentials Container from menu Tools, and use that in G1ANT's 
code by special variable `♥credential⟦⟧`.

![Credential Container](credentialcontainer.jpg)

```G1ANT
procedure ProcessWebService url ‴‴ json ‴‴ filename ‴‴
    if ♥token==‴‴
        ⊂
            System.Net.WebClient client = new System.Net.WebClient();
            client.Encoding = System.Text.Encoding.UTF8;
            client.Headers.Add("Content-Type", "application/json");
            string url = "https://api.elis.rossum.ai/v1/auth/login";
            string query = {"username": "chris@g1ant.com", "password": "♥credential⟦Rossum:chris@g1ant.com⟧"};
            return client.UploadString(url, "POST", query);
        ⊃
        ♥token = ♥result⟦.key⟧
        test ♥token!=""
    end
    ⊂
        System.Net.WebClient client = new System.Net.WebClient();
        client.Encoding = System.Text.Encoding.UTF8;
        client.Headers.Add("Content-Type", "application/json");
        client.Headers.Add("Authorization", "token " + ♥token);
        if ♥filename != ""
            return client.UploadFile(♥url, "POST", ♥filename);
        else
            return client.UploadString(♥url, "POST", ♥json);
    ⊃
    ♥json = ♥result
end
```
