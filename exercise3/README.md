# Exercise 3 - Getting Recommendations from Personalize

## Overview

After you create a campaign using Amazon Personalize, you are able to get two different types of recommendations, dependent on what recipe type was used to train the model. For user-personalization and related-items recipes, the GetRecommendations API returns a list of recommended items. For example, products or content can be recommended for users signed in to your website.

For search-personalization recipes, the PersonalizeRanking API re-ranks a list of recommended items based on a specified query.

In this workshop we have been focused on building a user-personalization solution, so far trained on historical event data from Segment. In this exercise we will demonstrate how you can integrate recommendations from Personalize into your applications using a REST API. Once our API is in place, we will enhance it to demonstrate how traits on Segment customer profiles, such as recent purchases, can be used to filter recommendations from Personalize.

### What You'll Be Building

We will start by building an API Gateway endpoint that calls a Lambda function to fetch recommendations from Personalize. This example will show how to build a basic API endpoint to call Personalize directly from your applications for use cases where you will want to directly get recommendations. Then we will enhance the Lambda function to integrate calls to Segment Personas.

### Exercise Preparation

If you haven't already cloned this repository to your local machine, do so now.

```bash
git clone https://github.com/james-jory/segment-personalize-workshop.git
```

## Part 1 - Create API Endpoint & Lambda Function

![Exercise 3 Architecture](images/Architecture-Exercise3-Part1.png)

First we will create a Lambda function that will be called by an API Gateway endpoint. In the AWS console for the account you've been assigned for the workshop, browse to the Lambda service page. Click the "Create function" button to create a new function.

![Lambda Create Function](images/LambdaCreateFunction.png)

Enter a name for your function and specify Python 3.7 as the runtime. Expand the "Permissions" panel and select an existing IAM role, that has already been created for you, with a name like `module-personalize-SegmentPersonalizeLambdaRole-...`). Click "Create function".

![Lambda Function Config](images/LambdaRecEndpointCreate.png)

Scroll down to the "Function code" panel. The source code for the function has already been written and is provided in this repository at [recommendations/lambda_function.py](recommendations/lambda_function.py). Open this file in a new browser tab/window, copy it to your clipboard, and paste it into the source code editor for our Lambda function as shown below. Click the "Save" button at the top of the page when you're done.

![Lambda Function Code](images/LambdaRecCode.png)

### Wire up Personalize API using Lambda Layer (Preview only)

You will notice in the function source the following `import` and function call.

```python
import of import init_personalize_api as api_helper
...
api_helper.init()
```

This `import` and function call utilize some boilerplate code, packaged as a [Lambda Layer](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html), needed to configure the Personalize API with the AWS Python SDK. ***This is only necessary while Personalize is in Preview. Once Personalize is GA and the API is bundled with the Python SDK, as well as other language SDKs, this supporting Layer will no longer be needed.*** For now, though, we need to install this Layer once so we can use it across the functions we build in this workshop.

To install our Layer, open the Lambda navigation panel and click "Layers".

![Lambda Nav Panel](images/LambdaNav.png)

![Lambda Layers Nav](images/LambdaLayersNav.png)

From the Lambda Layers view, click the "Create layer" button.

![Lambda Create Layer](images/LambdaCreateLayer.png)

Create the layer by specifying a name such as "PersonalizeApiInstaller", browsing to the pre-made zip in this repository at `support/layer/python_personalize_init.zip`, and selecting Python 3.7 as the compatible runtime. Click the "Create" button to upload the zip file and create the layer.

![Lambda Create Layer Config](images/LambdaCreateLayerConfig.png)

Next we need to add the layer just created to our function. Return to the Lambda function by opening the Lambda navigation panel and clicking "Functions".

![Lambda Nav Panel](images/LambdaNav.png)

![Lambda Function Layer Add](images/LambdaFunctionsNav.png)

Click on your function name to access the configuration page again. In the Lambda Designer, click the "Layers" panel below the function name and then the "Add layer" button in the "Referenced layers" panel at the bottom of the page.

![Lambda Function Layer Add](images/LambdaLayerAdd.png)

Select the layer we just added and the latest version. Click "Add" to add the layer to the function.

![Lambda Function Layer Add](images/LambdaLayerAddSelect.png)

Next, we will connect Amazon API Gateway to our Lambda funciton. Select "API Gateway" in the "Add triggers" panel in the Designer panel.

![Lambda API Gateway Trigger](images/LambdaRecAPIGW_Trigger.png)

Scroll down to the "Configure triggers" panel. For the API dropdown, select "Create a new API" and set the Security as "Open". For a production deployment you would want to [control access](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-control-access-to-api.html) to this endpoint but that is beyond the scope of this exercise.

![Lambda API Gateway Config](images/LambdaRecAPIGW_Config.png)

Click "Add" to add API Gateway as a trigger to our function and then click "Save" at the top of the page to save our changes.

Next, we need to add environment variables for Segment and for the function to tell it the Personalize Campaign to call for retrieving recommendations.

