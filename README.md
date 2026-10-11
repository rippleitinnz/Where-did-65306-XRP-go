# Where did 65306 XRP go?

An audit of the XRP Ledger's supply, from the earliest available ledger in January 2013 to today.

## Short version

- Add up every drop of XRP held on the ledger today and you get **65,306.139720 XRP less** than the ledger's own `total_coins` figure. It's missing, not extra. **No XRP has been created.**
- I went back through the ledger's history and found where it went. **65,303.219010 XRP of it, 99.996%, was destroyed by 29 transactions between late 2013 and July 2014.** All 29 hit the same bug in rippled's old payment engine, and that bug was fixed in rippled 0.25.2 in July 2014.
- The remaining **2.92 XRP** is from mid-to-late 2013 plus a scattering of 1–2 drop rounding losses. I'm still tracking it down.
- The ledger has balanced **to the drop** every day since October 2016. Since July 2017 the `XRPNotCreated` invariant has checked every transaction anyway.

Everything here can be checked against any full-history server. Every transaction is listed below in full.

## Why I looked

Following up on a claim the XRPL showed a 65,305 XRP discrepancy between its total supply and what accounts actually during discussions around the xrpld 3.4.1 patch of an overflow bug that *could* have minted XRP. That bug was never exploited, but people were understandably asking whether XRP had been created.

The discrepancy is real. I reproduced it myself. The question was which way it went and where it came from.

## What "total supply" means on the ledger

Every ledger header carries `total_coins`. It started at 100,000,000,000 XRP and only ever goes down, because every transaction fee is destroyed. So:

```
burned = 100,000,000,000 XRP - total_coins
```

XRP can only sit in three places in ledger state:

| Where | Field |
|---|---|
| Account balances | `AccountRoot.Balance` |
| Escrows | `Escrow.Amount` (XRP escrows only) |
| Payment channels | `PayChannel.Amount - PayChannel.Balance` |

Reserves are part of account balances. AMM pools hold their XRP in the pool's own account. Offers, Checks and NFT offers don't hold XRP at all, since they're only promises against a balance. Sponsorship objects would be a fourth place, but the Sponsor amendment isn't enabled on mainnet.

If nothing has gone wrong, those three add up to exactly `total_coins`.

## The check today

At ledger 107,570,434, walking every object with exact integer drops:

| | Objects | XRP |
|---|---|---|
| Account balances | 8,222,590 | 68,440,302,820.762821 |
| Escrows | 14,608 | 31,545,202,376.523354 |
| Payment channels | 2,608 | 22,223.298149 |
| **Held** | | **99,985,527,420.584324** |
| `total_coins` | | 99,985,592,726.724044 |
| **Difference** | | **-65,306.139720** |

Accounts hold *less* than `total_coins`. So XRP left balances without the header recording it as burned. In effect it's burned XRP that was never counted:

| | XRP |
|---|---|
| Burned according to the header | 14,407,273.275956 |
| Actually gone | 14,472,579.415676 |

I also checked that `total_coins` has only ever dropped by exactly the fees charged. It has. The gap is entirely on the balance side.

## Going back in time

Full-history servers keep the complete state of every ledger. So the same audit can be run at any point in the past: read that ledger's `total_coins`, walk that ledger's accounts, and compare.

Ledgers 1 to 32,569 were lost in 2012 and aren't on any server. At **ledger 32,570**, the earliest that exists, the books balance exactly:

```
sum         99,999,999,999.996320 XRP
total_coins 99,999,999,999.996320 XRP
difference  0
```

Normally every transaction lowers balances and `total_coins` by the same fee, so the gap stays put. It only moves when XRP leaks. That makes it searchable by halving: audit the middle of a range, keep the half where the gap changed, and repeat. These are the points I measured:

