# Payouts

On Didacta Cloud **your students pay on Didacta and Didacta transfers your share to you**. You do not paste Stripe keys or set up webhooks: you connect a payout account in about 5 minutes and the platform does the rest.

## How the money flows

1. **The student pays by card** inside your sales page, with your logo (or your initials) and your colour. Stripe processes the payment and the student gets the receipt by email. They can pay from any country.
2. **You appear as the seller**: on the payment page, on the receipt and on the student's bank statement. Didacta collects the payment on your behalf with its own Stripe account and keeps its fee (the [5%](pricing.md)) and the card fee.
3. **Didacta transfers the rest to your Stripe payout account**, normally within minutes and at most within a few hours of Stripe confirming the payment.
4. **Stripe sends the balance to your bank once a week, on Mondays**, with the money from sales that are at least about 7 days old, always leaving a minimum balance of **€50** (or the equivalent in your currency) in your payout account to cover refunds and disputes. In practice, the money usually reaches your bank **7 to 14 days** after the sale.

!!! info "Didacta is not a payment institution"
    The money moves within Stripe, an authorised institution, at all times, and your share does not go through Didacta's bank accounts. Didacta acts only on your behalf, never on the student's: once the student has paid, they have paid.

## Connecting your payouts

It is the fourth step of ["Start here"](getting-started.md#3-start-here); you can also do it from **Sales**.

1. Click **Connect payouts**.
2. Choose your **country** (the one your bank account is in) and **how you will get paid**: as an individual or sole trader, or as a company.
3. Stripe's sign-up opens. It asks for your details, an identity document and your bank account, and for you to accept its terms. Payouts for creators are provided by Stripe and subject to the [Stripe Connected Account Agreement](https://stripe.com/connect-account/legal/full).
4. You come back to the academy and see where you are:

| Message in the academy | What it means |
| --- | --- |
| *We need one more thing before we can pay you: …* | Stripe needs something more (your bank account, a document, your personal or business details, accepting its terms). Click **Finish**. |
| *Stripe is reviewing your details* | You do not need to do anything: when it finishes, you will be able to sell. |
| *Stripe hasn't approved this payouts account* | You cannot sell for now. Write to [support](support.md) and it is reviewed. |
| *All set. You're paid in … every Monday* | You can sell. **See my payouts on Stripe** opens your Stripe dashboard. |

Until Stripe gives the go-ahead, the academy does not let anyone buy your products. Stripe will ask for a **customer-service phone number**: your students will see it on their receipts; if you do not want to give your personal mobile, use a work number.

## Countries and currencies

### Where you can get paid

You can get paid if you **live and have your bank account** in one of these 33 countries: the 27 **European Union** members (Spain included), **Norway**, **Liechtenstein**, the **United Kingdom**, **Switzerland**, the **United States** and **Canada**.

Today Didacta **cannot** pay creators whose bank is in Mexico, Brazil, Argentina, Colombia, Chile, Peru or the rest of Latin America. You can still open your academy and build your community and free courses, but you cannot get paid for sales. The list depends on where Stripe allows Didacta to pay; if it changes, it is updated.

Your **students** can pay from any country.

### The currency you get paid in

- Your **payout currency** is that of your Stripe account, according to your country: euros, pounds, US or Canadian dollars, Swiss francs, krona/krone (Swedish, Danish, Norwegian), Czech koruna, złoty, forint or leu.
- You can set the price in any of those twelve currencies. If it is your account's currency, the 5% comes out exact and Stripe can show each student the price converted into their own currency (the student pays that conversion, not you).
- If you choose a currency **other than** your account's, Stripe charges a conversion cost that the creator pays, and the student no longer sees the converted price. The price screen warns you.
- You can also set **your own price for other markets** (up to 20 extra currencies).

## Your sales

In **Sales** you see:

- **The summary by currency:** sold, already in your bank, on the way and held.
- **Each sale** with who bought, what, how much they paid, the card fee, Didacta's 5%, what you receive and when it arrives. You can filter by dates, product, status or text.
- **Export** sales to CSV or Excel and buyers to CSV (with the filters you have set; it is personal data: keep it carefully).

Statuses you will see on each sale:

| Status | Meaning |
| --- | --- |
| Paid · working out fees | The student has paid; the split is being calculated. |
| Sending you the money / Sent to your account | The transfer to your payout account is on the way or done. |
| Held | The money is safe but not transferred yet: your [tax details](refunds-and-taxes.md#tax-details) are missing, or Stripe has not finished verifying your account, or it will be retried the next day. |
| Refunded / Partly refunded | See [Refunds](refunds-and-taxes.md#refunds). |
| Disputed · Dispute won / lost | The student disputed the payment with their bank. See [Disputes](refunds-and-taxes.md#disputes). |
| Used to pay what you owed | That sale was applied to an outstanding [debt](refunds-and-taxes.md#what-you-owe). |

## When Didacta may delay a transfer

Only in these cases, and it always tells you the reason unless the law forbids it:

- While you have not completed your tax details.
- While Stripe has not finished verifying your account or has restricted it.
- If there are reasonable signs of fraud or unauthorised sales.
- If your refunds or disputes are abnormally high.
- If an authority or a judge requires it.
- If you seriously breach the terms.

A hold lasts as long as strictly necessary and at most **up to 120 days** after the last affected sale, which is the period in which a student can open a dispute.

## Discount codes

Stripe discount codes **do not work** in Didacta Cloud academies for now: the payment page rejects them. What you do have is the [discount through "Invite and earn"](referrals.md) on the membership fee.

## What does not change with Cloud

The student buys just as in Community: the course sales page (`/catalog/<course>`) or the membership page (`/join`), and access is granted automatically on payment. The [payment modules](../modules/catalog/billing.md) are the same; the difference is **who collects** (the platform, not a key of yours).
