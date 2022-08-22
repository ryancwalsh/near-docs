---
id: series
title: Customizing the NFT Contract
sidebar_label: Lazy Minting, Collections, and More!
---

In this tutorial, you'll learn how to take the existing NFT contract you've been working with and modify it to meet some of the most common needs in the ecosystem. This includes:
- Lazy Minting NFTs
- Creating Collections
- Restricting Minting Access
- Highly Optimizing Storage 
- Hacking Enumeration Methods

## Introduction

While the current NFT contract works really well for most simple use cases, it's really meant to be a foundation for creating unique and creative use-cases. The purpose of the zero to hero tutorial is to help garner a deep understanding of the tools such that you can create your own custom contracts to meet whatever use cases you might have. 

While we may not have the instructions for baking blueberry muffins, if you understand how to create the batter for regular muffins, hopefully you'll know that by simply adding blueberries to the mix or altering the amount of salt and vanilla, you'll be able to create your own delicious blueberry muffins!

<img width="45%" src="/docs/assets/nfts/customizing-logic-meme.png" />

### Collections

One of the biggest problems with the current NFT repository is repeated data. Currently, if you mint NFTs, you'll need to store the metadata in the contract every single time. While this is fine for NFTs with drastically different metadata, having repeated similar data is screaming for optimization. In addition, one of the most common use cases for NFTs currently is *collections*.

A collection is a set of NFTs that are related to each other in some way. More often than not, this means they share the same media, title, description, number of copies, etc. If you wanted to make a collection of 1000 NFTs with the current contract, you'd need to store this duplicate information 1000 times when in reality, the only thing changing between the NFTs in the collection is the token ID. We can fix this by introducing the idea of a series. 

A series can be thought of as a bucket. This bucket will contain NFTs and all the NFTs in that bucket will derive from a set of pre-defined information specified when the series was created. You can specify the metadata, and the royalties whereby all NFTs in the series will inherit from.

### Restricted Access

Currently, the NFT contract allows anyone to mint NFTs. While this works well for some projects, the vast majority of dApps and creators want to restrict who can create NFTs on the contract. This is why you'll introduce an allowlist functionality for both series and for NFTs. You'll have two data structures customizable by the contract owner:
- Approved Minters
- Approved Creators

If you're an approved minter, you can freely mint NFTs for any given series. You cannot, however, create new series.

On the other hand, you can also be an approved creator. This allows you to define new series that NFTs can be minted from. It's important to note that if you're an approved creator, you're not automatically an approved minter as well. Each of these permissions need to be given by the owner of the contract and they can be revoked at any time.

### Lazy Minting

Lazy minting is a really powerful feature that can save users from having to spend a ton of $NEAR on potentially non profitable NFTs. To understand what it is, let's look at a common scenario: 

Benji has created an amazing digital painting of the famous Go Team gif. He wants to sell 1000 copies of it for 1 $NEAR each. Using the traditional approach, he would have to mint each copy individually and pay for the storage himself. He would then need to pay for the storage to put 1000 copies up for sale on a marketplace contract, and he would need to put each up for sale **individually** which would cost a lot in Gas. After that, people would purchase the NFTs, and there would be no guarantee that all or even any would be sold. There's a real possibility that nobody buys a single piece of his artwork, and Benji spent all that time and effort and money on nothing. 😢  

Lazy minting would allow the NFTs that Benji specified in the series to be *automatically minted on-demand*. Rather than having to purchase NFTs from a marketplace, a user could directly call the `nft_mint` function. Benji could specify a price within the series data and then the caller would need to attach enough $NEAR to cover both the storage costs and the price specified by Benji.

Using this model, NFTs would **only** be minted when they're actually purchased and there wouldn't be any upfront fee that Benji needs to pay in order to mint all 1000 NFTs. In addition, it removes the need to have a separate marketplace contract.

With this example laid out, a high level overview of lazy minting is that it gives the ability for someone to mint "on-demand" - they're lazily minting the NFTs instead of having to mint everything up-front even if they're unsure if there's any demand for the NFTs. With this model, you don't have to waste Gas or storage fees because you're only ever minting when someone actually purchases the artwork.


## New Contract File Structure

Now that you have a good understand of what we're tying to solve with this contract, let's start analyzing how it's all implemented. The first thing to do is to make sure you're on the main branch in the `nft-tutorial` repo. If you've already cloned the repository, make sure to pull the latest changes as you might not have the series contract yet.