To obtain the Personalize Campaign ARN, browse to the Personalize service landing page in the AWS console. Select the Dataset Group you created earlier and then Campaigns in the left navigation. Click on the "segment-workshop-campaign" campaign you created earlier and copy the "Campaign arn" to your clipboard.

![Personalize Campaign ARN](images/PersonalizeCampaignArn.png)

Return to our Lambda function and scroll down to the "Environment variables" panel. Add an environment variable with the key `personalize_campaign_arn` and value of the Campaign ARN in your clipboard. Click the "Save" button at the top of the page to save your changes.

![Lambda Campaign ARN Environment Variable](images/LambdaRecCampaignArn.png)

Now let's browse to the API Gateway service page in the AWS console to test our endpoint. Under "APIs" you should see the recommendations API created when we setup our Lambda trigger. Click on the API name.

![API Gateway APIs](images/APIGW_endpoint.png)

Click on the "Test" link to build a test request.

![API Gateway Test](images/APIGW_Test.png)

Select "GET" as the Method and enter a Query String of `userId=2941404340`. This is one of the users in our test dataset. Scroll to the bottom of the page and click the "Test" button.

![API Gateway Test](images/APIGW_TestGet.png)

This will send a request through API Gateway which will call our Lambda function. The function will query Personalize for recommendations and return the results to API Gateway.

![API Gateway Test](images/APIGW_TestGetResults.png)

As you can see, the GetRecommendations endpoint for Personalize returns itemIds for recommended items for the specified user. You would then use these itemIds to retrieve meta infromation from your application such as item names, descriptions, and images to build a user interface for the end-user.

This API framework and function also provides the foundation for enhancing our function to filter recommendations based on recent purchase history for the user/customer in Segment.

## Part 2 - Filtering Recommendations using Customer Profile Traits

For this final step, you will configure Personas real-time computed traits, and then we will update our Lambda function to access those traits to filter recent purchases from the recommendations before they are returned to the caller.  

This example will allow Personas to update a user trait for each user that completes an order via the `Order Completed` event.  So let’s set up that trait first.

Click the Personas orb.

![](https://d2mxuefqeaa7sj.cloudfront.net/s_12A58A92405C56E8DE968E6DE63B3BFED1BB822EB344A5A338801C8B717BCDEB_1551194485012_image.png)

Then select ‘Computed Traits’.

![](https://d2mxuefqeaa7sj.cloudfront.net/s_12A58A92405C56E8DE968E6DE63B3BFED1BB822EB344A5A338801C8B717BCDEB_1551194585193_image.png)

Then click ‘Create Computed Trait’.

![](https://d2mxuefqeaa7sj.cloudfront.net/s_12A58A92405C56E8DE968E6DE63B3BFED1BB822EB344A5A338801C8B717BCDEB_1551194594333_image.png)

Select ‘Unique List.’  This will allow us to compute a list of unique skus from an Order Completed event.

![](https://d2mxuefqeaa7sj.cloudfront.net/s_12A58A92405C56E8DE968E6DE63B3BFED1BB822EB344A5A338801C8B717BCDEB_1551194599821_image.png)


![](https://d2mxuefqeaa7sj.cloudfront.net/s_12A58A92405C56E8DE968E6DE63B3BFED1BB822EB344A5A338801C8B717BCDEB_1551194605292_image.png)

Select the ‘Order Completed’ event in the ‘Select Event..’ dropdown

![](https://d2mxuefqeaa7sj.cloudfront.net/s_12A58A92405C56E8DE968E6DE63B3BFED1BB822EB344A5A338801C8B717BCDEB_1551194610863_image.png)

Then, select the ‘sku’ event property, and then click the ‘Preview’ button.  

Do not select a Time Frame for this trait!  Doing so will disable real-time computation.

![](https://d2mxuefqeaa7sj.cloudfront.net/s_12A58A92405C56E8DE968E6DE63B3BFED1BB822EB344A5A338801C8B717BCDEB_1551194620736_image.png)

This will generate a list of users that have completed orders, and their traits.

![](https://d2mxuefqeaa7sj.cloudfront.net/s_12A58A92405C56E8DE968E6DE63B3BFED1BB822EB344A5A338801C8B717BCDEB_1551194625799_image.png)

Skip this next step by clicking ‘Review & Create’

![](https://d2mxuefqeaa7sj.cloudfront.net/s_12A58A92405C56E8DE968E6DE63B3BFED1BB822EB344A5A338801C8B717BCDEB_1551194631795_image.png)

On this screen, name your trait ‘purchased_products’. This will be used in the Lambda function, so it must match this string or you have to update the Lambda code. Click ‘Create Computed Trait’.

![](https://d2mxuefqeaa7sj.cloudfront.net/s_12A58A92405C56E8DE968E6DE63B3BFED1BB822EB344A5A338801C8B717BCDEB_1551194636961_image.png)

This trait will now be computed for all users that interact with your Segment instance.

In the final [exercise](../exercise4) we will bring everything together and learn how to integrate recommendations from Personalize with your customer profiles in Segment. This allows you to activate recommendations across other integrations in your Segment account.