| Ledger | Roughly when | Gap (XRP) | |
|---|---|---|---|
| 32,570 | 2013-01 | 0.000000 | earliest ledger on full-history servers; books balance exactly |
| 1,591,206 | about 2013-08 | -0.000001 |  |
| 3,149,842 | about 2013-11 | -2.908857 |  |
| 6,267,115 | 2014-04-24 | -5,627.980366 |  |
| 6,656,774 | about 2014-05-16 | -5,627.980382 |  |
| 7,046,433 | about 2014-06-06 | -39,919.778515 |  |
| 7,825,751 | 2014-07-18 | -65,306.130491 |  |
| 9,384,388 | about 2014-10-15 | -65,306.131342 |  |
| 12,501,661 | about 2015-03-27 | -65,306.132241 |  |
| 24,970,753 | 2016-10-21 | -65,306.139720 | Flow amendment activates |
| 107,570,434 | 2026-10 | -65,306.139720 | unchanged since 2016 |

Nearly all of it went in about seven months, from late 2013 to mid July 2014. After that it's a few thousand drops of rounding, then nothing at all from October 2016 on.

## Finding the transactions

For any single transaction the metadata records every balance before and after. The rule is simple: across all accounts, escrows and channels, XRP holdings must change by exactly minus the fee. Anything else and XRP appeared or vanished inside that transaction.

Random sampling found nothing. I checked over 12,000 transactions from the period, including the largest XRP trades, and everything balanced apart from 1–2 drop rounding. The losses turned out to be rare and concentrated.

What did find them was looking at who was trading. In 2014 a handful of arbitrage bots were sending thousands of **partial payments to themselves**, for example USD → XRP → JPY and back to the same account. Checking the complete history of those accounts turned up every one of the large losses.

## One transaction in detail

`0F4667D8100D159A2972E479D86CEC4BE2F610E9E4717AA9C3FD1DE272877F0D`, ledger 6,936,214.

`rDGLD5Ci7CviQQgHz1zkWVzSwapmmKTDrS` pays **itself**: deliver 5,000 JPY, spend up to 47.17 USD, partial payment allowed, routed through XRP. Result `tesSUCCESS`, fee 10 drops.

The metadata:

| Entry | Change | Matches |
|---|---|---|
| `rnqFWtx…` offer (selling XRP for USD) | -12,048.192771 XRP, -50.48 USD | the full 5,000 JPY attempt |
| `rnqFWtx…` XRP balance | **-12,048.192771** | the full attempt |
| `rnqFWtx…` USD received | +10.37 | what actually happened (~20.5%) |
| `rQLEv…` offer (selling JPY for XRP) | -5,000 JPY, -12,048.19 XRP | the full attempt |
| `rQLEv…` XRP balance | **+2,469.724510** | what actually happened |
| JPY delivered to the sender | +1,024.94 | what actually happened |

Two different calculations ended up in the same committed ledger. Debits from an attempt to move the full amount, and credits from the smaller pass that really went through. **12,048.192771 - 2,469.724510 = 9,578.468261 XRP** was taken from one account and given to no one. About 3,975 JPY and 40 USD went the same way on the trust-line side.

## The bug

rippled's payment engine at the time (`RippleCalc`) delivered a payment in passes. Each pass saved a snapshot of the ledger state, tried every path, rolled back between attempts, then kept the best path and went round again.

When a later pass found **no usable path at all**, this branch ran (rippled 0.24.0 / 0.25.1, `RippleCalc.cpp`):

```cpp
// Not done and ran out of paths.
else if (!bPartialPayment)
{
    // Partial payment not allowed.
    terResult   = tecPATH_PARTIAL;
}
// Partial payment ok.
else if (!saDstAmountAct)
{
    // No payment at all.
    terResult   = tecPATH_DRY;
}
else
{
    terResult   = tesSUCCESS;
}
```

The last `else` returns success **without restoring the snapshot**. So whatever the last path it tried had done to the working ledger got committed with the real payment. If that abandoned path had already debited an offer owner's XRP before giving up, the XRP was simply gone.

It only bit when all of these lined up:

1. A **partial payment**. A normal payment that runs out of paths fails with `tecPATH_PARTIAL`, and the working state is thrown away.
2. The payment ended because liquidity ran out, not because `SendMax` was used up. That ending goes through a different branch that's fine.
3. At least one successful pass before the empty one.
4. The last path tried got partway through moving value and then failed.
5. That path had already consumed an XRP offer.

