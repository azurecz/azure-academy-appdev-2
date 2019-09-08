# Human interaction in Durable Functions - Phone verification sample

This sample demonstrates how to build a Durable Functions orchestration that involves human interaction. Whenever a real person is involved in an automated process, the process must be able to send notifications to the person and receive responses asynchronously. It must also allow for the possibility that the person is unavailable. (This last part is where timeouts become important.)

This sample implements an SMS-based phone verification system. These types of flows are often used when verifying a customer's phone number or for multi-factor authentication (MFA). This is a powerful example because the entire implementation is done using a couple small functions. No external data store, such as a database, is required.

## Scenario overview

Phone verification is used to verify that end users of your application are not spammers and that they are who they say they are. Multi-factor authentication is a common use case for protecting user accounts from hackers. The challenge with implementing your own phone verification is that it requires a **stateful interaction** with a human being. An end user is typically provided some code (for example, a 4-digit number) and must respond **in a reasonable amount of time**.

Ordinary Azure Functions are stateless (as are many other cloud endpoints on other platforms), so these types of interactions involve explicitly managing state externally in a database or some other persistent store. In addition, the interaction must be broken up into multiple functions that can be coordinated together. For example, you need at least one function for deciding on a code, persisting it somewhere, and sending it to the user's phone. Additionally, you need at least one other function to receive a response from the user and somehow map it back to the original function call in order to do the code validation. A timeout is also an important aspect to ensure security. This can get fairly complex quickly.

The complexity of this scenario is greatly reduced when you use Durable Functions. As you will see in this sample, an orchestrator function can manage the stateful interaction easily and without involving any external data stores. Because orchestrator functions are *durable*, these interactive flows are also highly reliable.

## Prerequisities
1. Add this `<PackageReference Include="Microsoft.Azure.WebJobs.Extensions.Twilio" Version="3.0.0" />` to extensions.csproj file in ../wwwroot
2. Install Extension - https://docs.microsoft.com/cs-cz/azure/azure-functions/install-update-binding-extensions-manual
3. Install dependencies - sample of package.json `"dependencies": {
                                        "uuidv4": "4.0.0",
                                        "durable-functions": "^1.1.2",
                                        "twilio": "^3.0.0",
                                        "moment": "^2.21.0",
                                        "seedrandom": "3.0.3"
                                    }`



4. Configure Twilio integration

