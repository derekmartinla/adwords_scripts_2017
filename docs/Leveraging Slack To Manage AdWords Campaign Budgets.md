## Leveraging Slack To Manage AdWords Campaign Budgets

When it comes to managing paid search campaigns, it never hurts to have an extra set of eyes. One common issue that I've encountered when managing AdWords accounts is pacing — how does one ensure that a campaign hasn't exceeded it's daily or monthly budget? 

Getting this question wrong can, at best, lead to client disappointment and at worst, lead to practictioners (agencies and/or freelancers) having to issue credits back to clients. This is never a good time!

Thankfully, Google AdWords Scripts combined with the Slack API can serve as an extra layer of monitoring for accounts. In this tutorial, I'll show you how to leverage these tools to create your own campaign budget monitor. 

> Essentially, the code shown below will review your active campaigns for high spend and send a Slack alert to the channel/user of your choice 

Let's get started! 

### Setting up the Slack API

Before we dive into the AdWords Script, lets make sure that you have the proper Slack credentials. Head over to the Slack API page, located here [Slack API page](https://api.slack.com/apps#)

> Make sure that you are logged into the Slack account that you plan on sending alerts to.

Next, click the **'Create New App'** button (screenshot below), and enter your new **App Name** and appropriate development team — in my case it is 3Q Digital. Make sure that you click **'Create App'** and scroll down the page to find out your new App's **ClientID** and **Client Secret** values.

![Screen Shot 2017-06-05 at 7.14.12 PM](/Users/derekmartin/Desktop/Screen Shot 2017-06-05 at 7.14.12 PM.png)

![Screen Shot 2017-06-05 at 7.15.31 PM](/Users/derekmartin/Desktop/Screen Shot 2017-06-05 at 7.15.31 PM.png)



> If you're using a corporate account (i.e. 3Q Digital), you'll need to reach out to your Slack Channel admins and ask them to approve your app.  

You now have the keys to the castle. Before we move on, we'll need to setup our incoming **webhooks**.

### Enabling Slack Webhooks

In order to leverage Slack, we will need a way to send messages via our Google AdWords Script that are of interest to the end user. We'll use the Slack API's **Incoming Webhooks** feature to accomplish this task. 

I'll go into detail as to how this works later in the tutorial but for now, navigate to your new app's configuration page and select **Incoming Webhooks** (screenshot below).

>  According to Slack, webhooks are a simple way to post messages from external sources (i.e. AdWords) into Slack. 

![Incoming Webhooks](/Users/derekmartin/Desktop/Screen Shot 2017-06-05 at 7.48.59 PM.png)

In order to use Webhook's, you'll need to decide **where** you want to send Slack messages. This could be a common team channel, an channel created specficially for alerts, or an individual user. 

> **Note:** You'll need to create a new Slack Webhook each time you want to send messages to a different user or channel.

Navigate to the bottom of the incoming Webhooks page and click the **Add New Webhook To Team** button (screenshot below). From here, select the user/channel that will receive alerts and click **Authorize**. 

![Screen Shot 2017-06-05 at 7.53.22 PM](/Users/derekmartin/Desktop/Screen Shot 2017-06-05 at 7.53.22 PM.png)

Upon completion, you should see a new row that contains your new Slack Webhook URL address (screenshot below). 

![Screen Shot 2017-06-05 at 7.54.31 PM](/Users/derekmartin/Desktop/Screen Shot 2017-06-05 at 7.54.31 PM.png)

>  Copy and paste this URL into a text editor — we'll need it later.

Make sure to take note of your credentials and head over to the AdWords UI tab. 

### Creating The Script Outline 

#### TO MCC or Not to MCC 

Before we dive in to the script, lets chat briefly about **where** to create the script. If you're working in an agency setting, I highly encourage you to consider running your custom AdWords Script at the **MCC (My Client Center)** level. 

>  Running your scripts at a MCC level allows you to **a)** apply your code to multiple clients and **b)** to retain control of your code in the case that you and a client part ways.

In this tutorial, I'll show you both the MCC approach and the non-MCC approach. Make sure to create your new script — in either the MCC or individual account — and let's get started!

#### Detour — Leveraging External JavaScript Libraries

