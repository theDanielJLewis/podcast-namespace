# The "podcast:value" Specification

<small>Version 1.4 by [Dave Jones](https://github.com/daveajones) - with [Gigi](https://github.com/dergigi)
, [Evan Feenstra](https://github.com/evanfeenstra) & [Paul Itoi](https://github.com/pitoi)</small><br>
<small>September 1st, 2021</small>

## Purpose

Here we describe an additional "block" of XML that gives podcasters (and other media creators) a way to receive direct
payments from their audience in response to the action of viewing or listening to a created work. Utilizing so called "
Layer 2"
cryptocurrency networks like Lightning, and possibly other digital currency mechanisms, near real-time micropayments can
be directly sent from listener or viewer to the creator without the need for a payment processor or other "middle men".

The value block designates single or multiple destinations for these micro-payments. The idea is that the creator of the
media
describes (within the feed) where and how they would like payments to be sent back to them during consumption
of that media. The format is designed to be flexible enough to handle many scenarios of direct payment streaming. Even
the use of fiat currency, if there is an API that is capable of interfacing as a receiver within this format.

## Play to Pay

This system can be thought of as Play-to-pay, rather than the traditional Pay-to-play paywall approach. When a
media listener (such as within a podcast app) presses the play button on an episode who's feed contains a value
block, a compatible app will be expected to begin streaming micro-payments to the designated recipients on a time
interval that makes sense for the app. The amount of each payment can be any amount the listener chooses, including
zero. If the listener chooses not to pay to listen to this media, then the app can ignore the value block of that feed.

## Payment Intervals

The "time interval" for calculating payments is **always 1 minute**. However, the actual interval between when payments
are sent can be longer. The interval should be chosen with a few factors in mind such as connectivity (is the app
currently on-line?), transaction fees (batch payments together to reduce fee percentage), cryptocurrency network load
(can the given crypto network or API support this payment rate?).

No matter what the chosen time interval for the actual transaction, the calculation should be done on a once-per-minute
basis. So, if the micro-payment is sent every 15 minutes, it should be calculated as 15 payments batched together
in a single transaction. Likewise, some apps with limited connectivity may need to only send payments once per
hour. In that scenario, there would be 60 payments added up into a single, larger payment. Batching transactions
like this also helps to minimize the impact of transaction fees on the underlying cryptocurrency network.

Note that playback speed is not a factor in this calculation. The "one minute payment interval" refers to the minutes
that make up the total runtime of the content, thus all payment calculations are independent of playback speed.

## Elements

There are two elements that make up what we refer to as the "value block". They are a parent element that specifies the
currency to use, and one or more child elements that specify who to pay using the currency and protocol described by the
parent.

### Value Element

The `<podcast:value>` tag designates the cryptocurrency or payment layer that will be used, the transport method for
transacting
the payments, and a suggested amount denominated in the given cryptocurrency.

This element can exist at either the `<channel>` or `<item>` level.

#### Structure:

```xml

<podcast:value
        type="[cryptocurrency or layer(string)]"
        method="[payment transport(string)]"
        suggested="[payment amount(float)]"
>
    [one or more "podcast:valueRecipient" elements]
</podcast:value>
```

#### Attributes:

- `type` (required) This is the service slug of the cryptocurrency or protocol layer.
- `method` (required) This is the transport mechanism that will be used.
- `suggested` (optional) This is an optional suggestion on how much cryptocurrency to send with each payment.

#### Explanation:

Using Lightning as an example, the `type` would be "lightning". Various possible `type` values will be kept
in a slug list [here](valueslugs.txt). This is the only type currently in active use. Others are under development
and will be added to the list as they see some measure of adoption, or at least a working example to prove viability.

The `method` attribute is used to indicate a sub-protocol to use within the given `type`. Again, returning to
Lightning as an example, the `method` would be "keysend". Normally, a Lightning payment requires an invoice
to be generated by the payee in order to fulfill a transaction. The "keysend" protocol of Lightning allows payments
to be streamed to what is, essentially, an open invoice. Other cryptocurrencies may have a similar protocol that
would be used here. If not, a value of "default" should be given.

The "suggested" amount is just that. It's a suggestion, and must be changeable by the user to another value, or
to zero. The suggested amount depends on the payment protocol being used. For instance, with Lightning on the Bitcoin
network, the amount can be as low as one millisatoshi, expressed as `0.00000000001` BTC.

A single value tag can contain many `<podcast:valueRecipient>` tags as children. All of these given recipients are
sent individual payments when the payment interval triggers.

The value tag, when it exists at the `<channel>` level, designates the payment scheme for the entire podcast. When it
exists at the `<item>` level, it is intended to override the channel level tag for that episode only.

#### Example:

The following example uses the `keysend` method of the `lightning` network and
sets the suggested value to 5 sats. A sat, short for satoshi, is a
one-hundred-millionth of a single bitcoin (0.00000001 BTC). The smallest unit on
the Lightning Network is a millisat, which is a thousandth of a sat.

```xml

<podcast:value
        type="lightning"
        method="keysend"
        suggested="0.00000005000"
></podcast:value>
```

### Value Recipient Element

The `valueRecipient` tag designates various destinations for payments to be sent to during consumption of the enclosed
media. Each recipient is considered to receive a "split" of the total payment according to the number of shares given
in the `split` attribute.

This element may only exist within a parent `<podcast:value>` element.

There is no limit on how many `valueRecipient` elements can be present in a given `<podcast:value>` element.

#### Structure:

```xml

<podcast:valueRecipient
        name="[name of recipient(string)]"
        type="[address type(string)]"
        address="[the receiving address(string)]"
        customKey="[optional key to pass(mixed)]"
        customValue="[optional value to pass(mixed)]"
        split="[share count(int)]"
        fee=[true|false]
        />
```

#### Attributes:

- `name` (recommended) A free-form string that designates who or what this recipient is.
- `customKey` (optional) The name of a custom record key to send along with the payment.
- `customValue` (optional) A custom value to pass along with the payment. This is considered the value that belongs to
  the `customKey`.
- `type` (required) A slug that represents the type of receiving address that will receive the payment.
- `address` (required) This denotes the receiving address of the payee.
- `split` (required) The number of shares of the payment this recipient will receive.
- `fee` (optional) If this attribute is not specified, it is assumed to be false.

#### Explanation:

The `name` is just a human readable description of who or what this payment destination is. This could be something
simple like
"Podcaster", "Co-host" or "Producer". It could also be more descriptive like "Ronald McDonald House Charity", if a
podcaster
chooses to donate a percentage of their incoming funds to a charity.

The `type` denotes what type of receiving entity this is. For instance, with lightning this would typically be "node".
This would
indicate that the `address` attribute for this recipient is a Lightning node that is capable of directly receiving
incoming keysend payments. Valid values for
the `type` attribute are kept in the accompanying file [here](valueslugs.txt). Another option is given in examples
below.

Payments must be sent to a valid destination which is given in the `address` attribute. This address format will vary
depending on
the underlying currency being used.

The `split` attribute denotes an amount of "shares" of the total payment that the recipient will receive when each timed
payment is made.
When a single `<podcast:valueRecipient>` is present, it should be assumed that the `split` for that recipient is 100%,
and the "split" should
be ignored. When multiple recipients are present, a share calculation (see below) should be made to determine how much
to send to each recipient's address.

The `fee` attribute tells apps whether this split should be treated as a "fee", or a normal split. If this attribute is
true, then this split should be calculated
as a fee, meaning its percentage (as calculated from the shares) should be taken off the top of the entire transaction
amount. This is the preferred way for service
providers such as apps, hosting companies, API's and third-party value add providers to add their fee to a value block.

#### Custom Key/Value Pairs

The `customKey` and `customValue` pair can be used (especially for the Lighning Network) to help a receiving application
route or process payments that have all arrived at one node.

The idea is that Podcast Index will parse and store and all client apps will always send a `customKey:customValue` pair
if these are found in the Value Block.

For example, the `customKey`'s of `818818`, `112111100` are used to route payments to Hive accounts or specific wallets
on LNPay respectively. These fields are
documented [in the list maintained by Satoshis Stream](https://github.com/satoshisstream/satoshis.stream/blob/main/TLV_registry.md)
.

If your specific application would benefit from your own `customKey:customValue` pair which will be passed along from
the player to your app, and for which nothing already exists, add your own.

### Payment calculation

The interval payment calculation is:

    (Number of shares / Share total) * Interval payout * Interval count

To calculate payouts, let's take the following value block as an example:

```xml

<podcast:value type="lightning" method="keysend" suggested="0.00000015000">
    <podcast:valueRecipient
            name="Host"
            type="node"
            address="02d5c1bf8b940dc9cadca86d1b0a3c37fbe39cee4c7e839e33bef9174531d27f52"
            split="50"
    />
    <podcast:valueRecipient
            name="Co-Host"
            type="node"
            address="032f4ffbbafffbe51726ad3c164a3d0d37ec27bc67b29a159b0f49ae8ac21b8508"
            split="40"
    />
    <podcast:valueRecipient
            name="Producer"
            type="node"
            address="03ae9f91a0cb8ff43840e3c322c4c61f019d8c1c3cea15a25cfc425ac605e61a4a"
            split="10"
    />
</podcast:value>
```

This block designates three payment recipients. On each timed payment interval, the total payment will be split into 3
smaller
payments according to the shares listed in the split for each recipient. So, in this case, if the listener decided to
pay 100 sats per minute for listening
to this podcast, then once per minute the "Host" would be sent 50 sats, the "Co-Host" would be sent 40 sats and the
"Producer" would be sent 10 sats - all to their respective lightning node addresses using the "keysend" protocol.

If, instead of a 50/40/10 (total of 100) split, the splits were given as 190/152/38 (total of 380), the respective
payment amounts each minute would still
be 50 sats, 40 sats and 10 sats because the listener chose to pay 100 sats per minute, and the respective shares (as a
percentage of the total) would remain the same.

On a 190/152/38 split, each minute the payment calculation would be:

- Interval payout: 100 sats

- Share total: 380

  - Recipient #1 gets a payment of: 50 sats (190 / 380 \* 100)
  - Recipient #2 gets a payment of: 40 sats (152 / 380 \* 100)
  - Recipient #3 gets a payment of: 10 sats (38 / 380 \* 100)

If an app chooses to only make a payout once every 30 minutes of listening/watching, the calculation would be the same
after multiplying
the per-minute payment by 30:

- Interval payout: 3000 sats (100 \* 30)

- Shares total: 380

  - Recipient #1 gets a payment of: 1500 sats (190 / 380 \* 3000)
  - Recipient #2 gets a payment of: 1200 sats (152 / 380 \* 3000)
  - Recipient #3 gets a payment of: 300 sats (38 / 380 \* 3000)

As shown above, the once per minute calculation does not have to actually be sent every minute. A longer payout period
can be chosen. But,
the once-per-minute nature of the payout still remains in order for listeners and creators to have an easy way to
measure and calculate how much
they will spend and charge.

### Supported Currencies and Protocols

The value block is designed to be flexible enough to handle most any cryptocurrency, and even fiat currencies with a
given
API that exposes a compatible process.

Currently, development is centered mostly on [Lightning](https://github.com/lightningnetwork) using the "keysend"
method. Keysend allows for push
based payments without the recipient needing to generate an invoice to receive them.
Another method to send spontaneous payments via Lightning is AMP, atomic
[multipath payments][AMP]. AMP supersedes keysend in many ways and allows for
more robust and larger payments. However, it is still in beta and thus not
widely implemented as of now.

[AMP]: https://bitcoinops.org/en/topics/multipath-payments/

#### Lightning

For the `<podcast:value>` tag, the following attributes MUST be used:

- `type` (required): "lightning"
- `method` (required): "keysend" or "amp"
- `suggested` (optional): A float representing a BTC amount.
  e.g. 0.00000005000 is 5 Sats.

For the `<podcast:valueRecipient>` tag, the following attributes MUST be used:

- `type`: "node"
- `address`: \<the destination node's pubkey\>
- `split`: \<the number of shares\>

If the receiving Lightning node, or service, requires application specific data to be sent with the payment in the
lightning message `extension` (a _TLV stream_, see the Appendix section), the `customKey` and `customValue` can be
utilized as follows:

- `type`: "node"
- `method`: "keysend" or "amp"
- `customKey`: \<tlv record type, a 64 bit integer greater than or equal to 2^16\>
- `customValue`: \<tlv record value, a string\>
- `address`: \<the destination node's pubkey\>
- `split`: \<the number of shares\>

When sending a payment containing application specific data, the client must use UTF-8 as encoding for `customValue`.

**Remarks:**

- `customValue` is specified as a string due to the emergence of known users for this field (see Appendix). If we decide
  to support raw binary data in the future, a new attribute can be introduced to indicate the different behavior
- There is at least one known shared node ([satoshis.stream](https://satoshis.stream/)) that requires, in addition to
  this specification, the inclusion of the TLV record with type `7629169`, as
  defined [here](blip-0010.md), in order to correctly route the payment to the corresponding receiver

##### Example

This is a live, working example of a Lightning keysend value block in production. It designates four recipients for
payment - two
podcast hosts at 49 and 46 shares respectively, a producer working on per episode chapter creation who gets a 5 share,
and
a single share (effectively 1%) fee to the Podcastindex.org API.
Since the value block is defined at the `<channel>` level, it applies to every podcast episode.

```xml
...
<channel>
    <podcast:value type="lightning" method="keysend" suggested="0.00000015000">
        <podcast:valueRecipient
                name="Adam Curry (Podcaster)"
                type="node"
                address="02d5c1bf8b940dc9cadca86d1b0a3c37fbe39cee4c7e839e33bef9174531d27f52"
                split="49"
        />
        <podcast:valueRecipient
                name="Dave Jones (Podcaster)"
                type="node"
                address="032f4ffbbafffbe51726ad3c164a3d0d37ec27bc67b29a159b0f49ae8ac21b8508"
                split="46"
        />
        <podcast:valueRecipient
                name="Dreb Scott (Chapter Creation)"
                type="node"
                address="02dd306e68c46681aa21d88a436fb35355a8579dd30201581cefa17cb179fc4c15"
                split="5"
        />
        <podcast:valueRecipient
                name="Podcastindex.org (Donation)"
                type="node"
                address="03ae9f91a0cb8ff43840e3c322c4c61f019d8c1c3cea15a25cfc425ac605e61a4a"
                split="1"
                fee="true"
        />
    </podcast:value>
    ...
    <item>...</item>
    <item>...</item>
    ...
</channel>
```

To use Atomic Multipath Payments (AMP) instead of `keysend`, simply set the
payment `method` to `amp`:

```xml
...
<channel>
    <podcast:value type="lightning" method="amp" suggested="0.00000015000">
        ...
    </podcast:value>
</channel>
```

##### Example: `<Item>` Override

To set up different payment splits for individual episodes, a value block has to
be defined on the `<item>` level. This will override the value settings set on
the `<channel>` level.

The following example defines different value blocks for each episode in order
to include the guests as value recipients. Payments are split 50/50 between host
and guest.

```xml
...
<channel>
    <podcast:value type="lightning" method="keysend" suggested="0.00000021000">
        <podcast:valueRecipient
                name="John Vallis (Host)"
                type="node"
                address="02a9cd2bca29dd7e29bdfdf485a8e78b8ccf9327517afa03a59be8f62a58792e1b"
                split="100"
        />
    </podcast:value>
    ...
    <item>
        <title>#00 - John's Solo Episode</title>
        ...
    </item>
    <item>
        <title>#01 - John and Gigi</title>
        <podcast:value type="lightning" method="keysend" suggested="0.00000021000">
            <podcast:valueRecipient
                    name="John Vallis (Host)"
                    type="node"
                    address="02a9cd2bca29dd7e29bdfdf485a8e78b8ccf9327517afa03a59be8f62a58792e1b"
                    split="50"
            />
            <podcast:valueRecipient
                    name="Gigi (Guest)"
                    type="node"
                    address="02e12fea95f576a680ec1938b7ed98ef0855eadeced493566877d404e404bfbf52"
                    split="50"
            />
        </podcast:value>
        ...
    </item>
    <item>
        <title>#02 - John and Paul</title>
        <podcast:value type="lightning" method="keysend" suggested="0.00000021000">
            <podcast:valueRecipient
                    name="John Vallis (Host)"
                    type="node"
                    address="02a9cd2bca29dd7e29bdfdf485a8e78b8ccf9327517afa03a59be8f62a58792e1b"
                    split="50"
            />
            <podcast:valueRecipient
                    name="Paul Itoi (Guest)"
                    type="node"
                    address="03a9a8d953fe747d0dd94dd3c567ddc58451101e987e2d2bf7a4d1e10a2c89ff38"
                    split="50"
            />
        </podcast:value>
        ...
    </item>
    ...
</channel>
```

### TLV Records and Extensions

Lightning payments are performed using lightning messages as specified
in [BOLT #1: Base Protocol](https://github.com/lightningnetwork/lightning-rfc/blob/master/01-messaging.md).

One part of the message is the `extension`, a TLV (Type-Length-Value) stream. Podcast specific data can be added to
transactions using the key **7629169** with the value described in [bLIP 10](blip-0010.md)

A community maintained registry of additional known custom record types and formats, governed by satoshis.stream, can be
found at the document [TLV record registry](https://github.com/satoshisstream/satoshis.stream/blob/main/TLV_registry.md)
. In special, the section _Fields used in customKey / customValue Pairs_ documents the known use cases for
the `customKey` and `customValue` attributes.

### Payment Actions

There are currently 3 payment "actions" that are defined in the BLIP-10 TLV extension that is embedded in the payment
payload: "stream", "boost" and "auto".

- `stream` - This means the payment is a timed interval payment (i.e. - every minute) that is sent or queued while the
  user is engaged in active listening/viewing of the content.
- `boost` - This means the payment is a user generated one-time payment that happens in response to a user initiated
  action like a "Boost" button push, or some other clearly labeled payment initiation function. These
  types of payments can contain textual messages (i.e. - a boostagram).
- `auto` - This means the payment was an app initiated payment that recurs at a specific long-form interval such as
  weekly, monthly, yearly, etc. The user configures an interval and payment amount in the app and the app
  then sends that amount at the specified time when each interval passes.

### Value Recipient Address Types

There are a few different available recipient address types:

- `node` - The public address of a node. For instance, in the `lightning` value type this would represent a node's
  public address.
- `lnaddress` - A so-called "lightning address", which takes the form of an email address that gets resolved into an
  options file which holds the underlying destinations for payment. See the full document
  [here](lnaddress.md) for explanation.

More recipient address types will be added in the future.
