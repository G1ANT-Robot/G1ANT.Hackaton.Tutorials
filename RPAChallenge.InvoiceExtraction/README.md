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
call ProcessWebService query /v1/organizations method GET
dialog ♥result

procedure ProcessWebService query ‴‴ json ‴‴ filename ‴‴ method ‴POST‴
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

Everything looks fine. All fields were detected in the correct way. So we can write the rest of our script.

