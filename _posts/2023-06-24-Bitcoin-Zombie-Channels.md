---
title: Fixing Zombie Channels on the Bitcoin Lightning Network
date: 2023-06-24 17:00
categories: [Bitcoin]
tags: [bitcoin,lightning] #TAG names should always be lowercase
---
![img-description](/assets/bitcoin_zombie-arm.png)

## Zombie Channels...ugh
The Bitcoin Lightning Network is quite amazing; it enables instantaneous remmittance of Bitcoin payments at over 1 million transactions per second using a layer 2 network. Its only capable of doing so by connecting channels between peers. However, there are situations where a lightning channel could enter a zombie state where its node operators are no longer functional, such as hardware failure, and the channels become inoperable. 

Lightning channels cannot be dead, only the node operators themselves can be. Hence, 'zombie' channels were coined since there was no ability to close the channels and therefore no way of returning the channel funds back to the operators. Not only was this troublesome for the node operators, but also negatively affects the lightning network as all other nodes must maintain information about the channel(s). 

## Hardware Specs
At the time of this issue, I was running a <a href="https://github.com/raspiblitz/raspiblitz" target="_blank" rel="noopener norefferrer">raspiblitz</a> (raspberry pi running LND):
* lnd v0.15.4-beta
* Raspberry Pi 5.15.32-v8+

## So, what happened?
In my case, my channel partner and I both had our nodes go offline for quite a few weeks. 

On my side, I was messing with a watchtower config and didn't fully understand the changes I was making to my LN config at the time. In over my head, I messed around with a few other config values and botched the node itself. Reverting to last resort options, I decided to factory reset my node, thinking I was safe as I had my on-chain seed phrase along with my node's <a href="https://docs.lightning.engineering/lightning-network-tools/lnd/safety#static-channel-backups-scbs" target="_blank" rel="noopener norefferrer">Static Channel Backup</a>. 

