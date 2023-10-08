# How to use the stripe payment API
For you to effectively use the stripe payment api I developed, you first need to know some few things. Let's first start with the architecture to understand how things flow in our program. We can then proceed to look at the endpoints, middlewares and how to use the provided endpoint.

## Stripe Payment Architecture
```mermaid
flowchart TB
    client([Client])
    stripe_api(Stripe API)
    stripe_webhook(Stripe Webhook)
    stripe_ui(Payment UI)
    todo("Create Invoice in DB<br/>Update Invoice in DB")

    %% API call to /api/stripe express endpoint
    client-- /api/stripe -->stripe_api

    %% Starting Stripe event session
    stripe_api-- Trigger Stripe Payment Event -->stripe_webhook

    %% Redirection to Stripe UI for payment processing
    stripe_webhook--ostripe_ui

    %% Create an invoice in database on session completion
    %% Update the invoice on payment success
    stripe_webhook--actions--otodo

    %% Return back to client callback function
    stripe_webhook-- Return to Callback -->client
```
Let's now look at each component in the diagram above and provided an implementation for each. At the end we will rap everything together to make it functional

## Stripe API
To use the stripe API, one must first install the stripe node module as follows. We will also install dotenv for our environment variables
```bash
npm install stripe dotenv nodemon express cors
```
Remember to edit the package.json scripts like below
```json
{
    // Other stuffs above
    "scripts": {
        "start": "nodemon index.js"
    }
    // Other stuffs below
}
```

Store the stripe publishable and secret keys in a .env file
```env
STRIPE_API_KEY=something
STRIPE_SECRET_KEY=something
```
```javascript
// file path: ./src/stripe/StripeRouter
const express = require("express");
const StripeService = require("./StripeService");

const router = express.Router();

router.post("/api/stripe", 
    check("email")
        .notEmpty()
        .withMessage("Email cannot be empty")
        .bail()
        .isEmail()
        .withMessage("Email is invalid"),
    async(req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.send({message: "Input error messages"}).status(401)
        }

        try {
            const {email} = req.body;
            const stripeSession = await StripeService.create(email);
            return res.send({ url: stripeSession.url });
        } catch(error) {
            console.log("[STRIPE_ERROR]", error);
        }

    }
)
```

```javascript
// file path: ./src/stripe/StripeService
const { stripe } = require('./src/config/stripe');

const create = async (email) => {
    const stripeSession = await stripe.checkout.sessions.create({
        success_url: "/johnson",
        cancel_url: "/johnson",
        payment_method_types: ["card"],
        mode: "",
        billing_address_collection: "auto",
        customer_email: email,
        line_items: [
            {
                price_data: {
                    currency: "USD",
                    product_data: {
                        name: "Johnson Donation Platform",
                        description: "Donate to support children",
                    },
                    unit_amount: 500,
                    recurring: {},
                },
                quantity: undefined,
            }
        ],
        metadata: {
            email,
        }
    })

    return stripeSession
}

module.exports = {
    create
}
```

## Stripe Webhook
Webhooks are a way for Stripe to notify your server of events that happen in your account, such as successful payments or failed charges. In this our application, we are using webhook to trigger creation of user invoices in our database.
```javascript
const express = require("express");
const db = require("./src/config/db");
const { stripe } = require("./src/config/stripe");

const router = express.Router();

router.post("/api/webhook", async (req, res) => {
    const signature = req.header['stripe-signature'];

    let event;

    try {
        // UI generated here
        event = stripe.webhooks.constructEvent(
            req.body,
            signature,
            process.env.STRIPE_WEBHOOK_SECRET!
        );
    } catch (error) {
        return res.status(400).send({error: `Webhook Error: ${error.message}`});
    }

    const session = event.data.object;

    if (event.type === "checkout.session.completed") {
        const subscription = await stripe.subscriptions.retrieve(
            session.subscription as string
        );

        if (!session?.metadata?.email) {
            return res.send(msg: "User email is required").status(400);
        }

        await db.userSubscription.create({
            data: {
                userEmail: session?.metadata?.email,
                stripeSubscriptionId: subscription.id,
                stripeCustomerId: subscription.customer as string,
                stripePriceId: subscription.items.data[0].price.id,
            }
        });
    }

    if (event.type === "invoice.payment_succeeded") {
        const subscription = await stripe.subscriptions.retrieve(
            session.subscription as string
        );

        await db.userSubscription.update({
            where: {
                stripeSubscriptionId: subscription.id,
            },
            data: {
                stripePriceId: subscription.items.data[0].price.id,
            }
        })
    }

    res.send({msg: "Payment successful"}).status(200);
})
```

## Putting everything together
Let's now combine everything together into our app.js, where our endpoints will leave.

```javascript
const express = require("express");
const StripeRouter = require("./stripe/StripeRouter");
const StripeWebHook = require("./middleware/StripeWebHook");
const cors = require("cors");

const app = express();
app.use(express.json());
app.use(cors());

// Middleware function
app.use(StripeWebHook);

// Stripe
app.use(StripeRouter);

module.exports = app;
```

We can then start our express application in a index.js file. We also need to initialize our prisma client so we can use the prisma ORM with our MySQL database.

```javascript
const app = require("./src/app.js");

app.listen(3000, () => console.log("App is running"));
```

```javascript
// file path: "./src/config/db";
const { PrismaClient } = require("prisma/client");

export const db = new PrismaClient();
```

## Making the client side request with axios in html file
We need to import the axios cdn script in our html file like so and write the function to make the api call to our created server
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/axios/1.2.1/axios.min.js"></script>
<script>
function handleRequest() {
    axios.post("url", {name: "data"}).then(function (response) {
        console.log(response)
        // do whatever you want if console is [object object] then stringify the response
    })
}
// WILL UPDATE THIS LATER
<script>
```