# RPAChallenge.com Shortest Path

You can find out this excersise on the website [RPAChallenge.com Shortest Path](http://rpachallenge.com/assets/shortestPath/public/shortestpath.html):

1. The goal of this challenge is to create an attended robot that works in tandem with the user.
2. The user needs to match one Demand (marked as a RED point) to the closest Supply (marked as a GREEN point) which will populate two tables with data.
3. The robot needs to fill the required data inside the form using the data in the tables generated at step 2. The form will return a contract value that needs to be inserted back in the 'Contract' text box.
4. Repeat until there are no more RED points on the map.

![Map](map.jpg)

When you click "red" point, and after that "green" point, 
you will see two filled tables like that:

![Tables](tables.jpg)

The difficulties based on captcha used within "Demand" and "Supply" labels, 
and our OCR recognition should be very accurate. We still can find HTML items by [XPath](https://www.w3schools.com/xml/xpath_syntax.asp) 
like in our previous Excersise - [Find Forms](https://github.com/G1ANT-Robot/G1ANT.Hackaton.Tutorials/tree/master/RPAChallenge.InputForms)
We highly recommend to understand that example, where is explanation how to use 
**getElementByXpath** function.