Arbitrage bots running partial self-payments through thin XRP-bridged order books hit 1, 2, 3 and 5 all day. Number 4 is the rare part. That's why it's only 29 transactions out of hundreds of thousands.

The fix is commit [`828c2e3c7`](https://github.com/XRPLF/rippled/commit/828c2e3c7) ("Restore checkpointed ledger under all paths in PathFind", 25 June 2014), tidied up by [`b9f1b0562`](https://github.com/XRPLF/rippled/commit/b9f1b0562). Both shipped in **rippled 0.25.2 on 7 July 2014**:

```cpp
else
{
    // We must restore the activeLedger from lesCheckpoint in the case
    // when iBest is -1 and just before the result is set to tesSUCCESS.

    activeLedger.swapWith (lesCheckpoint);
    terResult   = tesSUCCESS;
}
```

The branch without the restore is identical in every release I checked: 0.19.0 (November 2013), 0.22.0, 0.23.0, 0.24.0, 0.25.0 and 0.25.1. The last loss is at ledger 7,388,938 in late June 2014, a couple of weeks before 0.25.2 went out.

A few things worth saying plainly:

- The bots didn't profit from this. In every case I've taken apart, the account that lost XRP was an **offer owner**, a market maker whose offer sat on the abandoned path.
- At mid-2014 prices, roughly half a US cent, the whole 65,306 XRP was worth a few hundred US dollars. It's not surprising nobody noticed at the time.
- The same leftover state would have damaged issued-currency balances too, in paths that never touched XRP. That doesn't affect the XRP supply, so I haven't measured it, but some 2014 trust-line balances are probably slightly off.

## This wasn't rounding

It's tempting to put this down to the early engine's rounding errors. Those were real: in payments that crossed a lot of offers, the sender was sometimes charged a drop or two more than the offer owners received in total. I found those too, and they come to about 0.01 XRP across 2013–2016.

The 29 transactions above are a different thing:

- **Size.** Rounding loses 0.000001 XRP at a time. These lost between 9 and 11,148 XRP each. Getting to 65,000 XRP by rounding would take tens of billions of rounding errors.
- **Shape.** Rounding looks like "charged 10.000001, paid 10.000000". These are an offer owner debited for the full size of an offer (12,048.19, 1,000, 1,428.57 XRP) with only part of it credited onward. In the transaction above, every debit matches an attempt to fill 5,000 JPY and every credit matches the roughly 20.5% that actually went through. That's two calculations mixed together, not imprecise arithmetic.
- **Timing.** Rounding drift carried on until Flow in 2016. The large losses stop right after 0.25.2 shipped the checkpoint fix in July 2014.
- **Code.** The faulty branch is easy to point at: no usable path on a later pass, `tesSUCCESS`, no `activeLedger` restore.

## The 29 transactions

All amounts are exact, in XRP. "Debited" and "credited" are the XRP balance changes in each transaction's metadata, excluding the sender's 10-drop fee. Debits minus credits is what vanished.

| # | Ledger | Transaction hash | Sender | XRP debited | XRP credited | Vanished (XRP) |
|---|---|---|---|---|---|---|
| 1 | 4,052,441 | `94937D115154B2AD22FA5C45F39E17297B71F62778DBBF383948AF6CA2C15CD7` | `rLCKD9avAYJENYpHzEi4BovVCJZJQZ6KX` | `rUipqpbkHWsEfdHNYVjqViHeM8pUxLzmLe` -215.000000 | `rEx9aixkPDiWZpKE66fJeDQmpUSyeVzLkB` +74.187654 | **140.812346** |
| 2 | 4,052,497 | `C598F961BFB85C6B5E7B51A522456BFF7AF3708D87D7D139DA25118D746AAF8E` | `rLfgAh1m7vVtZjxj5PjD983DbRtSPeRy7F` | `r3FvhzKtvLRiD3CMs3Xo9iytoSy1TMFkuD` -215.000000 | `rEx9aixkPDiWZpKE66fJeDQmpUSyeVzLkB` +205.948289 | **9.051711** |
| 3 | 4,056,164 | `1A31056692F7660C3008993B000BCE10F638E201B9A4A07F62286B5576B9B183` | `rLfgAh1m7vVtZjxj5PjD983DbRtSPeRy7F` | `raPVuJgUNSroYbWPTCU3aiszn8pzhcxoWR` -210.000000 | `rEx9aixkPDiWZpKE66fJeDQmpUSyeVzLkB` +80.078461 | **129.921539** |
| 4 | 4,056,186 | `6EBAFD87144E337C9C7F5A67E4AFD5E4CAD97D4D6A9B9C6204DDBF2CFF1DF4D3` | `rLfgAh1m7vVtZjxj5PjD983DbRtSPeRy7F` | `raPVuJgUNSroYbWPTCU3aiszn8pzhcxoWR` -210.000000 | `rEx9aixkPDiWZpKE66fJeDQmpUSyeVzLkB` +197.547170 | **12.452830** |
| 5 | 4,056,430 | `315EC77F128BA0BD14B27C33A7940F871A033C295F2792A89EE7A8484801F2FD` | `rLfgAh1m7vVtZjxj5PjD983DbRtSPeRy7F` | `rKMdfrrSpSw8oqUs1pw1o7uy7ipqFEDr9A` -210.000000 | `rEx9aixkPDiWZpKE66fJeDQmpUSyeVzLkB` +153.153204 | **56.846796** |
| 6 | 4,056,536 | `359BDFB7841FCA84D2BB712A1C69FB06BB0E7C371B9128C06F5904FE7E26B42B` | `rLCKD9avAYJENYpHzEi4BovVCJZJQZ6KX` | `rEpNyx68G6qy3KWZWf4cY1amPrncoGka93` -210.000000 | `rEx9aixkPDiWZpKE66fJeDQmpUSyeVzLkB` +110.653385 | **99.346615** |
| 7 | 4,214,735 | `7899BF92EB23E9AE3E1CFEA251C990FE9C1DBC6536A105C351D414426CC6BFCE` | `rMKAapx5AUWvUVQ4rFv5jGu3xVTS9sDjqx` | `rsb1PD3qT5bAmeaRB5nwWEQB6iCzKtKPQS` -367.300422<br>`rpk9z22QJQoboCixZSFFd86UgPDRv5eGds` -0.001631 | `rHK6QpWnzqFFZzPTrVP4FUzVAxPTcy2UzY` +83.047697 | **284.254356** |
| 8 | 4,453,224 | `77D8F88B0F1AA8FBFDCDD933EFB1D6A9D159BC0563D0EF9A32A3C3025AADF08B` | `r42rYHtBsvgrUxdmv8GvLEaV5srcjv5kxr` | `r95koB4DXDc7oBo38AUEi51KnU1qURPVwV` -151.540833<br>`rhioxDeqPJffiXBJmucyjLWrtyFHaqbxsx` -183.459167<br>`rQhJNpaJg13TD4Tn2DxZ6vb7fJUoCHi9se` -65.000000 | `rUsnAg9eKJ2AjpkRDpZLnhNXfCHMxRXZMy` +151.540833 | **248.459167** |
| 9 | 4,454,409 | `F77D5CD4705E350632DCCBC9C5A2E20DAEC58AFFE9FD9D479D7AA5BD5FBC8FC1` | `rHTsQWMrieh4TQCEUqdVRN3TUV2ncVbfXH` | `r9CpygojFG5zKXvJaG8wDn6oXWGb87DRP` -400.000000 | `rUsnAg9eKJ2AjpkRDpZLnhNXfCHMxRXZMy` +245.677973 | **154.322027** |
| 10 | (see note) | `(see note)` | `rHTsQWMrieh4TQCEUqdVRN3TUV2ncVbfXH` | `rpP3zg6zZCRr3KL7FGNMz7Es28w8dcSf97` -404.203719 | `rUsnAg9eKJ2AjpkRDpZLnhNXfCHMxRXZMy` +325.788027 | **78.415692** |
| 11 | 5,976,764 | `00B2E14F4936E5EEB4120B7B6AE1332BE484F52341E43311606DCEB93520A390` | `rHpoggSkNY7puahMUGVafWPZQ5JH8piZVQ` | `rDCp46HXhStBY5zgS5h913WtsYtKL55MoR` -999.999999<br>`rsd2qPfwi9NpFt8xrHY3nXcHF5pceoyMj8` -0.000001 | `rJdZXpCSBKyVtdBeZFgcPGjCvybRct68nC` +8.000001 | **991.999999** |
| 12 | 5,977,639 | `ABBFF205A5A954462B34CD498DFF45B01B0046CE29A5E2C5B059E128DCB52CC3` | `rHpoggSkNY7puahMUGVafWPZQ5JH8piZVQ` | `rLcyivdJMj2ZELJkZZLbCr5WVUU7ek1DMu` -1,000.000000 | `rGg7i5p53paiYDrJhmXwdHvYzihJMGdWbT` +140.398224 | **859.601776** |
| 13 | 6,108,330 | `7FCEA00BB1196DBA5BB179F3EC57D9D100E10962EE1B10EDB9CFEDE08F4BEA08` | `rwzGqdQmrGov6kQ9o6YWuaqJxz94UrLnW7` | `rwBYyfufTzk77zUSKEu4MvixfarC35av1J` -378.624876 | `r3uAR9gtU1ZA7Xizor2MTnwAxDNgN998jT` +72.791542 | **305.833334** |
| 14 | 6,198,471 | `32931C7025E967124C2FAAE58E8F529849F022107FCF9C2BF614059F8C5FB264` | `rD8FXhcHtG8m1iHTJogoahPx3LJRjT7tpa` | `rhq3s15LQXFysfWzZ16FbH4Y5ZJDH8Q4yB` -1,333.333334 | `rwNZwd9i9RyPsMCciHWLF86c4yu6qKGVoo` +437.315263 | **896.018071** |
| 15 | 6,226,714 | `01030E435C2EEBE2689FED7494BC159A4C9B98D0DF0B23F7DFCE223822237E8C` | `rHpoggSkNY7puahMUGVafWPZQ5JH8piZVQ` | `rUxXgX1dZgrEZyj644jsMXXrKEFDMphU75` -1,424.501425 | `r9RR643anesxNCoNkkuYfEfzpxZUeK5Qzw` +794.217063 | **630.284362** |
| 16 | 6,227,263 | `FC38AA21CF4419CBEE5AFB5E2AFA7076B6DA3294364DD6F3735D11C59F75E7DA` | `rHpoggSkNY7puahMUGVafWPZQ5JH8piZVQ` | `rHsZHqa5oMQNL5hFm4kfLd47aEMYjPstpg` -1,428.571429 | `rsiTjDRK2bSWaF87mj5m6p86jibVrrAN9e` +701.121131 | **727.450298** |
| 17 | 6,754,728 | `20FC6D31B6B8ACD40CD3362ED9640418CADEBB6A2E5E3F0FBAAEE84026C67EF6` | `rD8FXhcHtG8m1iHTJogoahPx3LJRjT7tpa` | `rhWcbzUj9SVJocfHGLn58VYzXvoVnsU44u` -2,175.631973 | `rw4QHTBYBDVp8whT8Vx6r2PYTGmCsE5k76` +893.425419 | **1,282.206554** |
| 18 | 6,764,486 | `9E49706F4D9CBB1BE12896A67286424CB7071AE1A2F72C89EBA60443B40C1076` | `rJK9mUJZZiKWu4V3KzdziGAu8xthmTXoiv` | `rQasZcnibdbZibG8g7voveNEjRDnqpRrLF` -2,654.867257 | `rJx6zPChNx6e3mMJu9uvZELY3pBoyimubP` +506.748369 | **2,148.118888** |
| 19 | 6,779,184 | `8C67CBA8D2A9EA293D8562D58B14C19DB6C1A30EE38ED3247555406879AFC367` | `rD8FXhcHtG8m1iHTJogoahPx3LJRjT7tpa` | `rUBLCjWdsPPMkppdFXVJWhHnr3FNqCzgG3` -14,705.882353 | `r37RhTPgw8Eu6ucV2Ut47zBBRvU316dmp` +11,618.227552 | **3,087.654801** |
| 20 | 6,781,921 | `D72F5822F16630861C8A94BEC7CC5009845FB8D07C4BED66731B3EADBC6B4320` | `rD8FXhcHtG8m1iHTJogoahPx3LJRjT7tpa` | `rpkj1xuwkjSNFbUzZinzQH9f118YP6FQNw` -6,432.782267 | nobody | **6,432.782267** |
| 21 | 6,830,202 | `558EBD7E0595FA8BAFD96A42429E30F4F0E4A114C260F85691BA82C1AEEB8375` | `rD8FXhcHtG8m1iHTJogoahPx3LJRjT7tpa` | `rJ8mrkA1SjanrjxVrhKiM8f1ju34wfL3Jk` -14,083.577308 | `rJxiLqvkhCzVyk5npe2cAqtV51gKBr1iyv` +3,300.559199<br>`rwNZwd9i9RyPsMCciHWLF86c4yu6qKGVoo` +6,214.715209 | **4,568.302900** |
| 22 | 6,936,214 | `0F4667D8100D159A2972E479D86CEC4BE2F610E9E4717AA9C3FD1DE272877F0D` | `rDGLD5Ci7CviQQgHz1zkWVzSwapmmKTDrS` | `rnqFWtxFPLdQWDhAiTBxtbrHmrqpwkj5rg` -12,048.192771 | `rQLEvSnbgpcUXkTU8VvSzUPX4scukCjuzb` +2,469.724510 | **9,578.468261** |
| 23 | 6,992,066 | `28A6997805C3F4D2413F56776E8BE4FE0CDA3AA3EF75E2F9D1CABDF4F3EC511B` | `rDGLD5Ci7CviQQgHz1zkWVzSwapmmKTDrS` | `rwBYyfufTzk77zUSKEu4MvixfarC35av1J` -10,638.297873 | `racJpvgLpaNQGKB8nhKd1gTEVVA1uQWRKs` +3,444.033473 | **7,194.264400** |
| 24 | 7,076,057 | `2047BF3B2D7FD38AF7EC0107EF46E98F40921B39DF214AF19CDAD4E4DF091355` | `rD8FXhcHtG8m1iHTJogoahPx3LJRjT7tpa` | `rnKaKk3ivygVaDSdw7X8ZkYVoxMgCMvsry` -12,128.562766 | `rQLEvSnbgpcUXkTU8VvSzUPX4scukCjuzb` +980.259738 | **11,148.303028** |
| 25 | 7,226,257 | `411F73CF66101B9454684A24349F38D64248C625C5430D6E469FF2926B7FEAE2` | `rD8FXhcHtG8m1iHTJogoahPx3LJRjT7tpa` | `rEhKZcz5Ndjm9BzZmmKrtvhXPnSWByssDv` -922.503253 | nobody | **922.503253** |
| 26 | 7,351,928 | `193B18D033B08D1715E3B13D17ACFA94122261A0F3A6A9D0CCDEFF16C600AE9B` | `rDGLD5Ci7CviQQgHz1zkWVzSwapmmKTDrS` | `rwBYyfufTzk77zUSKEu4MvixfarC35av1J` -999.999999 | nobody | **999.999999** |
| 27 | 7,366,105 | `58F40AA6F7A5D6CB8610BE1C341E66708FE84585926CA087992D35216C5E4FE1` | `rHpoggSkNY7puahMUGVafWPZQ5JH8piZVQ` | `rJUieDFQxq5wngq5VQvnumJSkEVdy2DVgz` -5,720.823799 | `r37RhTPgw8Eu6ucV2Ut47zBBRvU316dmp` +2,261.834111 | **3,458.989688** |
| 28 | 7,367,000 | `8352F16142114C5A3B7E4D019D383B2ED073CCDC92EFE86BBD58A846A31734FB` | `rDGLD5Ci7CviQQgHz1zkWVzSwapmmKTDrS` | `rDsijnqWPCSj8M1J4QpUzEYRgS2sdSxNyY` -11,483.165680 | `r3H5NQrcRZTyv2yzaq5HDZDfUQgKbzaN7v` +10,623.584145 | **859.581535** |
| 29 | 7,388,938 | `27CD4BF4795D2B3E5397D97270BE49D6CC5E8D72340A4297E7F32C4993AF637E` | `rGsEuH6KAitvAHSMmPpysHNVvaVrnEeYg7` | `rn7uHNfZYNRfxUNGyqUBDL2YVf4NDwap6w` -8,116.883117 | `rQLEvSnbgpcUXkTU8VvSzUPX4scukCjuzb` +119.910600 | **7,996.972517** |
| | | | | | **Total** | **65,303.219010** |

Note on #10: the ledger index and hash for this one got cut off in my terminal...you get the picture though

| Period | Transactions | XRP |
|---|---|---|
| Late 2013 to early 2014 (ledgers 4.05M–4.45M) | 10 | 1,213.883079 |
| April 2014 (ledgers 5.97M–6.23M) | 6 | 4,411.187840 |
| May to June 2014 (ledgers 6.75M–7.39M) | 13 | 59,678.148091 |
| **Total** | **29** | **65,303.219010** |

## Where the numbers stand

| | XRP |
|---|---|
| Gap at ledger 107,570,434 | 65,306.139720 |
| 29 transactions above | 65,303.219010 |
| Rounding losses identified so far | 0.001252 |
| **Still to locate** | **2.919458** |

The 2.92 XRP is about 2.908 XRP between ledgers 1,591,206 and 3,149,842 (mid-to-late 2013), and about 0.011 XRP of 1–2 drop rounding spread through 2014–2016. The old engine rounded XRP separately for each offer a payment crossed, and in big multi-offer payments the sender was sometimes charged a drop or two more than the offer owners received.

## What this means

- **XRP was destroyed, not created.** Holdings are below `total_coins`. Every flagged transaction I found was a loss. The gap never went up at any point I measured.
- **The cause is known and was fixed in 2014.** One bug, one fix, and the losses stop when it shipped.
- **Nothing has leaked since October 2016**, and since July 2017 the `XRPNotCreated` invariant rejects any transaction that would change the XRP total by anything other than its fee.
- In practice the true amount burned is 14,472,579.42 XRP, not the 14,407,273.28 the header implies.

## Check it yourself

Everything here comes straight from a full-history rippled/xrpld server (`s1.ripple.com`, `s2.ripple.com` and `xrplcluster.com` all work). Nothing relies on an explorer or a third-party dataset.

To look at any of the 29 transactions:

```bash
curl -s -X POST https://s2.ripple.com:51234 -H 'Content-Type: application/json' \
  -d '{"method":"tx","params":[{"transaction":"0F4667D8100D159A2972E479D86CEC4BE2F610E9E4717AA9C3FD1DE272877F0D"}]}'
```

Add up the `Balance` changes across every `AccountRoot` in `meta.AffectedNodes`. For a normal transaction the total is minus the fee. For these 29 it's minus the fee minus the amount in the table.

To check the supply at any ledger, page through `ledger_data` with `"type": "account"` (plus `"escrow"` and `"payment_channel"` after ledger 28,685,313) at a fixed `ledger_index`, add up the balances as integer drops, and compare with `total_coins` from `ledger` at the same index. At 32,570 they match exactly. At any ledger from October 2016 onwards they're 65,306.139720 XRP apart.

## Things that tripped me up

- **Explorers that use floating point.** One explorer showed "fees burned: -2,501,010.10101 XRP" for an ordinary ledger. A 64-bit float can't hold 100 billion XRP to the drop, so differences of `total_coins` come out as noise. Keep XRP in integer drops.
- **Gaps in public history.** s1 returned `markerDoesNotExist` partway through one 2014 ledger, so it's missing some state from that period. When that happens the same request on another server works. Every figure here agrees across more than one server.
- **Third-party datasets.** The ClickHouse dataset I used for candidate searches undercounts 2016 account activations by about 11,800 and has a few hundred extra fee rows. It's fine for finding candidates and not for final numbers. All final numbers here come straight from rippled.
- **A full state walk is slow on public servers.** Expect 30–100 accounts a second. Escrows and channels can't exist before ledger 28,685,313, so there's no need to walk them for earlier ledgers.