Since AdWords Scripts are written in **JavaScript**, we can leverage popular utilities like the popular [**Underscore**](http://underscorejs.org/) and [MomentJS](https://momentjs.com/) library. These libraries will help us deal with numerous tedious tasks — like map/reduce, date formatting, and loop iteration — in ways that are both easier to understand and syntatically cleaner than vanilla JavaScript. 

In order to make life easier, I've ported over both libraries so that they work within Google AdWords. You can find these ports on my [GitHub](https://github.com/derekmartinla/adwords_scripts_2017/tree/master/external_libraries). It's good practice to copy and paste these libraries into the bottom of new scripts so that you can use them. 

> **UnderscoreJS** - provides a numer of useful functional programming helpers 
>
> **MomentJS** - handles the parsing, validatation, manipulation, and display of dates and times in JavaScript.

All code in the rest of this tutorial assumes that you've included **Underscore**.

#### Creating the MCC Script Outline

Let's create the basic skeleton for our MCC script. Make sure that you have the relevant Google AdWords Customer IDs (**CIDS**) before using the code below. 

> **Note:** Remove all dashes from your CIDs when adding them to your script. 

```javascript
function monitorCampaigns(account) {
 // do something 
 Logger.log(account.getName());
}

function main() {
  var accountIter = MccApp.accounts()
    .withIds([CID1, CID2, CID3,]) // You can list up to 50 CIDs 
    .get();

    while(accountIter.hasNext()) {
     var account = accountIter.next();
     MccApp.select(account);

     monitorCampaigns(account);
    }  
} 

```

> **Note:** If you are not using a MCC, simply running the code associated with the **monitorCampaigns()** function should suffice.

In the code above, we use a MCCApp account **selector** to select the AdWords accounts that we'd like to monitor. We iterate over each account, select it — so that we can manipulate it if necessary — and then pass the account to a function called **monitorCampaigns()**.

> **monitorCampaigns(account)** will pull all active campaigns, check their spend levels, and send relevant Slack messages as applicable. 

### Defining Relevant URLs & Monitoring Thresholds 

 At this point if you were to run the script, you'd simply get a console message containing your selected AdWords accounts. 

We can now start defining the relevant urls and monitoring **thresholds** that will be used on excution. Add the following code to the top of the script:

```javascript
var SLACK_URL = 'https://hooks.slack.com/services/XXX/YOUR_APP/WEBHOOK_ID';
var BUDGET_THRESHOLD = 0.90; // aka 90% of budget 
var TIME_RANGE = 'TODAY'; // Defaults to same-day spend but can be changed to other AdWords constants (i.e. 'YESTERDAY', 'LAST_7_DAYS', etc.)
```

 This code should be straightforward. We define the URL address that will process our Slack messages while also defining what budget spend threshold will serve as the point in which we'll send messages. 

> This script assumes that the budgets set in place for each campaign are indicative of desired spend levels. Feel free to tweak as your needs call for.

### Creating our Slack Messaging Function 

Now that we have a Slack Webhook to use, lets test it before creating the AdWords functionality. Add the following code block below the constants that we defined, but make sure that it is above the **monitorCampaigns()** function.

```javascript
/* This code is courtesy of the AdWords Scripts Slack example 
 * found here: https://developers.google.com/adwords/scripts/docs/features/third-party-apis
 */

/**
 * sendSlackMessage()
 * Processes text messages to Slack Webhook URLs 
 * @param {String} text
 * @param {String} [opt_channel]
 */

function sendSlackMessage(text, opt_channel) {
  var slackMessage = {
    text: text, 
    icon_url: 'http://www.gstatic.com/images/icons/material/product/1x/adwords_64dp.png',
    username: 'Campaign Alerts',
	as_user: true,
    channel: opt_channel || '#general'
  };
  
  var options = {
    method: 'POST',
    contentType: 'application/json',
    payload: JSON.stringify(slackMessage),
    muteHttpExceptions: true, // prevent HTTP exceptions from throwing fatal errors
  };
  
  UrlFetchApp.fetch(SLACK_URL, options);
}
```

> **Note**: Being able to to set the Slack icon image is still an open question. If you know how to do this, please ping me!

The code above creates a two JavaScript object variables. The first object defines the **payload** data that we'll send to the Slack API. The second JS object defines the various **HTTP** options that we'll pass to the AdWords **URLFetchApp** class so that we can send our request. 

Let's test this code! Amend **monitorCampaigns(account)** to the following: 

```javascript
function monitorCampaigns(account) {
  sendSlackMessage('This is a test message', '@yourChannel');
}
```

You should see the equivalent of the screenshot below in your respective Slack channel. If so, congratulations! You've accomplished the most difficult aspect of the script. 

>  Before moving on, Comment the **sendSlackMessage()** line from **monitorCampaigns(account)**.

![Screen Shot 2017-06-05 at 9.02.25 PM](/Users/derekmartin/Desktop/Screen Shot 2017-06-05 at 9.02.25 PM.png) 

### Checking Active Campaign Budgets 

Now that we know Slack works, let's write a function that iterates over active campaigns and checks current spend levels against our **COST_THRESHOLD**. 

If all goes to plan, this **auditCampaigns(costThreshold)** function will return an array (i.e. list) of campaigns that need our attention.  Let's write the function as follows:

```javascript
function auditCampaigns(costThreshold) {
  var threshold = costThreshold || COST_THRESHOLD;
  var campaigns = [];
  
  var campIter = AdWordsApp.campaigns().withCondition("Status = ENABLED").get();
  
  while (campIter.hasNext()) {
   var campaign = campIter.next();
   var metrics = {
     budget: campaign.getBudget().getAmount(),
     threshold: campaign.getBudget().getAmount() * threshold, 
     cost: campaign.getStatsFor(TIME_RANGE).getCost()
    };
    
    metrics.cost >= metrics.threshold 
    ? campaigns.push({
      name: campaign.getName(),
      id: campaign.getId(),
      spend: metrics.cost,
      budget: metrics.budget
    }) : undefined;    
  }
  return campaigns;
}
```

This code is fairly straightforward and does the following:

1. Declares empty variables for cost threshold and flagged campaigns.
2. Iterates through all enabled AdWords campaigns and stores its respective campaign budget, budget threshold amount, and current spend.
3. Performs a logical test on whether the current spend **exceeds** the calculated budget threshold. If this threshold has been exceeded, campaign details are appended to the **campaigns** array. If not then no action is taken.
4. Returns a list of flagged campaigns, which will be used for further processing — in our case, sending a Slack alert. 

### Alerting The User 

At this point, we are almost home free. We've defined our budget thresholds, ensured that Slack messages can be sent, and have now written the code that will identify campaigns that have exceeded their budget threshold. 

All we need to do at this point is iterate through this list of campaigns and alert the user. Amend **monitorCampaigns(account)** to the code listed below.

```javascript
/*
 * monitorCampaigns(account)
 * Obtain all active campaigns that have exceeded budget threshold.
 * Send Slack alerts to specified webhook of the campaigns in question.
 * @param {account} AdWords Account that will be processed
 */

function monitorCampaigns(account) {
  var campaigns = auditCampaigns();
  
  if (campaigns.length > 0) {
    sendSlackMessage('Budget Alert (' + moment().format("MM/DD/YYYY h:mm:ss a") + ')');
    sendSlackMessage('The following campaigns have exceeded cost thresholds:');
	// iterate through campaigns
    _.each(campaigns, function(campaign) {
      sendSlackMessage(campaign.name + ' - ' + 'Spend: $' + campaign.spend + ' - (Budget: $' + campaign.budget + ')'); 
    });
    sendSlackMessage(' '); // add whitespace to keep message thread clean
  }  
}
```

In the code above, we called on **auditCampaigns()** to identify all campaigns worth mentioning. From there, we check if there are any campaigns worth alerting.

> **Note:** we could have passed an optional **costThreshold** to **auditCampaigns** if necessary.

If there are none (i.e. **campaigns.length** is less than 0), then the script will move on to the next AdWords Account in question. Otherwise, the script sends a header message via Slack and then uses underscore's **_.each()** function to iterate through each campaign and alert the end user. 

![final](/Users/derekmartin/Desktop/final.png)

**You did it!** You now have a MCC-level Google AdWords script that will keep tabs on current campaign spend levels. Be sure to set your new script's **schedule** — daily is my suggested interval — and happy optimizing!

*Derek Martin is an Account Manager at 3Q Digital — a fully integrated digital marketing agency based in Silicon Valley.* 