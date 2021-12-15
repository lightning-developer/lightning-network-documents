# A blame ascribing protocol towards ensuring time limitation of stuck HTLCs in flight.

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

