# Refunds, disputes and taxes

What happens when money goes back to the student, which tax details you need in order to get paid, and what falls to you as the seller. For the normal flow of money, see [Payouts](payouts.md).

## Refunds

You decide whether to give money back. You do it from your panel and Didacta returns its proportional share; what Stripe does not return is on you.

### How to refund

1. In **Sales**, open the sale and click **Refund**.
2. Choose **the full amount** or **part of it** and enter how much.
3. Before you confirm, the academy shows what the student receives and how much comes off your money.

Nobody at Didacta refunds a sale of yours on their own, unless the law, an authority or Stripe requires it, or to stop obvious fraud; in that case it tells you and the same rules apply.

### What happens to the money

- The money goes back to the student **the same way they paid**; they usually see it within 5 to 10 days.
- If your share had already been transferred, the proportional part is **taken back from your Stripe balance**. If there is not enough balance, the difference becomes a [debt](#what-you-owe). If it had not been sent yet, it is subtracted from what you were going to receive.
- Didacta **returns its 5%** in the same proportion as you refund.
- **Stripe does not return its fee** on a refund. You bear that fee and it is deducted from your next payouts.
- With a **full** refund, the student automatically loses access to what they bought. With a **partial** one, they keep it unless you remove it.

**Example.** You fully refund a €100 sale paid with a Spanish card. The student gets €100 back. The €93.25 that was transferred to you is reversed, Didacta returns its €5.00, and Stripe's €1.75 fee, which cannot be recovered, is deducted from your next payouts. If you refund €40, €37.30 is reversed and Didacta returns €2.00.

## Disputes

A dispute (or chargeback) is when a student tells their bank they do not recognise a payment or did not receive what they bought, and the bank takes the money back while it reviews. **Disputes are always on your account, whether you win or lose.**

1. Didacta notifies you **by email and in your panel** as soon as Stripe notifies it, with the deadline to respond. While it is decided, that sale is held.
2. If you have evidence that the student received what they bought (access logs, messages, an invoice), **send it by email**: Didacta presents it to Stripe on your behalf.
3. When the bank takes the money back, what was transferred to you for that sale is reversed from your payout account.

| Outcome | What you bear | Didacta's 5% |
| --- | --- | --- |
| **You lose** | The sale amount, Stripe's dispute fee (currently €20 or the equivalent) and Stripe's fee on that sale. | Didacta gives it back to you: the money goes back to the student. |
| **You win** | The dispute fee that Stripe does not return. What was reversed is returned to you. | Not returned: the student does not get their money back. |

## What you owe

If, after a refund or a dispute, money is owed between you and Didacta:

- **What you owe is deducted from your next payouts**, in the same currency. You see each debt in your panel with its reason and outstanding amount.
- If after **30 days** there are not enough payouts to cover it, or you have stopped using Didacta Cloud, Didacta asks you to pay by email with the details and you have **another 30 days** to pay by bank transfer. If you do not pay, it can claim it through legal channels.
- If Didacta owes you something (for example, a dispute you won after the contract ended), it pays it to your payout account or, if that no longer exists, by transfer to the account you indicate.

## Tax details

For payments to reach you, the academy asks for your **tax details** (in the **Payouts** section, which in simple mode is under **Sales → See details**, and as the last part of "Connect your payouts"). The law requires platforms to collect them, and they are used to issue the fee invoice.

| Detail | Notes |
| --- | --- |
| **How do you sell?** | Individual or sole trader, or company. |
| **Name or company name** | |
| **Country of tax residence** | |
| **Tax ID** | Your ID or NIE with the letter, or your company's tax ID; outside Spain, your tax identification number. The academy checks the format and the check digit. |
| **EU VAT number** (optional) | If you have one and you are in the EU, the invoice has no VAT and you account for it yourself. It is checked on VIES. |
| **Tax address** | |
| **Date of birth** | Individuals only: required by the rules on information exchange for digital platforms (DAC7). |

!!! warning "Without tax details, the money is held"
    Until you complete them, your sales are **still collected**, but Didacta **does not transfer** the money to your payout account. It is kept safe and reaches you as soon as you complete them. You can publish and sell without them, but the money does not reach you.

The details are stored **encrypted** and used only for your invoices and for what tax law requires. As a platform operator, Didacta may have to report to the Spanish Tax Agency each year your sales and the fees charged (form 238). If you do not provide the details after two reminders, the law requires it to stop paying you or to close your account.

### Fee invoice

Every month Didacta issues you an invoice for its fee of the previous month, with VAT itemised according to your case (in the **Payouts** section, **Your invoices** card, PDF download). You will see the regime applied: VAT included, Spanish VAT included, no VAT under reverse charge, no VAT because you are outside the EU, or no VAT for the Canary Islands, Ceuta or Melilla. If an issued invoice is corrected, a **corrective invoice** appears. More in [Pricing and fee](pricing.md#taxes-on-the-fee).

## You are the seller

The purchase contract is **between you and your student**: Didacta provides the tool and collects on your behalf. So:

- You decide the price, what each course or membership includes, your terms of sale and your refund policy, and you publish them in your academy.
- You answer to your students for the content and for it being what you advertised.
- You give your students the information the law requires before they buy: who you are, what they are buying, the total price with taxes, how long a subscription lasts and how it is cancelled, and their right of withdrawal.
- If you sell to EU consumers, they have **14 days to withdraw**. For digital content that right is lost if the student asks to start before the 14 days end, expressly consents and acknowledges that they lose it; Didacta's payment page collects that consent when the student buys digital content.
- **You issue invoices to your students** when the law requires it: the Stripe receipt is not an invoice.
- **You declare and pay the taxes on your sales** (VAT, OSS…).

!!! warning "Taxes: ask your adviser"
    This is information, not tax advice. The terms of use for creators also set out a warning about VAT on electronic services sold through platforms: Didacta is designed so that you are the supplier, but it cannot guarantee how each tax authority will classify these sales. Read it in the [terms](https://didacta.io/en/legal/terms) (section 6).
