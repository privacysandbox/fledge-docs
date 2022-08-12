# FLEDGE Bidding and Auction Services API Proposal

[The Privacy Sandbox][4] aims to develop technologies that enable more private
advertising on the web and mobile devices. Today, real-time bidding and ad
auctions are executed on servers that may not provide technical guarantees of
security. Some users have concerns about how their data is handled to
generate relevant ads and in how that data is shared with ad providers.
FLEDGE (Android, [Chrome][5]) provides ways to preserve privacy and limit
third-party data sharing by serving personalized ads based on previous mobile
app or web engagement.

For all platforms, FLEDGE may require [real-time services][6]. In the initial
[FLEDGE proposal][7], bidding and auction for remarketing ad demand is
executed locally. This can demand computation requirements that may be
impractical to execute on devices with limited processing power, or may be
too slow to render ads due to network latency.

The Bidding and Auction Services API proposal outlines a way to allow FLEDGE
computation to take place on cloud servers in a trusted execution
environment, rather than running locally on a user's device. Moving
computations to cloud servers can help optimize the FLEDGE auction, to free
up computational cycles and network bandwidth for a device. This is not a
requirement for Chrome at this point.

This contrasts with [Google Chrome's initial FLEDGE proposal][7], which
proposes bidding and auction execution to occur locally. There are other
ideas, similar to FLEDGE, that propose server-side auction and bidding, such
as [Microsoft Edge's PARAKEET][8] proposal. Unlike PARAKEET and Criteo's
Gatekeeper designs, this proposal does not involve any communication between
services that cannot be attested.

This document focuses on the API design for FLEDGE's bidding and auction
services. Based on adtech feedback, changes may be incorporated in the
design. This API proposal in this document does not currently cover sell-side
and buy-side reporting, but it will be updated later to incorporate both. A
document that details system design for bidding and auction services will
also be published at a later date.

## Background

Read the [FLEDGE services overview][6] to learn about the environment, trust
model, server attestation, request-response encryption, and other details.

Each FLEDGE service is hosted in a virtual machine (VM) within a secure,
hardware-based trusted execution environment (TEE). Adtech platforms operate
and deploy FLEDGE services on a public cloud. Adtechs may choose the cloud
platform from one of the options that are planned. As cloud customers,
adtechs are the owners and only tenants of such VM instances.

## Bidding and auction system architecture

The following is a high-level overview of the architecture of the bidding and
auction system.

![Diagram of service workflow.](images/fledge-service-workflow.png)

_In this diagram, one seller and one buyer are represented in the service
 workflow. In reality, a single seller auction has multiple participating
 buyers_.

1. The client starts the auction process.
  - The client sends a `SelectWinningAd` request to the `SellerFrontEnd` service.
    This request includes the seller's auction configuration and encrypted input
    for each participating buyer.
1. The `SellerFrontEnd` service orchestrates `GetBid` requests to participating
   buyers’ `BuyerFrontEnd` services.
1. Both the seller and buyer’s front-end services fetch proprietary code.
    - The `SellerFrontEnd` service fetches the seller’s proprietary code required
      to score ads.
  - The `BuyerFrontEnd` services fetch real-time data from the buyer’s key-value
    service, as well as the buyer-owned proprietary code to generate bids.
1. The `BuyerFrontEnd` service sends a `GenerateBids` request to the bidding
   service. The bidding service returns ad candidates with bids.
1. The `BuyerFrontEnd` selects the top eligible ad candidate and returns the
   selection to the `SellerFrontEnd` with `AdWithBid`.
1. Once `SellerFrontEnd` has received bids from all buyers, it requests real-time
   data from the seller’s key/value service required to score the ads for
   auction.
  1. `SellerFrontEnd` sends a `ScoreAds` request to the `auction` service to
       score and select a winner.
1. `SellerFrontEnd` returns the winning ad and additional data to the client to
   render.

### Sell-side platform (SSP) system

The following are the FLEDGE services that are to be operated by an SSP, also
referred to as a seller. Server instances are deployed so that they are
co-located in a data center within a cloud region.

#### SellerFrontEnd service

The `SellerFrontEnd` service orchestrates calls to other adtechs. For a
single-seller auction, this service sends requests to DSPs participating in
the auction for bidding. This service also fetches real-time seller signals
and adtech proprietary code required for the auction.

_Note: In this model and with the_ [_proposed APIs_][9], _a seller can have
 auction execution on a cloud platform only when all buyers participating in
 the auction also operate services for bidding on any supported cloud
 platform. If required for testing purposes, the client-facing API can be
 updated to support seller-run auctions in the cloud seamlessly, without
 depending on a buyer's adoption of services_.

#### Auction service

The `Auction` service only responds to requests from `SellerFrontEnd` service, with
no outbound network access.

For every ad auction request, the `Auction` service executes seller owned auction
code written in JavaScript or WebAssembly in a sandbox instance within a TEE.
Execution of code in the sandbox within a TEE ensures that all input and output
(such as logging, file access, and disk access) is disabled, and the service has
no network or storage access.

The hosting environment protects the confidentiality of the seller's proprietary
code, if the execution happens only in the cloud and proprietary code is fetched
in a `SellerFrontEnd` service.

#### Seller's key/value service

A key/value service is a critical dependency for the auction system. The
[FLEDGE key/value service][10] receives requests from the `SellerFrontEnd`
service in this architecture (or directly from the client in case bidding and
auction runs locally on client's device). The service returns real-time seller
data required for auction that corresponds to lookup keys available in buyers'
bids (such as `ad_render_urls` or `ad_component_render_urls`).

The seller’s key/value system may include other services running in a TEE. The
details of this system are out of scope of this document.

### Demand-side platform (DSP) system

This section describes FLEDGE services that will be operated by a DSP, also
referred to as a buyer. Server instances are deployed such that they are
co-located in a data center within a given cloud region.

#### `BuyerFrontEnd` service

The front-end service of the system that receives requests to generate bids from
a `SellerFrontEnd` service. This service fetches real-time bidding signals, buyer
signals, and proprietary adtech code that is required for bidding.

_Note: With the proposed APIs, a `BuyerFrontEnd` service can also receive requests
directly from the client, such as an Android app or web browser. This is
supported during the interim testing phase so that buyers (DSPs) can roll out
servers independently without depending on seller (SSP) adoption_.

#### Bidding service

The FLEDGE bidding service can only respond to requests from a `BuyerFrontEnd`
service, and otherwise has no outbound network access. For every bidding
request, the service executes buyer-owned bidding code written in JavaScript and
(optional) WebAssembly in a sandbox instance within a TEE. All input and output
(such as logging, file access, and disk access) are disabled, and the service
has no network or storage access.

This environment protects the confidentiality of a buyer's proprietary code, if
the execution happens only in the cloud and proprietary code is fetched in a
`BuyerFrontEnd` service.

#### Buyer’s key/value Service

A buyer's key/value service is a critical dependency for the bidding system.
The [FLEDGE key/value service][10] receives requests from the `BuyerFrontEnd`
service in this architecture (or directly from the client in case bidding and
auction runs locally on client's device). The service returns real-time buyer
data required for bidding, corresponding to lookup keys
(`bidding_signal_keys`).

The buyer’s key/value system may include other services running in a TEE. The
details of this system are out of scope of this document.

### Dependencies

Through techniques such as prefetching and caching, the following dependencies
are in the non-critical path of ad serving.

- A key management system required for FLEDGE service attestation. Learn more
  in the [FLEDGE Services public document][6].
- A [K-anonymity service][11] ensures the ad is served to K unique users. This
  service is a dependency for the `BuyerFrontEnd` service in this architecture.
  The details are not included in this document, but will be covered in a
  subsequent document.

## API

This API proposal for FLEDGE services is based on the gRPC framework.
[gRPC][12] is an open source, high performance RPC framework built
on top of HTTP2 that is used to build scalable and fast APIs. gRPC uses
[protocol buffers][13] as the [interface description
language][14] and underlying message interchange format.

FLEDGE services expose RPC API endpoints. In this document, the proposed API
definitions use [proto3][15].

All communications between FLEDGE services are RPC and are encrypted. All
client-to-server communication is also encrypted. Refer to the [FLEDGE services
explainer][6] for more information.

A client, such as an Android app or web browser, can call FLEDGE services using
RPC or HTTPS. A proxy service hosted in the same VM instance as the FLEDGE
service converts HTTPS requests to RPC. Details of this service are out of scope
of this document.

Requests to FLEDGE services and corresponding responses are encrypted. Every
request includes an encrypted payload (`request_ciphertext`) and a raw key
version (`key_id`) which corresponds to the public key that is used to encrypt
the request. The service that decrypts the request will have to use private
keys(corresponding to the same key version) to decrypt the request.

### Public APIs

A client such as an Android app or web browser can access public APIs. Clients
can send RPC or HTTPS to FLEDGE service.

#### Protocol buffer <-> JSON / YAML

Given gRPC APIs are defined as protocol buffers, following are some options to
convert protocol buffers to JSON or YAML.

[Proto3][15] has a built-in JSON serializer and parser. These libraries have
multi-language support and can be used to convert protocol buffer message
objects to JSON and vice-versa.

- C++:
  - [json_util.h][16]
    - `MessageToJsonString()`: Converts Protobuf message to JSON.
    - `JsonStringToMessage()`: Converts JSON to Protobuf message.
- Java:
  - [`JsonFormat.Printer`][17]: Converts Protobuf message to JSON format.
  - [`JsonFormat.Parser`][17]: Converts JSON to Protobuf message.

_Learn more about_ [_Proto3 to JSON mapping_][18].

YAML can also be used with HTTPS. The open source tool [gnostic][19] can
convert protocol buffers to and from YAML Open API descriptions.

Also, you may refer to the [protoc-gen-openapi plugin][20] to generate Open
API output corresponding to a proto definition.

#### SelectWinningAd

The seller front-end service (`SellerFrontEnd`) exposes an API endpoint
(`SelectWinningAd`). A client such as an Android app or web browser sends a
`SelectWinningAd` RPC or HTTPS request to `SellerFrontEnd`. After processing
the request, the `SelectWinningAdResponse` includes the winning ad for the
publisher ad slot that would render on the user's device.

_Note: The following API is designed for the desired end state, where sellers
 and buyers operate auction and bidding services (respectively) in the cloud.
 The `SelectWinningAd` API can be updated so that sellers (SSPs) can roll out
 services in the cloud independently. In such a scenario,
 `SellerFrontEndService` will not orchestrate bidding requests to buyers, but
 will still be able to execute auctions in the cloud. This setup is only
 recommended during the testing and early adoption phase_.

The following are custom client specific definitions that are required in
`AuctionConfig` that is passed in `SelectWinningAdRequest`. The corresponding
fields are copied to `AuctionCodeInput` that is passed in
[`ScoreAdsRequest`][21] for scoring ads.

```java
// Android specific Seller Input.
message AndroidSellerInput {
  // To be updated later if any custom fields are required to support Android.
}

// Web specific Seller Input.
message BrowserSellerInput {
  // This timeout can be specified to restrict the runtime (in milliseconds)
  // of the seller's scoreAd() script for Auction.
  int64 seller_timeout_ms = 1;

  // Optional. Component auction configuration can contain additional auction
  // configurations for each seller's "component auction".
  google.protobuf.Struct component_auctions = 2;

  // An JSON object constructed by the browser, containing information that the
  // browser knows and that the seller's auction code might want to verify.
  google.protobuf.Struct browser_signals = 3;
}
```

Following is the `SelectWinningAd` API definition.

```java
syntax = "proto3";

service SellerFrontEnd {
  // Selects a winning ad for the Publisher ad slot that would be
  // rendered on the user's device.
  rpc SelectWinningAd(SelectWinningAdRequest) returns (SelectWinningAdResponse) {}
}

// SelectWinningAdRequest is sent from user's device to the
// Seller FrontEnd Service.
message SelectWinningAdRequest {

  // Raw Request.
  message SelectWinningAdRawRequest {
    enum ClientType {
      UNKNOWN = 0;

      // An Android device with Google Mobile Services (GMS).
      // Note: This covers apps on Android and browsers on Android.
      ANDROID = 1;

      // An Android device without Google Mobile Services (GMS).
      ANDROID_NON_GMS = 2;

      // Any Browser (e.g. Chrome).
      BROWSER = 3;
    }

    // AuctionConfig required by the Seller for Ad Auction.
    // The auction config includes contextual signals and other data required by
    // the Seller for auction. This also includes the url to fetch SSP's proprietary
    // Auction logic and keys to lookup in Key/Value Service. This config is
    // passed from client to Seller FrontEnd Service in the umbrella request
    // (SelectWinningAdRequest).
    message AuctionConfig {

      // Client specific Seller Input.
      oneof SellerInput {

        // Android specific Seller Input.
        AndroidSellerInput android_seller_input = 1;

        // Web specific Seller Input.
        BrowserSellerInput browser_seller_input = 2;
      }

      // Url for fetching seller-owned auction code.
      // For simplicity, this is one url. However, there could be
      // separate endpoints for JS and WASM.
      string decision_logic_url = 3;
      // Information about the Buyers / DSPs that participate in the auction.
      repeated string custom_audience_buyer = 4;

      /*..............  Contextual Signals.........................*/
      // Contextual Signals refer to auction_signals, seller_signals and
      // per_buyer_signals, derived contextually.
      // Information about Auction (ad format, size). This information
      // is available both to the Seller and all Buyers participating in
      // auction.

      // Represents a JSON object.
      google.protobuf.Struct auction_signals = 5;

      // Seller specific signals that include information about the context
      // (e.g. Category blocks Publisher has chosen and so on). This can
      // not be fetched real-time from Key-Value Server.

      // Represents a JSON object.
      google.protobuf.Struct seller_signals = 6;
    }

    // Ad request timestamp.
    // Note: The precision will be limited to preserve privacy.
    google.protobuf.Timestamp timestamp = 1;

    // Optional. Required by Android to identify an ad selection request. This
    // is going to be cached on the server side to help with reporting.
    // Note: The precision will be limited to preserve privacy.
    int64 ad_selection_request_id = 2;

    // Encrypted BuyerInput per Buyer.
    // The key in the map corresponds to Buyer's (DSP) enrollment ID with
    // Privacy Sandbox.
    // The value corresponds to BuyerInput ciphertext that will be ingested by
    // the Buyer for bidding. BuyerInput include information about Custom
    // Audience / Interest Group including information about ads associated with
    // the audience list.
    map<string, bytes> encrypted_input_per_buyer = 3;

    // Includes configuration data required in Remarketing Ad
    // Auction. Refer below for AuctionConfig definition.
    AuctionConfig auction_config = 4;

    // Type of end user's device / client, that would help in validating the
    // integrity of an attested client.
    // Note: Not all types of clients will be attested.
    ClientType client_type = 5;

    // Field representing client attestation data will be added later.
  }

  // Encrypted SelectWinningAdRawRequest.
  bytes request_ciphertext = 1;

  // Version of the Public Key used for request encryption. The service
  // needs use Private Keys corresponding to same key_id to decrypt
  // 'request_ciphertext'.
  string key_id = 2;
}
message SelectWinningAdResponse {
  message SelectWinningAdRawResponse {
    // Custom Audience / Interest Group name.
    string custom_audience_name = 1;

    // The ad that will be rendered on the end user's device. This url is validated
    // on the client side to ensure that it actually belongs to Custom Audience
    // or Interest Group.
    string ad_render_url = 2;

    // Bid for the winning ad candidate, generated by a Buyer participating in
    // the auction.
    double bid_price = 3;

    // Score of the winning ad.
    double score = 4;

    // The version of the binary running on the server and Guest OS of Secure
    // Virtual Machine. The client may validate that with the version
    // available in open source repo.
    string server_binary_version = 5;
  }
// Ciphertext corresponding to SelectWinningAdRawResponse.
bytes response_ciphertext = 1;
}
```

##### BuyerInput

Encrypted `BuyerInput` data corresponding to a buyer.

Encrypted `BuyerInput` data corresponding to each buyer participating in the
auction, is passed in the umbrella request (`SelectWinningAdRequest`) from
the client to the `SellerFrontEnd` service. The `BuyerInput` is encrypted in
the client and decrypted in the `BuyerFrontEnd` service operated by the
buyer.

Following are custom client-specific definitions that are required for
`BuyerInput`. The corresponding fields are copied to `AuctionCodeInput` that
is passed in [`ScoreAdsRequest`][21] for scoring ads.

```java
syntax = "proto3";

// User Signals required by the Buyer for Bidding, specific to advertising
// on Android apps.

message AndroidAppUserSignal {
  // To be updated at a later date.
}

// User signals required for web advertising on desktop or mobile.
message BrowserUserSignal {
  // To be updated at a later date.
}

// Input to Buyer / DSP for advertising on Android App.
message AndroidBuyerInput {
  // To be updated later if any custom fields are required to support Android.
}

// Input to Buyer / DSP for advertising on browsers.
message BrowserBuyerInput {

  // This timeout can be specified to restrict the runtime (in milliseconds)
  // of the Buyer's generateBid() scripts for bidding. This can also be a
  // default value if timeout is unspecified for the Buyer.
  int64 buyer_timeout_ms =  1;

  // This can be specified to limit the number of Interest Groups / Custom
  // Audiences from a particular Buyer that participates in the auction. This
  // can also be a default value if the limit is unspecified for the Buyer.
  int64 buyer_group_limits = 2;
}
```

Following is the `BuyerInput` API definition.

```java
syntax = "proto3";

// A BuyerInput includes data that a Buyer (DSP) requires to generate bids.
message BuyerInput {

// CustomAudience (InterestGroup) includes the endpoint from where
// AdTech's proprietary code for bidding will be fetched and the set of ads
// corresponding to this audience. Each Custom Audience has a name that is
// unique for a Buyer.
  message CustomAudience {
    // Name or tag of Custom Audience / Interest Group.
    string name = 1;

    // Url for fetching DSP bidding code.
    string bidding_logic_url = 2;

    // Optional. The bidder (Buyer) may provide computationally-expensive
    // subroutines in WebAssembly (WASM) that can be fetched using this url.
    string bidding_wasm_helper_url = 3;

    // Ad creatives belonging to this CustomAudience / InterestGroup.
    repeated string ad_render_url = 4;

    // Optional.This field may be populated for Browser but not required for
    // Android at this point.
    //
    // This field contains the various ad components (or "products")
    // that can be used to construct Ads Composed of Multiple Pieces.
    // Each entry is an object that includes both a rendering URL and arbitrary
    // metadata that can be used at bidding time.
    repeated string ad_component = 5;

    // Optional. This field may be set for Browser but not required for Android
    // at this point.
    //
    // The priority is used to select which Interest Groups / Custom Audiences
    // participate in an auction when the number of Interest Groups are limited
    // by the buyer_group_limits in BrowserBuyerInput. If not specified, the
    // default value of 0.0 is assigned.
    //
    // These values are only used to select Interest Groups to participate in
    // an auction such that if there is an Interest Group participating in the
    // auction with priority x, all interest groups corresponding to the same
    // buyer having a priority y where y > x should be considered for generating
    // bids. In case due to buyer_group_limits if all Interest Groups with same
    // priority can not participate, then Interest Groups will be uniformly
    // randomly chosen from the set of interest groups with that priority.
    float priority = 6;
  }
  // The Custom Audiences (a.k.a Interest Groups) corresponding to the
  // DSP / Buyer.
  repeated CustomAudience custom_audience = 1;

  // The keys to fetch from Buyer Key Value Service.
  repeated string bidding_signal_keys = 2;

  // Android or Web specific user signal.
  oneof UserSignal {
    AndroidAppUserSignal android_app_user_signal = 3;
    BrowserUserSignal browser_user_signal = 4;
  }

  // Information the Sell Side Platform running Auction.
  // Represents a JSON object.
  google.protobuf.Struct seller_signals = 5;

  // Buyer may provide additional contextual information that
  // could help in generating bids. Not fetched real-time.
  // Represents a JSON object.
  google.protobuf.Struct buyer_signals = 6;

  // Custom Buyer Input for app or web advertising.
  oneof CustomBuyerInput {
   AndroidBuyerInput android_buyer_input = 7;
   BrowserBuyerInput browser_buyer_input = 8;
  }

  // Client (Android / Browser) stable identifier required for K-Anonymity
  // check.
  // Note: The BiddingFrontEnd would instrument K-Anonymity checks.
  string k_anon_member_id = 9;
}
```

#### GetBid

The `BuyerFrontEnd` service exposes an API endpoint `GetBid`. The
`SellerFrontEnd` service sends `GetBidRequest` to the `BuyerFrontEnd` service
with encrypted `BuyerInput` and other data. After processing the request,
`BuyerFrontEnd` returns `GetBidResponse`, which includes a bid and other data
corresponding to the top eligible ad candidate. Refer to [`AdWithBid`][22]
for more information.

The communication between the `BuyerFrontEnd` service and the `SellerFrontEnd`
service is between each service’s TEE and is end-to-end encrypted.

_Note: Temporarily, as adtechs test these systems, clients can call
 `BuyerFrontEnd` services directly using the API below_.

```java
syntax = "proto3";

// Buyer’s front-end service
service `BuyerFrontEnd` {
  // Returns bid for the top eligible ad candidate.
  rpc GetBid(GetBidRequest) returns (GetBidResponse) {}
}

// GetBidRequest is sent by the `SellerFrontEnd` Service to the
// `BuyerFrontEnd` Service.
message GetBidRequest{
  message GetBidRawRequest {
    // Encrypted BuyerInput corresponding to the Buyer.
    // This includes CustomAudiences (InterestGroups) owned by
    // the Buyer and signals required for filtering ads and generating bids.
    bytes buyer_input_ciphertext = 1;

    // Whether this is a fake request from SellerFrontEnd Service
    // and should be dropped.
    // Note: Seller FrontEnd Service may send chaffs to a few other Buyers
    // not participating in the auction. This is required for privacy reasons
    // to prevent Seller from figuring the Buyers by observing the network
    // traffic to `BuyerFrontEnd` Services, outside of TEE.
    bool is_chaff = 2;
  }
  // Encrypted BiddingRequest.
  bytes request_ciphertext = 1;

  // Version of the Public Key used for request encryption. The service
  // needs use Private Keys corresponding to same key_id to decrypt
  // 'request_ciphertext'.
  string key_id = 2;
}

// Response to GetBidRequest.
message GetBidResponse {
  // Unencrypted response.
  message GetBidRawResponse {
    // Includes ad_render_url and corresponding bid value pairs.
    // Represents a JSON object.
    AdWithBid bid = 1;
  }
  // Ciphertext corresponding to GetBidRawResponse.
  bytes response_ciphertext = 1;
}
```

##### AdWithBid

The bid for an ad candidate, includes `ad_render_url, ad_metadata` and
corresponding `bid_price`. This is returned in [`GetBidResponse`][23].

```java
syntax = "proto3";

// Bid for an ad candidate.
message AdWithBid {
  // Identifies an ad creative.
string ad_render_url = 1;
// Metadata of the ad, this will be passed to Seller's scoring function.
google.protobuf.Struct ad_metadata = 2;
// Name of the Custom Audience / Interest Group this ad belongs to.
string custom_audience_name = 3;
// Bid price corresponding to an ad.
double bid_price = 4;
}
```

### Internal APIs

Internal APIs refer to the interface for communication between FLEDGE services
within a SSP system or DSP system.

#### GenerateBids

The `Bidding` service exposes an API endpoint `GenerateBids`. The
`BuyerFrontEnd` service sends `GenerateBidsRequest` to the `Bidding` service,
which includes the buyer's proprietary bidding code and required input. After
processing the request, the bidding service returns the
`GenerateBidsResponse` which includes bids that correspond to each ad
(`AdWithBid`).

The communication between the `BuyerFrontEnd` service and `Bidding` service
occurs between each service’s TEE and request-response is end-to-end
encrypted.

```java
syntax = "proto3";

// Call the bidding service.
service Bidding {
  // Generate Bids for ads in Custom Audiences (a.k.a InterestGroups) and
  // filters ads.
  rpc GenerateBids(GenerateBidsRequest) returns (GenerateBidsResponse) {}
}

// Generate bids for all Custom Audiences / InterestGroups corresponding
// to the Buyer / DSP.
message GenerateBidsRequest {
  message GenerateBidsRawRequest {
    message BiddingCodeInput {
     // UserSignals required for Bidding.
     oneof UserSignal {
       AndroidAppUserSignal android_app_user_signal = 1;
       BrowserUserSignal browser_user_signal = 2;
     }

     // Unique string that identifies the Custom Audience / Interest Group for a
     // Buyer.
     string name = 3;

     // Ad creative render urls belonging to the Custom Audience / Interest Group.
     repeated string ad_render_url = 4;

     // Optional. This field may be populated for Browser but not required for
     // Android at this point.
     //
     // This field contains the various ad components (or "products") that can be
     // used to construct Ads Composed of Multiple Pieces.
     //
     // Each entry is an object that includes both a rendering URL and arbitrary
     // metadata that can be used at bidding time.
     repeated string ad_component = 5;

     // Optional. This field may be set for Browser but not required for Android
     // at this point.
     //
     // The priority is used to select which Interest Groups / Custom Audiences
     // participate in an auction when the number of Interest Groups are limited
     // by the buyer_group_limits in BrowserBuyerInput. If not specified, the
     // default value of 0.0 is assigned.
     //
     // These values are only used to select Interest Groups to participate in
     // an auction such that if there is an Interest Group participating in the
     // auction with priority x, all interest groups corresponding to the same
     // buyer having a priority y where y > x should be considered for generating
     // bids. In case due to buyer_group_limits if all Interest Groups with same
     // priority can not participate, then Interest Groups will be uniformly
     // randomly chosen from the set of interest groups with that priority.
     float priority = 6;

     // Information about Sell Side Platform conducting Ad Auction.
     // Represents a JSON object.
     //
     // Note: This is passed in encrypted BuyerInput, i.e.
     // buyer_input_ciphertext field in GetBidRequest. The BuyerInput is
     // encrypted in the client and decrypted in `BuyerFrontEnd` Service.
     // This data is copied from BuyerInput.
     google.protobuf.Struct seller_signals = 7;

     // Optional. Buyer may provide additional contextual information that
     // could help in generating bids. Not fetched real-time.
     // Represents a JSON object.
     //
     // Note: This is passed in encrypted BuyerInput, i.e.
     // buyer_input_ciphertext field in GetBidRequest. The BuyerInput is
     // encrypted in the client and decrypted in `BuyerFrontEnd` Service.
     // This data is copied from BuyerInput.
     google.protobuf.Struct buyer_signals = 8;

     /*...Real Time signals fetched from buyer’s Key/Value service...*/
     // Key-value pairs corresponding to keys in bidding_signal_keys.
     // Represents a JSON object.
     google.protobuf.Struct bidding_signals = 9;

      // Custom Buyer Input for app or web advertising.
      oneof CustomBuyerInput {
        AndroidBuyerInput android_buyer_input = 10;
        BrowserBuyerInput browser_buyer_input = 11;
      }
      // Any other Buyside Key Value signals will be updated at a later date.
    }

    // Buyer logic per Custom Audience / Interest Group.
    message BuyerCodePerAudience {
      // Note: The code runtime engine can accept JavaScript, WASM code or
      // both.
      //
      // Buyer owned JavaScript for Bidding. The JavaScript may include
      // embedded WASM binary code, no file access would be allowed. Refer here
      // for an example WASM instantiation.
      bytes js_code = 1;

      // Optional. AdTech owned WASM code for Bidding.
      bytes wasm_code = 2;

      // Inputs for Bidding code.
      BiddingCodeInput bidding_code_input = 3;
    }
    // Buyer Logic per Custom Audience / Interest Group.
    repeated BuyerCodePerAudience buyer_code_per_audience = 1;
  }
  // Encrypted GenerateBidsRequest.
  bytes request_ciphertext = 1;

  // Version of the Public Key used for request encryption. The service
  // needs use Private Keys corresponding to same key_id to decrypt
  // 'request_ciphertext'.
  string key_id = 2;
}
// Encrypted response to GenerateBidsRequest with bid prices corresponding
// to all eligible Ad creatives.
message GenerateBidsResponse {
  message GenerateBidsRawResponse {
    // Bids corresponding to filtered ads during bidding.
    repeated AdWithBid bid = 1;
  }
  // Ciphertext corresponding to GenerateBidsRawResponse.
  bytes response_ciphertext = 1;
}
```

#### ScoreAds

The `Auction` service exposes an API endpoint `ScoreAds`. The `SellerFrontEnd`
service sends a `ScoreAdsRequest` to the `Auction` service with the SSP's
proprietary auction code and inputs required by the code. The inputs include
bids from each buyer and other required signals. After processing the
request, the `Auction` service returns the `ScoreAdsResponse` that includes
scores corresponding to each ad.

The communication between the `SellerFrontEnd` service and `Auction` service
occurs within each service’s TEE and request-response is end-to-end
encrypted.

```java
syntax = "proto3";

// Call the `auction` service.
service Auction {
  // Scores all top ad candidates returned by each Buyer (Bidder)
  // participating in the auction.
  rpc ScoreAds(ScoreAdsRequest) returns (ScoreAdsResponse) {}
}

// Scores top Ad candidates of each Buyer.
message ScoreAdsRequest {
  message ScoreAdsRawRequest {
    // Inputs to JavaScript Auction Code module.
    message AuctionCodeInput {

    // Includes ad, bid corresponding to the ad, the Custom Audience
    // Interest Group name the ad belongs to.
    // The ad-bid pair will be converted to a JSON object before passing as an
    // input to ScoreAd function.
    // Note: Every ad is scored in a different thread in an isolated Sandbox
    // within the TEE.
    repeated AdWithBid buyer_bids = 1;

    // Real-time signals fetched from Seller Trusted Key Value Service.
    // Represents a JSON object.
    // Note: The keys used to look up scoring signals are ad_render_urls and
    // ad_component_render_urls that are part of the bids returned by Buyers
    // participating in the auction.
    google.protobuf.Struct scoring_signals = 2;

    /*..............  Contextual Signals.........................*/
    // Contextual Signals refer to auction_signals, seller_signals and
    // per_buyer_signals, derived contextually.
    // Note: The following are copied from AuctionConfig that is passed from
    // the client in SelectWinningAdRequest to SellerFrontEnd.

    // Information about Auction (ad format, size). This information
    // is available both to the Seller and all Buyers participating in
    // auction.
    // Represents a JSON object.
    //
    // Note: This is passed by client in AuctionConfig in SelectWinningAdRequest
    // to SellerFrontEnd Service. This data is copied from AuctionConfig.
    google.protobuf.Struct auction_signals = 3;

    // Seller specific signals that include information about the context
    // (e.g. Category blocks Publisher has chosen and so on). This can
    // not be fetched real-time from Key-Value Server.
    // Represents a JSON object.
    //
    // Note: This is passed by client in AuctionConfig in SelectWinningAdRequest
    // to SellerFrontEnd Service. This data is copied from AuctionConfig.
    google.protobuf.Struct seller_signals = 4;

    // Contextually derived (Contextual request dependent) per Buyer
    // signals.
    // Buyer Id to Buyer Signals pair. Represents a JSON object.
    //
    // Note: This is passed by client in AuctionConfig in SelectWinningAdRequest
    // to SellerFrontEnd Service. This data is copied from AuctionConfig.
    google.protobuf.Struct per_buyer_signals = 5;

    // Client specific Seller Input.
    oneof SellerInput {
      // Android specific Seller Input.
      AndroidSellerInput android_seller_input = 6;

      // Web specific Seller Input.
      BrowserSellerInput browser_seller_input = 7;
    }

    // Note: The code runtime engine can accept JavaScript, WASM code or
    // both.
    // Seller owned JavaScript for Auction. The JavaScript may include
    // embedded WASM.
    bytes js_code = 1;

    // Optional; Seller owned WASM code for Auction.
    // Note: The WASM should be initialized in the JavaScript and local file
    // access is prohibited with the Sandbox where adtech proprietary code would
    // execute.
    bytes wasm_code = 2;

    // Inputs for Auction code.
    AuctionCodeInput auction_code_input = 3;
  }

  // Encrypted ScoreAdsRawRequest.
  bytes request_ciphertext = 1;

  // Version of the Public Key used for request encryption. The service
  // needs use Private Keys corresponding to same key_id to decrypt
  // 'request_ciphertext'.
  bytes key_id = 2;
}

// Encrypted response that includes auction results returned by adtech's
// auction code.

message ScoreAdsResponse {
  // Identifies a scored ad belonging to a Custom Audience / Interest Group.
  message AdScore {

    // Ad creative render url.
    string ad_render_url = 1;

    // Score of the ad determined during the auction. Any value that is zero or
    // negative indicates that the ad cannot win the auction. The winner of the
    // auction would be the ad that was given the highest score.
    double score = 2;

    // Name of Custom Audience / Interest Group the ad belongs to.
    string custom_audience_name = 3;

    /***************** Only relevant to Component Auctions *******************/
    // Optional for Android, required for Web in case of component auctions.
    // If the bid being scored is from a component auction and this value is not
    // true, the bid is ignored. If not present, this value is considered false.
    // This field must be present and true both when the component seller scores
    // a bid, and when that bid is being scored by the top-level auction.
    bool allow_component_auction = 4;

    // Optional for Android, required for Web in case of component auctions.
    // Modified bid value to provide to the top-level seller script. If
    // present, this will be passed to the top-level seller's scoring function
    // instead of the original bid, if the ad wins the component auction and
    // top-level auction respectively.
    double bid = 5;
  }
  // Unencrypted response.
  message ScoreAdsRawResponse {
    // Scores of ads participating in the auction.
    repeated AdScore ad_score = 1;
  }
  // Ciphertext corresponding to ScoreAdsRawResponse.
  bytes response_ciphertext = 1;
}
```

[4]: https://privacysandbox.com
[5]: https://developer.chrome.com/docs/privacy-sandbox/fledge/
[6]: https://github.com/privacysandbox/fledge-docs/trusted_services_overview.md
[7]: https://github.com/WICG/turtledove/blob/main/FLEDGE.md
[8]: https://github.com/microsoft/PARAKEET
[9]: #selectwinningad
[10]: https://github.com/WICG/turtledove/blob/main/FLEDGE_Key_Value_Server_API.md
[11]: https://github.com/WICG/turtledove/blob/main/FLEDGE_k_anonymity_server.md
[12]: https://grpc.io
[13]: https://developers.google.com/protocol-buffers
[14]: https://en.wikipedia.org/wiki/Interface_description_language
[15]: https://developers.google.com/protocol-buffers/docs/proto3
[16]: https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.util.json_util
[17]: https://developers.google.com/protocol-buffers/docs/reference/java/com/google/protobuf/util/JsonFormat
[18]: https://developers.google.com/protocol-buffers/docs/proto3#json
[19]: https://github.com/google/gnostic
[20]: https://github.com/google/gnostic/tree/main/cmd/protoc-gen-openapi
[21]: #scoreads
[22]: #adwithbid
[23]: #getbid