```bash
git checkout main && git pull
```

You'll notice that there's a folder in the root of the project called `nft-series`. This is where the contract code lives. If you open the `src` folder, it should look similar to the following.

```
src
├── approval.rs
├── enumeration.rs
├── events.rs
├── internal.rs
├── lib.rs
├── metadata.rs
├── nft_core.rs
├── owner.rs
├── royalty.rs
├── series.rs
```

## Differences

If you sift through the code in these files, you'll notice that most of it is the same. There are only a few differences between this contract and the current NFT contract. 

### Main Library File 

Starting with `lib.rs`, you'll notice that the contract struct has been modified to now store the following information.

```diff
pub owner_id: AccountId,
+ pub approved_minters: LookupSet<AccountId>,
+ pub approved_creators: LookupSet<AccountId>,
pub tokens_per_owner: LookupMap<AccountId, UnorderedSet<TokenId>>,
pub tokens_by_id: UnorderedMap<TokenId, Token>,
- pub token_metadata_by_id: UnorderedMap<TokenId, TokenMetadata>,
+ pub series_by_id: UnorderedMap<SeriesId, Series>,
pub metadata: LazyOption<NFTContractMetadata>,
```

Most of the information is the same, although we've added 2 new lookup sets and we've changed the `token_metadata_by_id` to be `series_by_id`.

- **approved_minters**: Keeps track of accounts that can call the `nft_mint` function.
- **approved_creators**: Keeps track of accounts that can create new series.
- **series_by_id**: Map a series ID (u64) to its Series object.

In addition, we're now keeping track of a new object called a `Series`.

```rust
pub struct Series {
    // Metadata including title, num copies etc.. that all tokens will derive from
    metadata: TokenMetadata,
    // Royalty used for all tokens in the collection
    royalty: Option<HashMap<AccountId, u32>>,
    // Set of tokens in the collection
    tokens: UnorderedSet<TokenId>,
    // What is the price of each token in this series? If this is specified, when minting,
    // Users will need to attach enough $NEAR to cover the price.
    price: Option<Balance>,
    // Owner of the collection
    owner_id: AccountId,
}
```

This object stores information that each token will inherit from. This includes:
- The metadata.
- The royalties.
- The price.

If a price is specified, there will be no restriction on who can mint tokens in the series. In addition, if the `copies` field is specified in the metadata, **only** that number of NFTs can be minted. If the field is omitted, an unlimited amount of tokens can be minted.

We've also added a field `tokens` which keeps track of all the token IDs that have been minted for this series. This allows us to deal with the potential `copies` cap by checking the length of the set. It also allows us to paginate through all the tokens in the series.

### Series File

Moving on to a new file that has been introduced, you'll notice that there's no longer the `mint.rs` file and it's been replaced by a newer `series.rs` file. This is responsible for everything related to creating series as well as minting NFTs. Starting with the logic for creating a new series, you'll see a new function `create_series`.

```rust reference
https://github.com/near-examples/nft-tutorial/blob/main/nft-series/src/series.rs#L7-L58
```

The function takes in a series ID in the form of a `u64`, the metadata, royalties, and the price for tokens in the series. It will then create the `Series` object and insert it into the contract's `series_by_id` data structure. It's important to note that the caller must be an approved creator and they must attach enough $NEAR to cover storage costs.

#### Minting

Next, we'll look at is the minting function. If you remember from before, this used to take a slew of parameters:
- Token ID
- Metadata
- Receiver ID
- Perpetual Royalties 

With the new minting function, these parameters have been brought down to just two.
- The series ID
- The receiver ID.

Notice how the token ID isn't required? This is because the token ID is automatically generated when minting. The ID is stored on the contract as `${series_id}:${token_id}` where the token ID is a nonce that increases each time a new token is minted in a series. This not only reduces the amount of information stored on the contract but it also acts as a way to check the specific edition number.

The mint function will charge users for storage and if the series has a price, it will ensure that users attach enough $NEAR to cover the price **and** storage. At the end, it will then send all the $NEAR attached minus the storage costs to the **series owner**. It's important to note that in this case, the caller does **not** need to be an approved minter.

If a price wasn't specified, the $NEAR attached will be used to pay for storage and any excess will be refunded to the caller. In addition, all the callers must be approved minters.

```rust reference
https://github.com/near-examples/nft-tutorial/blob/main/nft-series/src/series.rs#L60-L149
```

