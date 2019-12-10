# RPAChallenge.com Movie Search

You can find out this excersise on the website [RPAChallenge.com//MovieSearch](http://rpachallenge.com/movieSearch):

1. The goal of this challenge is to create a workflow that will check which movie review is positive or negative.
2. Add 3 movies to your list in order to start the challenge. You can either search for them or get a list of 3 popular movies
3. Click on each movie in your Movie List in order to see the reviews. Check each review and see if it is positive or negative. Once you have checked all the reviews, press on Submit in order to see your score.

But, we still can find HTML items by 
like in our previous Excersise - 
We highly recommend to understand example [Find Forms](https://github.com/G1ANT-Robot/G1ANT.Hackaton.Tutorials/tree/master/RPAChallenge.InputForms), where is explanation how to use 
**getElementByXpath** function and [XPath](https://www.w3schools.com/xml/xpath_syntax.asp).

## Click GET POPULAR MOVIES

Open Chrome browser and click Inspect on the **GET POPULAR MOVIES** button. 
We can put this expression into Console tab, and we will see, that it's the correct button.

![inspect1](inspect.jpg)

```JavaScript
getElementByXpath('//button[2]/text()')
```

So we can enter first code in G1ANT

```G1ANT
selenium.open chrome url http://rpachallenge.com/movieSearch
window ‴Rpa Challenge - Google Chrome‴ timeout 100000
selenium.click search //button[2] by xpath
```

After execution, the movie list should appear on the screen, on the left. Inspect first element.

![inspect2](inspect2.jpg)

so, we can find all element in the console:

```JavaScript
getElementByXpath('//div[@_ngcontent-c1][6]/span')
getElementByXpath('//div[@_ngcontent-c1][7]/span')
getElementByXpath('//div[@_ngcontent-c1][8]/span')
```

and this example will close the modal window:

```JavaScript
getElementByXpath('//i[@class="material-icons right modal-action modal-close "]').click()
```

and We can extend our G1ANT script to clicks that elements

```G1ANT
selenium.open chrome url http://rpachallenge.com/movieSearch
window ‴Rpa Challenge - Google Chrome‴ style maximize timeout 100000
selenium.click search //button[2] by xpath
for ♥index from 1 to 3
    call ProcessMovie number ♥index
end

procedure ProcessMovie number 1
    ♥itemNumber = ♥number + 5
    selenium.click search //div[@_ngcontent-c1][♥itemNumber]/span by xpath
    delay 1
    selenium.click search ‴//i[@class="material-icons right modal-action modal-close "]‴ by xpath
end
```

## How to detect sentiment?

OK, we have all the reviews and how we can detect, which text is positive or negative? 
It's time to use external AI webservice, like 
[Azure Sentiment Analysis API](https://azure.microsoft.com/en-us/services/cognitive-services/text-analytics/)

* The sentiment analyzer classifies text as predominantly positive or negative. It assigns a score in the range of 0 to 1. Values close to 0.5 are neutral or indeterminate. A score of 0.5 indicates neutrality. When a string can't be analyzed for sentiment or has no sentiment, the score is always 0.5 exactly. For example, if you pass in a Spanish string with an English language code, the score is 0.5.
* Output is returned immediately. You can stream the results to an application that accepts JSON or save the output to a file on the local system. Then, import the output into an application that you can use to sort, search, and manipulate the data.

Remember, to use that service it is neccessary to create Microsoft.IotHUB on Microsoft Azure.

![Text Analytics Create](textanalytics-create.jpg)

There is explanation, how that service is working: 
[Sentiment Analysis is difficult, but AI may have an answer.](https://towardsdatascience.com/sentiment-analysis-is-difficult-but-ai-may-have-an-answer-a8c447110357)

And this is full documentation: [Text Analytics API (v2.1)](https://westcentralus.dev.cognitive.microsoft.com/docs/services/TextAnalytics-v2-1/operations/56f30ceeeda5650db055a3c9)

Come back to coding. We can find all the reviews by this Xpath:

```JavaScript
getElementByXpath('//div[@class="col s8 m8 l8 reviewsText"]/div[2]/div/p')
```

Whereas [2] means review index. The full code in G1ANT:

```G1ANT
selenium.open chrome url http://rpachallenge.com/movieSearch
window ‴Rpa Challenge - Google Chrome‴ style maximize timeout 100000
selenium.click search //button[2] by xpath
for ♥index from 1 to 3
    call ProcessMovie number ♥index
end

procedure ProcessMovie number 1
    ♥itemNumber = ♥number + 5
    selenium.click search //div[@_ngcontent-c1][♥itemNumber]/span by xpath
    for ♥reviewNumber from 1 to 5
        call ProcessReview number ♥reviewNumber
    end
    selenium.click search ‴//i[@class="material-icons right modal-action modal-close "]‴ by xpath
end

procedure ProcessReview number 1
    selenium.getattribute innerHTML search ‴//div[@class="col s8 m8 l8 reviewsText"]/div[♥number]/div/p‴ by xpath result ♥review
    call Sentiment text ♥review
    if ♥positive
        selenium.click search ‴//div[@class="col s8 m8 l8 reviewsText"]/div[♥number]/div/a[1]‴ by xpath 
    else
        selenium.click search ‴//div[@class="col s8 m8 l8 reviewsText"]/div[♥number]/div/a[2]‴ by xpath 
    end
end
```

## REST webservice

Following to [Microsoft's documentation](https://westcentralus.dev.cognitive.microsoft.com/docs/services/TextAnalytics-v2-1/operations/56f30ceeeda5650db055a3c9)
we should send Json request with the review into Azure.

```json
{
  "documents": [
    {
      "language": "en",
      "id": "1",
      "text": "Hello world. This is some input text that I love."
    }
  ]
```

And the answer will be like that:

```json
{
  "documents": [
    {
      "id": "1",
      "score": 0.92
    }
  ]
```

Whereas score >0.5 means positive review. 
There are also CURL example how to prepare query.


```
curl -v -X POST "https://westcentralus.api.cognitive.microsoft.com/text/analytics/v2.1/sentiment?showStats={boolean}"
-H "Content-Type: application/json"
-H "Ocp-Apim-Subscription-Key: {subscription key}"

--data-ascii "{body}" 
```

We can use C# snippet to create all web request to Microsoft Sentiment service:

```G1ANT
procedure Sentiment text ‴‴
    ♥body = ⟦text⟧{ "documents": [ {"language": "en", "id": "1", "text": "♥text" } ] }
    ⊂
        System.Net.WebClient client = new System.Net.WebClient();
        client.Encoding = System.Text.Encoding.UTF8;
        client.Headers.Add("Content-Type", "application/json");
        client.Headers.Add("Ocp-Apim-Subscription-Key", "19220d67553b4068ada632f6d526d1fa");
        string url = "https://g1ant-textanalytics.cognitiveservices.azure.com/text/analytics/v2.1/sentiment";
        string review = ♥body;
        string json = client.UploadString(url, "POST", review);
        return json;
    ⊃
    ♥score = ⟦float⟧♥result⟦documents[0].score⟧
    ♥positive = ♥score>0.5    
end
```

The last thing. We should click START button before first for loop and SUBMIT button after. 
The full code below:

```G1ANT
selenium.open chrome url http://rpachallenge.com/movieSearch
window ‴Rpa Challenge - Google Chrome‴ style maximize timeout 100000
selenium.click search //button[2] by xpath
selenium.click search ‴//button[@class=" uiColorButton waves-effect col s12 m12 l12 btn-large"]‴ by xpath
for ♥index from 1 to 3
    call ProcessMovie number ♥index
end
selenium.click search ‴//button[@class="uiColorButton waves-effect col s12 m12 l12 btn-large"]‴ by xpath

procedure ProcessMovie number 1
    ♥itemNumber = ♥number + 5
    selenium.click search //div[@_ngcontent-c1][♥itemNumber]/span by xpath
    for ♥reviewNumber from 1 to 5
        call ProcessReview number ♥reviewNumber
    end
    selenium.click search ‴//i[@class="material-icons right modal-action modal-close "]‴ by xpath
end

procedure ProcessReview number 1
    selenium.getattribute innerHTML search ‴//div[@class="col s8 m8 l8 reviewsText"]/div[♥number]/div/p‴ by xpath result ♥review
    call Sentiment text ♥review
    if ♥positive
        selenium.click search ‴//div[@class="col s8 m8 l8 reviewsText"]/div[♥number]/div/a[1]‴ by xpath 
    else
        selenium.click search ‴//div[@class="col s8 m8 l8 reviewsText"]/div[♥number]/div/a[2]‴ by xpath 
    end
end

procedure Sentiment text ‴‴
    ♥body = ⟦text⟧{ "documents": [ {"language": "en", "id": "1", "text": "♥text" } ] }
    ⊂
        System.Net.WebClient client = new System.Net.WebClient();
        client.Encoding = System.Text.Encoding.UTF8;
        client.Headers.Add("Content-Type", "application/json");
        client.Headers.Add("Ocp-Apim-Subscription-Key", "19220d67553b4068ada632f6d526d1fa");
        string url = "https://g1ant-textanalytics.cognitiveservices.azure.com/text/analytics/v2.1/sentiment";
        string review = ♥body;
        string json = client.UploadString(url, "POST", review);
        return json;
    ⊃
    ♥score = ⟦float⟧♥result⟦documents[0].score⟧
    ♥positive = ♥score>0.5    
end
```

It will be faster to send all reviews in one part to web service, 
but that task you can do alone.

Execute that example and you will see congratulations form.

![Congratulations](congratulations.jpg)

<!--

## Let's hack together

Why only 80%? We would like to have better results that sentiment service gives us. 

If we put all the review into google, we will see exact result in IMDB.com service, 
but that feature doesn't work for all reviews :)

![IMDB.com](imdb.jpg)

Maybe it's a better way to understand which review is positive or negative. 

This is the new Sentiment procedure:

1. Open Google.com tab
2. Enter full text review and click Search
3. Click on first link (it's the IMDB.com)
4. Take star points
5. Setup positive on true if points higher than 5.

That's all. This is the full code:

```G1ANT
```

-->