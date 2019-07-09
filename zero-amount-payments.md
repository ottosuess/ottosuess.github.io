# Exploiting zero amount payments on the lightning network
Zero amount payments are widely used on the lightning network. One big use case is to support donation addresses where users can send arbitrary amounts to lightning invoices. However they are not trustless and can be exploited by intermediate nodes on the payment route.

### Introduction to routing
I will start by introducing the relevant parts of the lightning network needed to understand the exploit. If you already know how routing works you can skip to the next section.

Payments in the lightning network are routed from the sender to the receiver with an onion routed package. Intermediate nodes on the route neither know the total length of the route nor the final receiver of the payment. They only know who their immediate predecessor is and have the relevant information to forward the payment to their successor on the route (i.e. the channel id of the successor and the amount to forward).

### Exploiting overpayment
Let's say there are three nodes. Alice, Bob and Mallory. Alice has a channel to Mallory, and Mallory has a channel to Bob. Bob wants to collect a tip from Alice and creates an invoice without a specifying the  amount. After receiving that invoice Alice decides to tip Bob 1000 satoshi and creates the onion routed package that routes the payment via Mallory to Bob.

When the onion routed package arrives at Mallory she knows that Bob is the successor in the route and she might know that Bob regularly collects tips without a specific amount. She can then send a **new payment** with the **same hash** but only 1 satoshi to Bob. Bob sees an incoming 1 satoshi payment and is very happy he received a donation. He reveals the payment’s preimage to claim 1 satoshi from Mallory with the same preimage Mallory can now claim 1000 satoshi from Alice.

Bob can’t know he was cheated because he does not know how much Alice intended to send. Like everybody else on the route he can only see his **immediate** predecessor and not the amount the original payer sent.

This exploit does not always succeed for Mallory. She does neither knows that Bob is the final destination of the payment nor that Bob didn’t specify an exact amount for that payment. But if Bob regularly gives out 0 amount invoices it might be worth a try. Especially because Bob can’t punish Mallory for that behavior.

### Why does lightning allow overpayments
[BOLT 4](https://github.com/lightningnetwork/lightning-rfc/blob/master/04-onion-routing.md#requirements-2) specifies that the payee should accept payments up to twice the amount expected. Why does the lightning specification allow overpaying invoices at all if they can be exploited like that?

Overpaying does aid privacy. Let's say you have a channel to a popular service selling articles for 150 satoshi. Every time you are routing a 150 satoshi payment over that channel you can assume that payment was used to buy an article. To prevent this people buying articles could overpay and send a more random looking amount to the final node.

The reason for only accepting payments up to twice the amount expected is to not allow "accidental gross overpayment". For zero amount payments this check is disabled.

###  A possible solution
A more secure way to send payments without a fixed amount would be so called spontaneous payments (formerly known as sphinx payments). Spontaneous payments can be sent without first receiving invoices from the destination. The sender creates the preimage and includes it in the onion routed package in a way that only the final destination can decode it.

If an intermediate node now sends a payment with the same hash to the final destination, the final destination has no way of knowing the preimage and collecting the payment.

Spontaneous payments are not available in any of the major lightning clients at the time of writing. However there is a [pull request for lnd](https://github.com/lightningnetwork/lnd/pull/2455) implementing spontaneous payments and a [sample implementation for c-lightning](https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-June/002009.html).

### Conclusion
Optional amount invoices should be deprecated from the LN-spec because they can not be executed in a trustless way. Spontaneous payments provide a more secure alternative that could replace zero amount invoices.

If you know a reason why 0 amount invoices should exist [let me know on twitter](https://twitter.com/ottosuess) or make a PR on GitHub.