### View Functions

Now that we've introduced the idea of series, more view functions have been added to support the potentially complex use cases that arise with this contract. With these functions, a new `JsonSeries` struct has been added due to the fact that you cannot serialize an `UnorderedSet` which is being stored in each series. 

```rust reference
https://github.com/near-examples/nft-tutorial/blob/main/nft-series/src/enumeration.rs#L5-L16
```

The view functions are listed below.
- **`get_supply_series`**: Get the total number of series currently on the contract.
- **`get_series`**: Paginate through all the series in the contract and return a vector of `JsonSeries` objects.
- **`get_series_info`**: Get the `JsonSeries` information for a specific series.
- **`nft_supply_for_series`**: View the total number of NFTs minted for a specific series.
- **`nft_tokens_for_series`**: Paginate through all NFTs for a specific series and return a vector of `JsonToken` objects.

Notice how with every pagination function, we've also included a getter to view the total supply? This is so that you can use the `from_index` and `limit` parameters of the pagination functions in conjunction with the total supply so you know where to end your pagination.

### Highjacking View Calls for Optimizations

The last major changes we'll look at pertains to the `nft_token` function. This function is used in most of the current enumeration methods and is a core function that we've changed to reflect the new series concept. 

When creating the contract, we wanted to showcase the idea of "highjacking" enumeration methods. Whenever dApps or users get information from the contract, they go through the enumeration functions. We can use this to our advantage to potentially save on storage.

An important piece of information that users want to know for each token is the edition number. If you own an NFT, you'll want to know which edition it is. As a way to show this, we'll append the edition number to the title of the token in its metadata. For example if a token had a title `"My Amazing Go Team Gif"` and the NFT was edition 42, we would want the new title to be `"My Amazing Go Team Gif - 42"`. If the NFT didn't have a title in the metadata, we should use a default title instead. This title would be of the form `Series {} : Edition {}`. As an example: `"Series 42 : Edition 42"`.

Rather than storing this information on the contract for every single token, which takes up space, we're simply going to highjack the `nft_token` function and forcefully modify the title when returning information during view calls. 

```rust reference
https://github.com/near-examples/nft-tutorial/blob/main/nft-series/src/nft_core.rs#L156-L193
```

This idea is extremely powerful as you can potentially save on a ton of storage. As an example: most of the time NFTs don't utilize the the following fields in their metadata.
- issued_at
- expires_at
- starts_at
- updated_at

As an optimization, you could change the token metadata that's **stored** on the contract to not include these fields but then when returning the information in `nft_token`, you could simply add them in as null values.

### Owner File

The last file we'll look at is the owner file found at `owner.rs`. This file simply contains all the functions for getting and setting approved creators and approved minters. Only the owner of the contract can call these functions and update this information.


There are some other smaller changes made to the contract that you can check out if you'd like. The most notable are:
- The `Token` and `JsonToken` objects have been changed to reflect the new series IDs.
- All references to `token_metadata_by_id` have been changed to `tokens_by_id`
- Royalty functions now calculate the payout objects by using the series' royalties.


## Building the Contract

Now that you hopefully have a good understanding of the contract, let's get started building it. Run the following build command to compile the contract to wasm.

```bash
yarn build
```

This should create a new wasm file in the `out/series.wasm` directory. This is what you'll be deploying on-chain. 

## Deployment and Initialization

Next, you'll deploy this contract to the network by using a dev-account. If you've already used one in this tutorial before, make sure you include the `-f` flag.

```bash
near dev-deploy out/series.wasm && export NFT_CONTRACT_ID=$(cat neardev/dev-account)
```
Check if this worked correctly by echoing the environment variable.
```bash
echo $NFT_CONTRACT_ID
```
This should return something similar to `dev-1660936980897-79989663811468`. The next step is to initialize the contract with some default metadata.

```bash
near call $NFT_CONTRACT_ID new_default_meta '{"owner_id": "'$NFT_CONTRACT_ID'"}' --accountId $NFT_CONTRACT_ID
```

If you now query for the metadata of the contract, it should return our default metadata.
```bash
near view $NFT_CONTRACT_ID nft_metadata
```

## Creating Some Series

The next step is to create two different series. One will have a price for lazy minting and the other will simply be a default series. The first step is to create an owner sub-accounts that you can use to create both series