What I didn't realize was that in order to close the channels in the event my peer <i>also </i> went offline and lost his node, I would also need the channel database file (/mnt/hdd/lnd/data/graph/mainnet/channel.db). This file is required in order to recover the lightning funds that were in the channel. And because my partner's node <i>also crashed</i> (not just went offline), both of us did not have a means of recovering that channel database file :\(

After restoring my seed phrase and static channel backup file, issuing a ```lncli pendingchannels``` resulted in the following:
![img-description](/assets/bitcoin_zombie-pendingchannels1.png)

Notice the ```waiting_close_channels``` channel state. 

At first without understanding the issue, I attempted to force close the channels:<br>
```lncli closechannel c2c5e72c007f87325e633233d4c0d27a29dd90a9dac6755333f4d0504383b42a 0```

This resulted in the following:<br>
```[lncli] rpc error: code = Unknown desc = cannot close channel with state: ChanStatusLocalDataLoss|ChanStatusRestored```

## Chantools to the rescue!
Taking a look at the LN documentation, I came across <a href="https://github.com/lightninglabs/chantools" target="_blank" rel="noopener norefferrer">chantools</a>. These are a set of tools designed to help rescue funds locked in lnd channels. Essentially, it's a last restort method of attempting to recover your lightning funds. 
![img-description](/assets/bitcoin_chantools-flowchart.png)

In this case, I attempted to perform a SCB recovery at Step 3. Despite recovery, the channel was still in the ```waiting_close_channel``` state. 

Step 5 is where you can pivot to either doing a zombie recovery, or a more desirable recovery method instead. In my case, because the channel DB was list during factory reset, the only option is to skip to Step 11 in the above flowchart (Manual Intervention), and subsequently perform the Zombie Channel Recovery.

## Using the Zombie Channel Recovery Matcher
Chantools developer <a href="https://github.com/guggero" target="_blank" rel="noopener norefferrer">@guggero</a> provides a website - <a href="https://node-recovery.com/" target="_blank" rel="noopener norefferrer">node-recovery.com</a>  - which helps node operators get in contact with their peers in hopes to recover the channel funds. If both node operators were to register with their public keys and a means of contact, then the database will find a match and attempt to put both operators in contact with one another.

Now in my case, I was already in contact with my peer via email. Unless you are in a similar situation where you have means of communication with your peer, you will need to register for <a href="https://node-recovery.com/" target="_blank" rel="noopener norefferrer">node-recovery</a>.  

## On with the recovery!
The following steps are taking directory from the <a href="https://github.com/lightninglabs/chantools" target="_blank" rel="noopener norefferrer">chantools</a> documentation.

Because I am already in contact with my peer, I am starting from <b>Step 3</b> of the process. 
 * I am User #1
 * My peer is User #2

Starting from Step 3 of chantools, we need to create a <b>.JSON file</b> and send this to my node. The .JSON file indicates each parties node identifier (public key), contact info (optional), and the channels that are part of the scope of impact. If we would have matched through the match utility, a .JSON file would already have been provided to both me and my peer, but in this case, we need to create it manually. 
![img-description](/assets/bitcoin_chantools-json.png)

To do so, I format a new .JSON file with the following values:
```
{
    "node1": {
        "identity_pubkey": "0357a02133bf4a49e222b6cb66f894c3b4878690b0d5310a3916ffa831669d19a2",
        "contact": "[my email or other means of contact]"
    },
    "node2": {
        "identity_pubkey": "039910a225afe022698e41afdc69ef7b23832cb03baf632476b42d1b7ccc596fcf",
        "contact": "[my peer's email or other means of contact]"
    },
    "channels": [
        {
            "short_channel_id": "850067624019296256",
            "chan_point": "c2c5e72c007f87325e633233d4c0d27a29dd90a9dac6755333f4d0504383b42a:0",
            "address": "bc1qaug9u98vqkg32wkrwk3xkjhdg3q6sk57dzhafmjkk4rtnl3g8fssnajx7u",
            "capacity": 110000
        },
	{
            "short_channel_id": "850063225917997057",
            "chan_point": "48c9fc5606311257c24e2310ee766b28ac1054da6d824c807707422034cd914e:1",
            "address": "bc1q2fflxz4890v4ect8xd45ht40u2x04uswvpfztyxgewprxlayhmyqqyg7k8",
            "capacity": 110000
        }
    ]
}
```
* For reference, the ```chan_point``` value is the funding transaction to the channel and is unique in nature, meaning, there is one chan_point per each channel. So, in my case, I had two channels open with my peer. 
* Thankfully, my peer and I only had 110,000 satoshis each in our channels, which, was approximately ~$70 at the time I was troubleshooting this issue. 

You can create the .JSON file directly on your node, or, create it on your local computer and transfer it using something like <a href="https://linuxize.com/post/how-to-use-scp-command-to-securely-transfer-files/" target="_blank" rel="noopener norefferrer">SCP</a>. 
![img-description](/assets/bitcoin_chantools-json2.png)

## Creating the Keys File
After <a href="https://github.com/lightninglabs/chantools#installation" target="_blank" rel="noopener norefferrer">installing chantools</a>on both our nodes, my peer and I must prepare a "keys" file using the .JSON file I created (or more typically, the one provided by node-recovery.com). 

To do this, I ran:<br>
```chantools zombierecovery preparekeys --payout_addr <on-chain wallet address where you'd like the recovered funds delivered to> --match_file <path to .JSON file>```
![img-description](/assets/bitcoin_chantools-preparekeys.png)

This will output a <b>preparedkeys</b> file into a subdirectory <b>"\results"</b> of the current running directory. 
![img-description](/assets/bitcoin_chantools-preparekeys-output.png)

This file will now need to be sent to my peer <b>(User #2)</b>. My peer will then also create their own respective <b>preparedkeys</b> file. Together, both key files can be used to create a Partially Signed Bitcoin Transaction (PSBT). 

When sending this to your peer (email, cloud storage, sftp server, etc.), User #1 will need to propose a <b>fee rate (sat/vB) to process the transaction on-chain</b>. 

## Creating the Offer
Once User #2 receives the preparedkeys file and proposed fee rate, they will need to create an <b>offer</b>. This is to be an agreed-upon value (or rough estimate) for how the funds within the channel are to be distributed amongst the parties. In my case, our channels had small capacity, and were only setup for testing, so we agreed to do a 50:50 split between the funds.

To do this, User #2 can issue the following command on their node from the same directory where the preparedkey files are stored:<br>
```chantools zombierecovery makeoffer --node1_keys preparedkeys-xxx-xx-xx-<pubkey1>.json --node2_keys preparedkeys-xxxx-xx-xx-<pubkey2>.json --feerate 15```

After User #2 executes the ```makeoffer``` command, they will be prompted to enter information and then be provided an output string which will act as the PSBT. The PSBT is signed by the peer which created the offer. 
```
Channel c2c5e72c007f87325e633233d4c0d27a29dd90a9dac6755333f4d0504383b42a:0 (1 of 2):
Capacity: 110000 sat
Funding TXID: https://blockstream.info/tx/c2c5e72c007f87325e633233d4c0d27a29dd90a9dac6755333f4d0504383b42a
Channel info: https://1ml.com/channel/850067624019296256
Channel funding address: bc1qaug9u98vqkg32wkrwk3xkjhdg3q6sk57dzhafmjkk4rtnl3g8fssnajx7u

How many sats should go to you (bc1q6ssk9kkk7wdpygyypm4haenj2pm2w9tectrgd8) before fees?: 55000

Will send:
55000 sats to our address (bc1q6ss____________________) and
55000 sats to the other peer's address (bc1qnxg____________________).

Channel 48c9fc5606311257c24e2310ee766b28ac1054da6d824c807707422034cd914e:1 (2 of 2):
Capacity: 110000 sat
Funding TXID: https://blockstream.info/tx/48c9fc5606311257c24e2310ee766b28ac1054da6d824c807707422034cd914e
Channel info: https://1ml.com/channel/850063225917997057
Channel funding address: bc1q2fflxz4890v4ect8xd45ht40u2x04uswvpfztyxgewprxlayhmyqqyg7k8

How many sats should go to you (bc1q6ssk9kkk7wdpygyypm4haenj2pm2w9tectrgd8) before fees?: 55000

Will send:
55000 sats to our address (bc1q6ss____________________) and
55000 sats to the other peer's address (bc1qnxg____________________).

Current tally (before fees):
To our address (bc1q6ss____________________): 110000 sats
To their address (bc1qnxg____________________): 110000 sats
Estimated fees (at rate 30 sat/vByte): 7965 sats

Current tally (after fees):
To our address (bc1q6ss____________________): 106018 sats
To their address (bc1qnxg____________________): 106018 sats

Done creating offer, please send this PSBT string to the other party to review and sign (if they accept):
cHNidP8BAJoCAAAAAiq0g0NQ0PQzU3XG2____________________________
```

User #2 now sends the output string back to User #1, where they can then sign the PSBT and subsequently broadcast a withdrawal transaction. Peers can exchange this information in whatever medium they choose, as the other user only needs to enter the string as it was outputted. 

## Signing the PSBT (almost there...)
After the PSBT string is provided back to User #1 (or in this case, myself), it must be signed, therefore indicating that the offer is accepted. By doing so, a Bitcoin transaction is created and can be broadcast to the network to release the funds. 

To do so, enter the following:<br>
```chantoools zombierecovery signoffer --psbt <offered_psbt_base64>```
![img-description](/assets/bitcoin_chantools-signoffer.png)

Take note of the outputted Bitcoin transaction - this value will be entered in the final step below.

## Broadcasting the Transaction
Once the Bitcoin transaction is created, it can finally be broadcasted to the network so that it gets included in a block to get mined and stored on the blockchain. The channels will then be closed and the funds disbursed as agreed-upon in the offer.

To do this, User #1 enters the following:<br>
```bitcoin-cli sendrawtransaction <output from psbt signoffer>```
![img-description](/assets/bitcoin_bitcoincli-sendrawtransaction.png)

Finally, after the transaction is included in a block and it gets confirmed, the channels states that were previously showing as ```waiting_close_channels``` are now cleared. 
![img-description](/assets/bitcoin_zombie-pendingchannels2.png)

Additionally, in my node's RideTheLightning web interface, I can see both channels as being ```Remote Force Closed```.
![img-description](/assets/bitcoin_thunderhub-postrecovery.png)

The channels now appear as closed on Lightning Channel Explorers like <a href="https://1ml.com/node/0357a02133bf4a49e222b6cb66f894c3b4878690b0d5310a3916ffa831669d19a2" target="_blank" rel="noopener norefferrer">1ml.com</a> and amboss.space

And best of all, the funds that were previously stuck in the channels have now been recovered to the address that was specified in the preparedkeys file (minus the fee for processing the transaction).
![img-description](/assets/bitcoin_transaction.png)

## References <3 
All credit goes to <a href="https://github.com/guggero" target="_blank" rel="noopener norefferrer">@guggero</a>
* <a href="https://github.com/lightninglabs/chantools/" target="_blank" rel="noopener norefferrer">Chantools Recovery Workflow</a> 
* <a href="https://github.com/lightninglabs/chantools/blob/master/doc/zombierecovery.md" target="_blank" rel="noopener norefferrer">Zombie Recovery</a>
* <a href=" https://github.com/lightningnetwork/lnd/discussions/7610" target="_blank" rel="noopener norefferrer">Github discussion</a>

![img-description](/assets/bitcoin_chantools-flowchart2.png)
