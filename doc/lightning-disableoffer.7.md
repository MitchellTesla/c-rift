lightning-disableoffer -- Command for removing an offer
=======================================================

SYNOPSIS
--------
**(WARNING: experimental-offers only)**

**disableoffer** *offer\_id*

DESCRIPTION
-----------

The **disableoffer** RPC command disables an offer, so that no further
invoices will be given out.

We currently don't support deletion of offers, so offers are not
forgotten entirely (there may be invoices which refer to this offer).

EXAMPLE JSON REQUEST
------------
```json
{
  "id": 82,
  "method": "disableoffer",
  "params": {
    "offer_id": "713a16ccd4eb10438bdcfbc2c8276be301020dd9d489c530773ba64f3b33307d ",
  }
}
```

RETURN VALUE
------------

Note: the returned object is the same format as **listoffers**.

[comment]: # (GENERATE-FROM-SCHEMA-START)
On success, an object is returned, containing:

- **offer\_id** (hash): the merkle hash of the offer
- **active** (boolean): Whether the offer can produce invoices/payments (always *false*)
- **single\_use** (boolean): Whether the offer is disabled after first successful use
- **bolt12** (string): The bolt12 string representing this offer
- **used** (boolean): Whether the offer has had an invoice paid / payment made
- **label** (string, optional): The label provided when offer was created

[comment]: # (GENERATE-FROM-SCHEMA-END)

EXAMPLE JSON RESPONSE
-----
```json
{
   "offer_id": "053a5c566fbea2681a5ff9c05a913da23e45b95d09ef5bd25d7d408f23da7084",
   "active": false,
   "single_use": false,
   "bolt12": "lno1qgsqvgnwgcg35z6ee2h3yczraddm72xrfua9uve2rlrm9deu7xyfzrcgqvqcdgq2z9pk7enxv4jjqen0wgs8yatnw3ujz83qkc6rvp4j28rt3dtrn32zkvdy7efhnlrpr5rp5geqxs783wtlj550qs8czzku4nk3pqp6m593qxgunzuqcwkmgqkmp6ty0wyvjcqdguv3pnpukedwn6cr87m89t74h3auyaeg89xkvgzpac70z3m9rn5xzu28c",
   "used": false
}

```


AUTHOR
------

Rusty Russell <<rusty@rustcorp.com.au>> is mainly responsible.

SEE ALSO
--------

lightning-offer(7), lightning-listoffers(7).

RESOURCES
---------

Main web site: <https://github.com/ElementsProject/lightning>

[comment]: # ( SHA256STAMP:e03f739fb57f18205421785604ea542e931db42b1accfcb196dfc147a7c8bf75)