```bash
near create-account owner.$NFT_CONTRACT_ID --masterAccount $NFT_CONTRACT_ID --initialBalance 25 && export SERIES_OWNER=owner.$NFT_CONTRACT_ID
```

You'll now need to create the simple series with no price and no royalties. If you try to run the following command before adding the owner account as an approved creator, the contract should throw an error.

```bash
near call $NFT_CONTRACT_ID create_series '{"id": 1, "metadata": {"title": "SERIES!", "description": "testing out the new series contract", "media": "https://bafybeiftczwrtyr3k7a2k4vutd3amkwsmaqyhrdzlhvpt33dyjivufqusq.ipfs.dweb.link/goteam-gif.gif"}}' --accountId $SERIES_OWNER --amount 1
```

The expected output is an error thrown: `ExecutionError: 'Smart contract panicked: only approved creators can add a type`. If you now add the series owner as a creator, it should work.

```bash
near call $NFT_CONTRACT_ID add_approved_creator '{"account_id": "'$SERIES_OWNER'"}' --accountId $NFT_CONTRACT_ID
```
```bash
near call $NFT_CONTRACT_ID create_series '{"id": 1, "metadata": {"title": "SERIES!", "description": "testing out the new series contract", "media": "https://bafybeiftczwrtyr3k7a2k4vutd3amkwsmaqyhrdzlhvpt33dyjivufqusq.ipfs.dweb.link/goteam-gif.gif"}}' --accountId $SERIES_OWNER --amount 1
```

If you now query for the series information, it should work!

```bash
near view $NFT_CONTRACT_ID get_series
```
Which should return something similar to:
```bash
[
  {
    series_id: 1,
    metadata: {
      title: 'SERIES!',
      description: 'testing out the new series contract',
      media: 'https://bafybeiftczwrtyr3k7a2k4vutd3amkwsmaqyhrdzlhvpt33dyjivufqusq.ipfs.dweb.link/goteam-gif.gif',
      media_hash: null,
      copies: null,
      issued_at: null,
      expires_at: null,
      starts_at: null,
      updated_at: null,
      extra: null,
      reference: null,
      reference_hash: null
    },
    royalty: null,
    owner_id: 'owner.dev-1660936980897-79989663811468'
  }
]
```

Now that you've created the first, simple series, let's create the second one that has a price of 1 $NEAR associated with it. 

```bash
near call $NFT_CONTRACT_ID create_series '{"id": 2, "metadata": {"title": "COMPLEX SERIES!", "description": "testing out the new contract with a complex series", "media": "https://bafybeiftczwrtyr3k7a2k4vutd3amkwsmaqyhrdzlhvpt33dyjivufqusq.ipfs.dweb.link/goteam-gif.gif"}, "price": "1000000000000000000000000"}' --accountId $SERIES_OWNER --amount 1
```

If you now paginate through the series again, you should see both appear.
```bash
near view $NFT_CONTRACT_ID get_series
```

Which has

```bash
[
  {
    series_id: 1,
    metadata: {
      title: 'SERIES!',
      description: 'testing out the new series contract',
      media: 'https://bafybeiftczwrtyr3k7a2k4vutd3amkwsmaqyhrdzlhvpt33dyjivufqusq.ipfs.dweb.link/goteam-gif.gif',
      media_hash: null,
      copies: null,
      issued_at: null,
      expires_at: null,
      starts_at: null,
      updated_at: null,
      extra: null,
      reference: null,
      reference_hash: null
    },
    royalty: null,
    owner_id: 'owner.dev-1660936980897-79989663811468'
  },
  {
    series_id: 2,
    metadata: {
      title: 'COMPLEX SERIES!',
      description: 'testing out the new contract with a complex series',
      media: 'https://bafybeiftczwrtyr3k7a2k4vutd3amkwsmaqyhrdzlhvpt33dyjivufqusq.ipfs.dweb.link/goteam-gif.gif',
      media_hash: null,
      copies: null,
      issued_at: null,
      expires_at: null,
      starts_at: null,
      updated_at: null,
      extra: null,
      reference: null,
      reference_hash: null
    },
    royalty: null,
    owner_id: 'owner.dev-1660936980897-79989663811468'
  }
]
```

## Minting NFTs

Now that you have both series created, it's time to now mint some NFTs. You can either login with an existing NEAR wallet or you can create a sub-account of one of the NFT contract. In our case, we'll use a sub-account.

