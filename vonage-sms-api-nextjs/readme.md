# Using Vonage SMS API with Next.js: Sending, Receiving, and Handling Delivery Receipts

## Introduction

The [Vonage SMS API](https://developer.vonage.com/en/api/sms) enables you to send and receive text messages to and from users worldwide, using our REST APIs. In this tutorial, we will focus on sending and receiving SMS messages and handling delivery receipts with Next.js using the SMS API. [Next.js](https://nextjs.org/) is a React framework for creating full-stack web apps. It uses React Components for UI development and incorporates extra features and optimizations via Next.js. If you are interested in learning more, you can access the [Vonage SMS API documentation](https://developer.vonage.com/en/messaging/sms/overview).

### Overview

1. How to send an SMS with Next.js
2. How to receive an SMS delivery receipts and SMS with Next.js
3. How to add authorization to the webhooks

## Prerequisites

This tutorial assumes you have a basic understanding of Next.js, React and JavaScript.

Before you begin, make sure you have the following:

- [Vonage API account](https://dashboard.nexmo.com/) - Access your Vonage API Dashboard to locate your API Key and API Secret, which can be found at the top of the page.
- [Vonage Virtual Number](https://dashboard.nexmo.com/buy-numbers) - Rent a virtual phone number to send or receive messages and phone calls.
- [Node.js](https://nodejs.org/) 18.17 or later - Node.js is an open-source, cross-platform JavaScript runtime environment.
- Deployment Platform such as [Vercel](https://vercel.app/), [Netlify](https://www.netlify.com/), etc. - We'll set up webhooks for the URLs that receive inbound SMS and delivery receipts. This way, we'll be able to receive SMS and handle delivery receipts.

## 1. How to send an SMS with Next.js

For the initial section of this tutorial, we'll explore how to send text messages using Next.js. We will use the Vonage SMS API to interact with Next.js. To send an SMS with Next.js and the Vonage SMS API, proceed with these instructions:

1. Create a new Next.js project
2. Declare environment variables
3. Install the [Vonage Node.js SDK](https://github.com/vonage/vonage-node-sdk)
4. Install the validation library [Zod](https://github.com/colinhacks/zod)
5. Create a server-only form

### 1. Create a new Next.js project

To create a Next.js app, run:

```bash=
npx create-next-app@latest
```

On installation, you'll see the following prompts:

```bash=
What is your project named? vonage-send-sms
Would you like to use TypeScript? No
Would you like to use ESLint? Yes
Would you like to use Tailwind CSS? Yes
Would you like to use `src/` directory? No
Would you like to use App Router? (recommended) Yes
Would you like to customize the default import alias (@/*)? No
```

In this tutorial, we'll use the App Router and JavaScript, so you can choose the same answers as above. After the prompts, `create-next-app` will create a folder with your project name and install the required dependencies.

### 2. Declare environment variables

Retrieve your API key and API secret from your [API settings](https://dashboard.nexmo.com/settings), also get your virtual number from the [your numbers page](https://dashboard.nexmo.com/your-numbers), and create a `.env.local` file with the following environment variables:

```bash=
VONAGE_API_KEY=your-vonage-api-key
VONAGE_API_SECRET=your-api-secret
VONAGE_VIRTUAL_NUMBER=your-virtual-number
```

#### To rent a Vonage virtual number:

1. Sign in to the [developer dashboard](https://dashboard.nexmo.com/).
2. In the left-hand navigation menu, click **Numbers** then **Buy numbers**.
3. Choose the attributes you need and then click **Search**.
4. Click the **Buy** button next to the number you want and validate your purchase.
5. Your virtual number is now listed in **Your numbers**.

See [rent a virtual number](https://developer.vonage.com/en/numbers/guides/number-management#rent-a-virtual-number).

### 3. Install the Vonage Node.js SDK

We'll use the Vonage Node.js SDK. It's so easy to send an SMS with Next.js.

To install the Node.js SDK, run:

```bash=
npm install @vonage/server-sdk
```

### 4. Install the validation library Zod

Zod is a TypeScript-first schema declaration and validation library. We will use it to check the values after the form is submitted. But it is not mandatory, **you may skip this step** if you want.

To install the Zod, run:

```bash=
npm install zod
```

### 5. Create a server-only form

Create a new folder called `lib` inside /app. Then, create a new `send-sms.js` file inside the `lib` folder with the following content:

```javascript
"use server";

import { revalidatePath } from "next/cache";
import { Vonage } from "@vonage/server-sdk";
```

After importing the dependencies, initialize the Vonage node library installed earlier:

```javascript=
const vonage = new Vonage({
  apiKey: process.env.VONAGE_API_KEY,
  apiSecret: process.env.VONAGE_API_SECRET,
});

const from = process.env.VONAGE_VIRTUAL_NUMBER;
```

Then initialize the Vonage node library, we will create an async function called sendSMS:

```javascript=
export async function sendSMS(prevState, formData) {
  try {
    const vonage_response = await vonage.sms.send({
      to: formData.get("number"),
      from,
      text: formData.get("text"),
    });

    revalidatePath("/");
    return {
      response:
        vonage_response.messages[0].status === "0"
          ? `🎉 Message sent successfully.`
          : `There was an error sending the SMS. ${
              // prettier-ignore
              vonage_response.messages[0].error-text
            }`,
    };
  } catch (e) {
    return {
      response: `There was an error sending the SMS. The error message: ${e.message}`,
    };
  }
}
```

To send an SMS message using the SMS API, we'll use the `vonage.messages.send` method of the Vonage Node.js library. This method accepts objects as parameters that contain information about the recipient, sender, and content. The SMS API has two types of response (`vonage_response`), one is message sent and the other is error. See [SMS responses](https://developer.vonage.com/en/api/sms#send-an-sms-responses).

Example response for the Message Sent:

```json=
{
   "message-count": "1",
   "messages": [
      {
         "to": "447700900000",
         "message-id": "aaaaaaaa-bbbb-cccc-dddd-0123456789ab",
         "status": "0",
         "remaining-balance": "3.14159265",
         "message-price": "0.03330000",
         "network": "12345",
         "client-ref": "my-personal-reference",
         "account-ref": "customer1234"
      }
   ]
}
```

Example response for the Error:

```json=
{
   "message-count": "1",
   "messages": [
      {
         "status": "2",
         "error-text": "Missing to param"
      }
   ]
}
```

We'll invalidate an entire route segment with `revalidatePath`, that allows you to purge cached data on-demand for a specific path. It does not return any value. See [revalidatePath](https://nextjs.org/docs/app/api-reference/functions/revalidatePath).

For more advanced server-side validation, use the Zod library. If you use it, the `send-sms.js` file should look like this:

```javascript=
"use server";

import { revalidatePath } from "next/cache";
import { Vonage } from "@vonage/server-sdk";
import { z } from "zod";

const vonage = new Vonage({
  apiKey: process.env.VONAGE_API_KEY,
  apiSecret: process.env.VONAGE_API_SECRET,
});

const from = process.env.VONAGE_VIRTUAL_NUMBER;

const schema = z.object({
  number: z
    .string()
    .regex(new RegExp(/^\d{10,}$|^(\d{1,4}-)?\d{10,}$/), "Invalid Number!"),
  text: z.string().min(1, "Type something, please!").max(140, "Too long text!"),
});

export async function sendSMS(prevState, formData) {
  try {
    const data = schema.parse({
      number: formData.get("number"),
      text: formData.get("text"),
    });

    const vonage_response = await vonage.sms.send({
      to: data.number,
      from,
      text: data.text,
    });

    revalidatePath("/");
    return {
      response:
        vonage_response.messages[0].status === "0"
          ? `🎉 Message sent successfully.`
          : `There was an error sending the SMS. ${
              // prettier-ignore
              vonage_response.messages[0].error-text
            }`,
    };
  } catch (e) {
    return {
      response: `There was an error sending the SMS. The error message: ${e.message}`,
    };
  }
}
```

For UI, create a new `send-form.jsx` file inside /app with the following content:

```jsx=
"use client";

import { sendSMS } from "@/app/lib/send-sms";
import { useFormStatus, useFormState } from "react-dom";

const initialState = {
  response: null,
};

function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button
      type="submit"
      aria-disabled={pending}
      className="border rounded-md hover:bg-slate-50 p-2 flex justify-center items-center"
    >
      {pending ? (
        <>
          <div class="border-gray-300 h-5 w-5 animate-spin rounded-full border-2 border-t-blue-600 mr-2" />
          Sending...
        </>
      ) : (
        "Send"
      )}
    </button>
  );
}

export function SendForm() {
  const [state, formAction] = useFormState(sendSMS, initialState);

  return (
    <form action={formAction} className="flex flex-col gap-y-2">
      <label htmlFor="number">Phone number:</label>
      <input
        name="number"
        id="number"
        type="number"
        placeholder="909009009099"
        autoComplete="off"
        className="border rounded p-2"
        required
      />
      <label htmlFor="text">Message:</label>
      <textarea
        name="text"
        id="text"
        rows={4}
        cols={40}
        placeholder="Hello from Next.js App!"
        className="border rounded p-2"
        required
      />
      <SubmitButton />
      <p aria-live="polite">{state?.response}</p>
    </form>
  );
}
```

You can use the `useFormStatus` hook to display a loading status when a form is being submitted to the server. Another hook, the `useFormState`, allows you to update state based on a form action result.

Finally, you can import the `send-form.jsx` file to the `page.js` inside /app.

```jsx=
import { SendForm } from "./send-form";

export default function Home() {
  return (
    <SendForm />
  );
}
```

Right now, your home page should look like this:
![image.png](https://hackmd.io/_uploads/HJ1tGMrQ6.png)

:::info
You cannot send an SMS to Turkish numbers unless...
:::

It's that easy!

## 2. How to receive an SMS delivery receipts and SMS with Next.js

When you successfully request the SMS API, it sends you an array of message objects, with each message having a status of `0` to indicate success. However, this does not guarantee that your recipients have received the message. To receive delivery receipts in our Next.js app, we must provide **a webhook endpoint** for Vonage to send them to.

1. Create a webhook endpoint
2. Create a handler for POST requests
3. Add a response to the handler
4. Configure the API Settings
5. Receive an SMS and handle delivery receipts

### 1. Create a webhook endpoint

Create a new folder called `webhook` inside `/app`. Then, create a new folder one more called `status` inside the `webhook` folder. After that, create a new `route.js` file inside the `status` folder. The folder structure should look like this:

```bash=
.
└── vonage-send-sms
    └── app
        ├── webhook
        │   └── status
        │       └── route.js
        ├── page.js
        └── layout.js
```

Since we will do exactly the same to capture incoming SMS, we will create another folder called `inbound` with the `route.js` file under the `webhook` folder:

```bash=
.
└── vonage-send-sms
    └── app
        ├── webhook
        │   ├── status
        │   │   └── route.js
        │   └── inbound
        │       └── route.js
        ├── page.js
        └── layout.js
```

### 2. Create a handler for POST requests

We will create a handler for POST requests at `/webhook/status` to handle delivery receipts and at `/webhook/inbound` to handle incoming SMS messages. Then, we'll log the body of the request to the console:

```javascript=
export async function POST(request) {
    const res = await request.json();
    console.log("The request from Vonage: ", JSON.stringify(res, null, 2));
}
```

Right now, you can see the message status on the logs and also add the message status to your database or something else.

### 3. Add a response to the handler

Vonage will assume that you have not received the message and will keep resending it for the next 24 hours. This way, the webhook endpoint must send a `200 OK` or `204 No Content` response:

```javascript=
export async function POST(request) {
    const res = await request.json();
    console.log("The request from Vonage: ", JSON.stringify(res, null, 2));

    return new Response("ok", {
      status: 200,
    });
}
```

### 4. Configure the API Settings

Your webhook endpoint is now deployable on either Vercel, Netlify or your own server. After deploying your app, fill out each field by appending `/webhook/inbound` and `/webhook/status` to the Inbound SMS URL and Delivery Receipts URL in the [API settings](https://dashboard.nexmo.com/settings).

![image.png](https://hackmd.io/_uploads/B1DhRFrm6.png)

### 5. Receive an SMS and handle delivery receipts

Right now, you should be able to see the requests from Vonage on your console logs. To see delivery receipts, send an SMS message through your Next.js application. The response from Vonage will look something like this:

```json=
{
  "msisdn": "905423247231",
  "to": "4759580387",
  "network-code": "28603",
  "messageId": "4a3cf988-570f-4cdb-95be-179f89c64498",
  "price": "0.02380000",
  "status": "failed",
  "scts": "2311031929",
  "err-code": "1",
  "api-key": "7725f733",
  "message-timestamp": "2023-11-03 19:29:32"
}
```

See [Understanding the delivery receipt](https://developer.vonage.com/en/messaging/sms/guides/delivery-receipts#understanding-the-delivery-receipt).

To see how incoming SMS messages appear in the console log, send an SMS message from your phone to your Vonage number:

```json=
{
  "msisdn": "905423247231",
  "to": "4759512387",
  "messageId": "2B00000018726238",
  "text": "Hi! This is test message from Emre!",
  "type": "text",
  "keyword": "HI!",
  "api-key": "7725f733",
  "message-timestamp": "2023-11-03 19:55:15"
}
```

See [Anatomy of an Inbound Message](https://developer.vonage.com/en/messaging/sms/guides/inbound-sms#anatomy-of-an-inbound-message).

## 3. How to add authorization to the webhooks

You can authenticate using the Vonage APIs. This enables the webhook to prevent unidentified traffic from reaching the endpoint. In this article, we will use the HTTP basic authentication. The HTTP basic authentication is a straightforward method for a server to ask a client for login credentials, including a username and password. The Vonage API sends login details to the server using an Authorization header.

1. Configure the API Settings
2. Install the basic-auth library
3. Import the dependencies to the webhook
4. Check the incoming credentials

### 1. Configure the API Settings

First, we need to configure the API settings as follows:
![image.png](https://hackmd.io/_uploads/ryoPR5r7p.png)
To enable basic HTTP authentication, prepend `username:password@` to the hostname in your webhook URL. We will send the credentials in the HTTP header. See [Enabling HTTP Basic Authentication](https://api.support.vonage.com/hc/en-us/articles/230076127-Enabling-HTTP-Basic-Authentication-for-Webhook-URL)

### 2. Install the basic-auth library

To parse the authorization header field, we will use the basic-auth library. To add this in your project, run:

```bash=
npm install basic-auth
```

### 3. Import the dependencies to the webhook

We will add the `headers` function from `next/headers` together with the `basic-auth` library. Thus, we're able to read the HTTP incoming request headers.

```javascript=
import auth from "basic-auth";
import { headers } from "next/headers";

export async function POST(request) {
  const headersList = headers();
  const authorization = headersList.get("Authorization");
  const { name, pass } = auth.parse(authorization);
  console.log("Credentials: ", name, pass);
}
```

See [headers](https://nextjs.org/docs/app/api-reference/functions/headers).

### 4. Check the incoming credentials

Add your user information as environment variables to the `.env.local` file:

```bash=
BASIC_AUTH_NAME=your-webhook-name
BASIC_AUTH_PASS=your-webhook-pass
```

Then check that the incoming credentials is correct as following:

```javascript=
import auth from "basic-auth";
import { headers } from "next/headers";

export async function POST(request) {
  const headersList = headers();
  const authorization = headersList.get("Authorization");
  const { name, pass } = auth.parse(authorization);
  console.log("Credentials: ", name, pass);

  if (
    name === process.env.BASIC_AUTH_NAME &&
    pass === process.env.BASIC_AUTH_PASS
  ) {
    const res = await request.json();
    console.log("SMS Status: ", JSON.stringify(res, null, 2));

    // Then add the message status to your database or something else,
    // Also you can see the message status on the logs.

    return new Response("ok", {
      status: 200,
    });
  } else {
    return new Response("No Content", {
      status: 201,
    });
  }
}
```

## Conclusion

Now that you know how to send and receive SMS messages and get delivery receipts with the Vonage SMS API and Next.js. Consider expanding this project by replying to incoming SMS messages or adding more complex, interactive elements. Also you can access **the project's source code** on Github for forking: https://github.com/emrecoban/vonage-send-sms

As always, feel free to reach out on the [Vonage Developer Slack](https://developer.vonage.com/en/community/slack) or [Twitter](https://twitter.com/emreshepherd) for any questions or feedback. Thank you for reading, and I look forward to connecting with you next time.

## Further Reading

- [Forms and Mutations](https://nextjs.org/docs/app/building-your-application/data-fetching/forms-and-mutations)
- [Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- [The useFormState hook](https://react.dev/reference/react-dom/hooks/useFormState)
- [The useFormStatus hook](https://react.dev/reference/react-dom/hooks/useFormStatus)

###### tags: `javascript` `next.js` `sms-api`