This sample involves using the [Twilio](https://www.twilio.com/) service to send SMS messages to a mobile phone. Azure Functions already has support for Twilio via the [Twilio binding](https://docs.microsoft.com/azure/azure-functions/functions-bindings-twilio), and the sample uses that feature.

The first thing you need is a Twilio account. You can create one free at https://www.twilio.com/try-twilio. Once you have an account, add the following three **app settings** to your function app.

| App setting name | Value description |
| - | - |
| **TwilioAccountSid**  | The SID for your Twilio account |
| **TwilioAuthToken**   | The Auth token for your Twilio account |
| **TwilioPhoneNumber** | The phone number associated with your Twilio account. This is used to send SMS messages. |

## IMPORTANT ##
* when yu create a trial account you have to verify your phonenumber and buy a number from which you will get a message. BE SURE that the number provide sms message. Ideally choose one of a few numbers from United States.

## The functions

This article walks through the following functions in the sample app:

* **SmsPhoneVerification**
* **SendSmsChallenge**

The following sections explain the configuration and code that are used for C# scripting and JavaScript. The code for Visual Studio development is shown at the end of the article.

## The SMS verification orchestration (Visual Studio Code and Azure portal sample code)

The **SmsPhoneVerification** function uses the standard *function.json* for orchestrator functions.

```json
{
  "bindings": [
    {
      "name": "context",
      "type": "orchestrationTrigger",
      "direction": "in"
    }
  ],
  "disabled": false
}
```

Here is the code that implements the function:

### C# !NOT TESTED!

```c#
#r "Microsoft.Azure.WebJobs.Extensions.DurableTask"

using System.Threading;

public static async Task<bool> Run(DurableOrchestrationContext context)
{
    string phoneNumber = context.GetInput<string>();
    if (string.IsNullOrEmpty(phoneNumber))
    {
        throw new ArgumentNullException(
            nameof(phoneNumber),
            "A phone number input is required.");
    }

    int challengeCode = await context.CallActivityAsync<int>(
        "E4_SendSmsChallenge",
        phoneNumber);

    using (var timeoutCts = new CancellationTokenSource())
    {
        // The user has 90 seconds to respond with the code they received in the SMS message.
        DateTime expiration = context.CurrentUtcDateTime.AddSeconds(90);
        Task timeoutTask = context.CreateTimer(expiration, timeoutCts.Token);

        bool authorized = false;
        for (int retryCount = 0; retryCount <= 3; retryCount++)
        {
            Task<int> challengeResponseTask = 
                context.WaitForExternalEvent<int>("SmsChallengeResponse");
                
            Task winner = await Task.WhenAny(challengeResponseTask, timeoutTask);
            if (winner == challengeResponseTask)
            {
                // We got back a response! Compare it to the challenge code.
                if (challengeResponseTask.Result == challengeCode)
                {
                    authorized = true;
                    break;
                }
            }
            else
            {
                // Timeout expired
                break;
            }
        }

        if (!timeoutTask.IsCompleted)
        {
            // All pending timers must be complete or canceled before the function exits.
            timeoutCts.Cancel();
        }

        return authorized;
    }
}

```

### JavaScript (Functions 2.x only)

``` javascript
const df = require("durable-functions");
const moment = require('moment');

module.exports = df.orchestrator(function*(context) {
    const phoneNumber = context.df.getInput();
    if (!phoneNumber) {
        throw "A phone number input is required.";
    }

    const challengeCode = yield context.df.callActivity("SendSmsChallenge", phoneNumber);

    // The user has 90 seconds to respond with the code they received in the SMS message.
    const expiration = moment.utc(context.df.currentUtcDateTime).add(90, 's');
    const timeoutTask = context.df.createTimer(expiration.toDate());

    let authorized = false;
    for (let i = 0; i <= 3; i++) {
        const challengeResponseTask = context.df.waitForExternalEvent("SmsChallengeResponse");

        const winner = yield context.df.Task.any([challengeResponseTask, timeoutTask]);

        if (winner === challengeResponseTask) {
            // We got back a response! Compare it to the challenge code.
            if (challengeResponseTask.result === challengeCode) {
                authorized = true;
                break;
            }
        } else {
            // Timeout expired
            break;
        }
    }

    if (!timeoutTask.isCompleted) {
        // All pending timers must be complete or canceled before the function exits.
        timeoutTask.cancel();
    }

    return authorized;
});
```

Once started, this orchestrator function does the following:

1. Gets a phone number to which it will *send* the SMS notification.
2. Calls **SendSmsChallenge** to send an SMS message to the user and returns back the expected 4-digit challenge code.
3. Creates a durable timer that triggers 90 seconds from the current time.
4. In parallel with the timer, waits for an **SmsChallengeResponse** event from the user.

The user receives an SMS message with a four-digit code. They have 90 seconds to send that same 4-digit code back to the orchestrator function instance to complete the verification process. If they submit the wrong code, they get an additional three tries to get it right (within the same 90-second window).

> [!NOTE]
> It may not be obvious at first, but this orchestrator function is completely deterministic. This is because the `CurrentUtcDateTime` (.NET) and `currentUtcDateTime` (JavaScript) properties are used to calculate the timer expiration time, and these properties return the same value on every replay at this point in the orchestrator code. This is important to ensure that the same `winner` results from every repeated call to `Task.WhenAny` (.NET) or `context.df.Task.any` (JavaScript).

> [!WARNING]
> It's important to [cancel timers](durable-functions-timers.md) if you no longer need them to expire, as in the example above when a challenge response is accepted.

## Send the SMS message

The **SendSmsChallenge** function uses the Twilio binding to send the SMS message with the 4-digit code to the end user. The *function.json* is defined as follows:

```json
{
  "bindings": [
    {
      "name": "name",
      "type": "activityTrigger",
      "direction": "in"
    },
    {
      "type": "twilioSms",
      "name": "message",
      "from": "%TwilioPhoneNumber%",
      "accountSidSetting": "TwilioAccountSid",
      "authTokenSetting": "TwilioAuthToken",
      "direction": "out"
    }
  ],
  "disabled": false
}
```

And here is the code that generates the 4-digit challenge code and sends the SMS message:

### C# NOT TESTED!
```c#
#r "Microsoft.Azure.WebJobs.Extensions.DurableTask"
#r "Microsoft.Azure.WebJobs.Extensions.Twilio"
#r "Microsoft.Extensions.Logging"
#r "Newtonsoft.Json"
#r "Twilio"

using Twilio.Rest.Api.V2010.Account;
using Twilio.Types;

public static int Run(
    string phoneNumber,
    ILogger log,
    out CreateMessageOptions message)
{
    // Get a random number generator with a random seed (not time-based)
    var rand = new Random(Guid.NewGuid().GetHashCode());
    int challengeCode = rand.Next(10000);

    log.LogInformation($"Sending verification code {challengeCode} to {phoneNumber}.");

    message = new CreateMessageOptions(new PhoneNumber(phoneNumber));
    message.Body = $"Your verification code is {challengeCode:0000}";

    return challengeCode;
}
```

### JavaScript (Functions 2.x only)

```javascript
const seedrandom = require("seedrandom");
const uuidv4 = require("uuidv4");

// Get a random number generator with a random seed (not time-based)
const rand = seedrandom(uuidv4());

module.exports = async function (context, phoneNumber) {
    const challengeCode = Math.floor(rand() * 10000);

    context.log(`Sending verification code ${challengeCode} to ${phoneNumber}.`);

    context.bindings.message = {
        body: `Your verification code is ${challengeCode.toPrecision(4)}`,
        to: phoneNumber
    };

    return challengeCode;
};
```

This **SendSmsChallenge** function only gets called once, even if the process crashes or gets replayed. This is good because you don't want the end user getting multiple SMS messages. The `challengeCode` return value is automatically persisted, so the orchestrator function always knows what the correct code is.

## Run the sample

Using the HTTP-triggered functions included in the sample, you can start the orchestration by sending the following HTTP POST request:

```
POST http://{host}/orchestrators/E4_SmsPhoneVerification
Content-Length: 14
Content-Type: application/json

"{this is your phonenumber in friendly format `420XXXXXXXXX`}"
```

```
HTTP/1.1 202 Accepted
Content-Length: 695
Content-Type: application/json; charset=utf-8
Location: http://{host}/admin/extensions/DurableTaskExtension/instances/741c65651d4c40cea29acdd5bb47baf1?taskHub=DurableFunctionsHub&connection=Storage&code={systemKey}

{"id":"741c65651d4c40cea29acdd5bb47baf1","statusQueryGetUri":"http://{host}/admin/extensions/DurableTaskExtension/instances/741c65651d4c40cea29acdd5bb47baf1?taskHub=DurableFunctionsHub&connection=Storage&code={systemKey}","sendEventPostUri":"http://{host}/admin/extensions/DurableTaskExtension/instances/741c65651d4c40cea29acdd5bb47baf1/raiseEvent/{eventName}?taskHub=DurableFunctionsHub&connection=Storage&code={systemKey}","terminatePostUri":"http://{host}/admin/extensions/DurableTaskExtension/instances/741c65651d4c40cea29acdd5bb47baf1/terminate?reason={text}&taskHub=DurableFunctionsHub&connection=Storage&code={systemKey}"}
```

The orchestrator function receives the supplied phone number and immediately sends it an SMS message with a randomly generated 4-digit verification code &mdash; for example, *2168*. The function then waits **90** seconds for a response.

To reply with the code, you can use [`RaiseEventAsync` (.NET) or `raiseEvent` (JavaScript)](durable-functions-instance-management.md) inside another function or invoke the **sendEventUrl** HTTP POST webhook referenced in the 202 response above, replacing `{eventName}` with the name of the event, `SmsChallengeResponse`:

```
POST http://{host}/admin/extensions/DurableTaskExtension/instances/741c65651d4c40cea29acdd5bb47baf1/raiseEvent/SmsChallengeResponse?taskHub=DurableFunctionsHub&connection=Storage&code={systemKey}
Content-Length: 4
Content-Type: application/json

2168
```

If you send this before the timer expires, the orchestration completes and the `output` field is set to `true`, indicating a successful verification.

```
GET http://{host}/admin/extensions/DurableTaskExtension/instances/741c65651d4c40cea29acdd5bb47baf1?taskHub=DurableFunctionsHub&connection=Storage&code={systemKey}
```

```
HTTP/1.1 200 OK
Content-Length: 144
Content-Type: application/json; charset=utf-8

{"runtimeStatus":"Completed","input":"+420XXXXXXXXX","output":true,"createdTime":"2017-06-29T19:10:49Z","lastUpdatedTime":"2017-06-29T19:12:23Z"}
```

If you let the timer expire, or if you enter the wrong code four times, you can query for the status and see a `false` orchestration function output, indicating that phone verification failed.

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 145

{"runtimeStatus":"Completed","input":"+420XXXXXXXXX","output":false,"createdTime":"2017-06-29T19:20:49Z","lastUpdatedTime":"2017-06-29T19:22:23Z"}
```

If you got a successful verification then CONGRATS!!!