```bash
near create-account buyer.$NFT_CONTRACT_ID --masterAccount $NFT_CONTRACT_ID --initialBalance 25 && export BUYER_ID=buyer.$NFT_CONTRACT_ID
```

### Lazy Minting

The first workflow you'll test out is lazy minting NFTs. If you remember, the second series has a price associated with it of 1 $NEAR. This means that there are no minting restrictions and anyone can try and purchase the NFT. Let's try it out. In order to view the NFT in the NEAR wallet, you'll want the `receiver_id` to be an account you have logged into the wallet site. This can be anything so let's export it to an environment variable. Run the following command but replace `YOUR_ACCOUNT_ID_HERE` with your actual NEAR account ID.

```bash
export NFT_RECEIVER_ID=YOUR_ACCOUNT_ID_HERE
```
Now if you try and run the mint command but don't attach enough $NEAR, it should throw an error.

```bash
near call $NFT_CONTRACT_ID nft_mint '{"id": "2", "receiver_id": "'$NFT_RECEIVER_ID'"}' --accountId $BUYER_ID
```

Run the command again but this time, attach 1.5 $NEAR.

```bash
near call $NFT_CONTRACT_ID nft_mint '{"id": "2", "receiver_id": "'$NFT_RECEIVER_ID'"}' --accountId $BUYER_ID --amount 1.5
```

This should output the following logs.

```bash
Receipts: BrJLxCVmxLk3yNFVnwzpjZPDRhiCinNinLQwj9A7184P, 3UwUgdq7i1VpKyw3L5bmJvbUiqvFRvpi2w7TfqmnPGH6
	Log [dev-1660936980897-79989663811468]: EVENT_JSON:{"standard":"nep171","version":"nft-1.0.0","event":"nft_mint","data":[{"owner_id":"benjiman.testnet","token_ids":["2:1"]}]}
Transaction Id FxWLFGuap7SFrUPLskVr7Uxxq8hpDtAG76AvshWppBVC
To see the transaction in the transaction explorer, please open this url in your browser
https://explorer.testnet.near.org/transactions/FxWLFGuap7SFrUPLskVr7Uxxq8hpDtAG76AvshWppBVC
''
```

If you check the explorer link, it should show that the owner received on the order of `1.493 $NEAR`. 

<img width="80%" src="/docs/assets/nfts/explorer-payout-series-owner.png" />

### Becoming an Approved Minter

If you try to mint the NFT for the simple series with no price, it should throw an error saying you're not an approved minter.

```bash
near call $NFT_CONTRACT_ID nft_mint '{"id": "1", "receiver_id": "'$NFT_RECEIVER_ID'"}' --accountId $BUYER_ID --amount 0.1
```

Go ahead and run the following command to add the buyer account as an approved minter.

```bash
near call $NFT_CONTRACT_ID add_approved_minter '{"account_id": "'$BUYER_ID'"}' --accountId $NFT_CONTRACT_ID
```

If you now run the mint command again, it should work.

```bash
near call $NFT_CONTRACT_ID nft_mint '{"id": "1", "receiver_id": "'$NFT_RECEIVER_ID'"}' --accountId $BUYER_ID --amount 0.1
```

### Viewing the NFTs in the Wallet

Now that you've received both NFTs, they should show up in the NEAR wallet. Open the collectibles tab and search for the contract with the title `NFT Series Contract` and you should own two NFTs. One should be the complex series and the other should just be the simple version. Both should have ` - 1` appended to the end of the title because the NFTs are the first editions for each series.

<img width="80%" src="/docs/assets/nfts/series-wallet-collectibles.png" />

Hurray! You've successfully deployed and tested the series contract! **GO TEAM!**.

## Conclusion

In this tutorial, you learned how to take the basic NFT contract and iterate on it to create a complex and custom version to meet the needs of the community. You optimized the storage, introduced the idea of collections, created a lazy minting functionality, hacked the enumeration functions to save on storage, and created an allowlist functionality.

You then built the contract and deployed it on chain. Once it was on-chain, you initialized it and created two sets of series. One was complex with a price and the other was a regular series. You lazy minted an NFT and purchased it for `1.5 $NEAR` and then added yourself as an approved minter. You then minted an NFT from the regular series and viewed them both in the NEAR wallet.

Thank you so much for going through this journey with us! I wish you all the best and am eager to see what sorts of neat and unique use-cases you can come up with. If you have any questions, feel free to ask on our Discord or any other social media channels we have.