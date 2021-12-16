# A blame ascribing protocol towards ensuring time limitation of stuck HTLCs in flight.

**This is a Draft version and Work in Progress.**

I was reviewing the [HOLD fee proposal by Joost](https://github.com/lightning/bolts/pull/843) and the [excellent summary of known mitigation techniques by t-bast](https://github.com/t-bast/lightning-docs/blob/master/spam-prevention.md) when I revisited the very [first idea to mitigate HTLC spam via onions](https://lists.linuxfoundation.org/pipermail/lightning-dev/2015-August/000135.html) that was discussed back in 2015 by Rusty, AJ and a few others. At that time the idea was to ascribe blame to a malicious actor by triggering a force close and proofing ones own honesty by providing the force close transaction. I think there is a lot of merit to the idea of ascribing blame and I think it might be possible with the help of [onion messages](https://github.com/lightning/bolts/pull/759) without the necessity to trigger full force closes. 

As I am not entirely sure if this suggestion is a reasonable improvement (it certainly does not resolve all the issues we have) I did not spec out the details and message formats and fields but only described the high level idea. I hope this is sufficient to discuss the principles and get the feedback from you if you consider this to be of use and if you think we should work on the details. 

Idea / Obervation:
=============
The key idea is to set a fixed time in seconds (the `reply_interval`) after successfully negotiating an HTLC until when a node requires a resultion or reply from its peer to which it previously has forwarded a downstream onion. If the HTLC is not resolved and no reply was sent the downstream peer is considered to be acting maliciously.

The amount in seconds can be proportional to the `cltv_delta` of that hop. To me the arbitrary choice of translating 10 blocks of `cltv_delta` to `1` second of expected reply time seems reasonable for now but could be chosen differently as long as the entire network (or at least every node included to the payment attempt) agrees upon the same conversion rate from `cltv_delta` to expected response time from downstream nodes. 

There are three cases for the reply:

The Good reply case (HTLC resolution):
==============================
The good case is if the payment will succesfully settle or fail within the `reply_interval`. Thus the reply comes in the form of either an `update_fulfill_htlc` or an `update_fail_htlc`. In any case the HTLC will be removed quickly and the reply can propagate back to the upstream peers.

The bad reply case:
===============
If a node `N` is not able to send one of those two `update_` messages because the HTLC was not resolved from the downstream channels it MUST send back a new message called `blame_channel`.

The `blame_channel` includes a proof that `N` has previously successfully set up the HTLC with the next peer. This may for example be done by including the `commitment_signed` message that the node has received from the next peer as this commits to this (and potentially other) HTLCs. (Alternatively one could extract the relevant `htlc_signature` or adopt `commitment_signed` to include a signature to the `payment_hash` that can be verified with the node's pubkey)

The propagated bad reply case:
========================
A node might have received an `blame_channel` message from a downstream channel and can propagate this back. To disallow spoofing, nodes might always have to extend the message with their own signature. (I have not thought about this extensively yet). This will make the downstream path of the payment transparent to every node on the upstream path (until the slow /misbehaving node). Nodes SHOULD propagate the `blame_channel` back in a timely manner unless they want to be blamed for the delay themselves. 

Extensions:
=========
Of course a malicious node `M` might after receiving an `update_add_htlc` from a node `N` interrupt the channel state machine at various moments before the state successfully moved forward. This would prevent the honest node `N` to ascribe blame to the malicious node `M` as it cannot include proof that it forwarded the HTLC successfully. Thus instead of including the proof of successfully having set up the HTLC the node `N` ascribing blame to `M` might include to the `blame_channel` the last messages it sent to the downstream peer that have not been acknowledged to be able to proof that it tried to move the state machine forward and set up the HTLC.

A node on an upstream channel could verify that `N` was indeed honest and has also delivered these messages to `M` by sending these messages via a different path as an onion message to `M`. 

Now if `M` was indeed dishonest `M` would not respond to the onion message indicating to the upstream node that `N` was honestly ascribing blame to `M`. Otherwise `M` could try to move the state forward and proof this to the upstream node in the onion reply indicating that it was actually `N` who tried to act malicious. In any case if there was a disagreement the upstream nodes would in any case learn that there is a problem on the channel between `N` and `M`.

Security considerations:
=========
On the Mailinglist [Bastien Teinturier noted the following](https://lists.linuxfoundation.org/pipermail/lightning-dev/2021-December/003409.html):

> If we have a payment: A -> B -> C -> D and C is malicious.
C can forward the payment to D, and even wait for D to correctly settle it
(with `update_fulfill_htlc` or `update_fail_htlc`), but then withhold that
message instead of forwarding it to B. Then C blames D, everyone agrees that
D is bad node that must be avoided. Later, C unblocks the `update_*_htlc`
and everyone thinks that D hodled the HTLC for a long time, which is bad.

The above issue can be addressed by `B` verifying the proof it received from `C`. This can be done by presenting the proof to `D` via an onion message along a different node than `C`. If `D` cannot refute the proof by presenting a newer state to `B` then `B` knows that `D` was indeed dishonest. Otherwise `D` and `B` have discovered that `C` was misbehaving and tried to frame `D`.

`B` indicates to `D` that it is allowed to ask such verification question by include the received proof from `C`. Note that `B` could never own such proof if `C` has not communicated with `B`. Of course if `C` has never talked to `B` in the first place `B` would have send a `TEMPORARY_CHANNEL_FAILURE` and if `C` stopped during the update of the statemachine to communicate to `B` then `B` can blame `C` via the above mechanism and `A` can verify the claim it received from `B`. 

Also `B` cannot just send garbage to `D` and try to frame `C` because as soon as `B` would frame `C` the upstream node `A` would talk to `C` and recognize that it was `B` who was dishonest.
 
Going back to the situation assuming that `C` and `D` have indeed already successfully resolved the HTLC then the node `D` could in the reply to `B` even securely include the preimage allowing `B` to reclaim the funds from `A` and settle the HTLC in the A->B channel. Only the HTLC in the B->C channel would be locked which doesn't have to bother `B` as `B` expects that `C` is pulling / settling the HTLC anyway.  Only `C` would have the disadvantage as it is not pulling its liquidity as soon as it can. 

So far - besides a rather complicated flow of information - I do not see why the principles of my suggestion would not be possible to work at any other point of the channel state machine. So when queried by `B` the node  `D` could always replay with the latest state it has in the C->D channel indicating to `B` that `C` was dishonest.

Of course we could ask now what is if `B` is also malicious? In this case `B` could propagate the `blame_channel` back but `A` could again use the onion trick to verify and discover that `B` and `C` are not following the protocol. 


Limitations:
=========
There are two limitations that I am currently aware of:

1. Being able to ascribe blame does not directly prevent spam via HTLCs 
2. With MPP the short `reply_interval` might be an issue as the honest recipient of a partial payment might just not have received enough parts of the entire payment and just can't fulfill the payment yet. In this case the blame is being ascribed to the recipient and nodes would learn the recipient. I hope we could resolve such issues by selecting a higher grace period until we ascribe blame. For example if all the times above are are added to the 60 seconds of timeout that recipients of MPP payments currently have this would not trigger if the sender just needs more time to send partial payments.

Advantages:
==========
1. Through the mechanism to ascribe blame several nodes will learn about an malicious or slow actor on the network and can take other preventive measures especially if they also have direct channels with that peer
2. Once we have PTLCs and a protocol for stuckless payments an honest sender of the payment may quickly discard this stuck HTLC (PTLC) and try another path without including the malicious node.

Conclusion:
=========
I am not sure if the suggestions in this proposal are as secure as we need them to be but I wasn't able to detect any obvious flaws which is why I would like to kindly ask you to review and criticize them. What I propose is not the full solution to nodes abusing the possibilities of onion routing and HTLCs spam but I believe it is a step in the right direction towards mitigation by presenting a collection of a few observations which are hopefully useful to bring us a step closer towards guaranteeing fast settlement of payments. 

