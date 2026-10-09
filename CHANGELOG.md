# Changelog

All notable changes to this project will be documented in this file.
## 0.9.8 - 2026-10-09
### Features
* Added `ErrorCode::InvalidArgument`, reserved for arguments rejected before they reach the core. `ErrorCode` is now `#[non_exhaustive]`: a `match` over it needs a wildcard arm, which also keeps future codes from breaking callers.
* Added `KameleoonError::into_message()`, which consumes the error and returns its message without copying it. `KameleoonError` now implements `std::error::Error`, `PartialEq` and `Eq`.
* [`set_legal_consent`](https://docs.kameleoon.com/developer-docs/sdks/web-sdks/rust-sdk#set_legal_consent) takes `Option<&mut dyn CookieAccessor>` instead of `Option<&mut impl CookieAccessor>`, so `None` no longer needs a type annotation. Call sites passing `Some(&mut accessor)` are unaffected; an `Option<&mut MyAccessor>` variable must be coerced (`.map(|c| c as &mut dyn CookieAccessor)`).
* Re-exported the builder error types (`KameleoonClientConfigBuilderError`, `RemoteVisitorDataFilterBuilderError`, `GeolocationBuilderError`) and `RemoteVisitorDataFilterBuilder`, and documented the public API (`#![warn(missing_docs)]`).
* Removed `KameleoonLogger::get_log_level()`; the log level is set through `set_log_level` and was never meant to be read back.

### Bug fixes
* [`KameleoonClientFactory::create_*`][create] now returns an `ErrorCode::Initialization` error when the site code is empty or when the client ID or client secret is empty, consistent with every other client-creation failure. The error message names the exact cause. The `SiteCodeIsEmpty` and `ConfigCredentialsInvalid` error codes previously used for these cases are no longer returned; they remain declared and are marked deprecated, so code naming them keeps compiling (with a deprecation warning).
* Creating a client with a configuration file path now fails with an `Initialization` error if the file cannot be read or parsed, instead of silently using the default configuration.
* [`get_datafile`](https://docs.kameleoon.com/developer-docs/sdks/web-sdks/rust-sdk#get_datafile) (and the deprecated `get_feature_keys`) now return an `Initialization` error until the client is ready, instead of an empty data file.
* Dropping the last `KameleoonClient` for a site code (after `forget`) no longer blocks the dropping thread while the SDK runtime shuts down, and is safe from inside an SDK event handler or callback.
* The deprecated [`on_datafile_update`](https://docs.kameleoon.com/developer-docs/sdks/web-sdks/rust-sdk#on_datafile_update) is now a shorthand for `set_event_handler` with `EventHandler::DataFileUpdate`. Its callback is thereby also protected against panics like every other event handler.
* Lowered the log level of the message emitted when the streaming server closes the real-time (SSE) connection from `ERROR` to `INFO`. The streaming server recycles connections periodically and the SDK reconnects automatically, so this is expected behavior. Connection failures (error responses, network errors) are still logged as `ERROR`.

[create]: https://docs.kameleoon.com/developer-docs/sdks/web-sdks/rust-sdk#create

## 0.9.7 - 2026-09-11
### Bug fixes
* Reduced the load on the Data API during its outages. After a failed tracking request, the SDK sends only one visitor's data per tracking request until a request succeeds, instead of retrying with a full-size (up to 2.5 MB) request on every tracking interval. Once the Data API is reachable again, the accumulated visitor data is sent in full-size requests as usual.


## 0.9.6 - 2026-08-28
### Features
* [`initialize`][initialize] now fail fast when the initial configuration fetch has failed: instead of waiting out the timeout, they immediately return the actual fetch error, such as a network or HTTP failure. A later successful fetch (for example, on the next polling cycle) still makes the client ready, and subsequent calls return successfully. The behavior of [`is_ready`][is_ready] is unchanged: it returns `true` only after a configuration has been successfully fetched, and never reverts afterwards.
* Added support for SDK event handlers through the new [`set_event_handler`][set_event_handler] API:
  - `EventType::DataFileUpdate` notifies when the SDK data file (configuration) is updated with [polling](https://docs.kameleoon.com/developer-docs/feature-experimentation/technical-reference/technical-considerations#polling-default) or [streaming](https://docs.kameleoon.com/developer-docs/feature-experimentation/technical-reference/technical-considerations#streaming-premium-option) modes.
  - `EventType::HttpRequest` notifies when SDK HTTP requests complete successfully or fail.
  - HTTP request events include the request type, HTTP status or failure details, and request duration.
  - Passing `None` — for example `EventHandler::HttpRequest(None)` — clears the handler for the selected event type.
* All event types are available from the new `kameleoon_client::events` module.
* The [`on_datafile_update`](https://docs.kameleoon.com/developer-docs/sdks/web-sdks/rust-sdk#on_datafile_update) method has been deprecated in favor of `set_event_handler` with the `EventType::DataFileUpdate` event type.

### Bug fixes
* Targeting conditions of an unknown type now evaluate to `false` instead of `true`. Previously an unsupported condition matched every visitor; now it matches none.

[initialize]: https://docs.kameleoon.com/developer-docs/sdks/web-sdks/rust-sdk#initialize
[is_ready]: https://docs.kameleoon.com/developer-docs/sdks/web-sdks/rust-sdk#is_ready
[set_event_handler]: https://docs.kameleoon.com/developer-docs/sdks/web-sdks/rust-sdk#set_event_handler

## 0.9.5 - 2026-06-19
### Patch Changes
* Dependency versions are now pinned to exact versions (`=`) instead of using compatible version ranges (`^`). Affected libraries:
  - [kameleoon-core][kameleoon-core]

[kameleoon-core]: https://crates.io/crates/kameleoon-core

## 0.9.5 - 2026-06-19 - 2026-06-17
### Patch Changes
* Dependency versions are now pinned to exact versions (`=`) instead of using compatible version ranges (`^`). Affected libraries:
  - [kameleoon-core][kameleoon-core]

[kameleoon-core]: https://crates.io/crates/kameleoon-core

## 0.9.4 - 2026-06-16
### Breaking Changes
* Replaced `String` return types with `Arc<str>` across several SDK Core APIs and data structures to improve performance and reduce unnecessary allocations.
  - Affected methods:
    - [`get_variation()`][get_variation]
    - [`get_variations()`][get_variations]
    - [`is_feature_active()`][is_feature_active]
    - [`get_datafile()`][get_datafile]
  - Affected types:
    - [`DataFile`][datafile]
    - [`Variation`][variation]
    - [`Variable`][variable]
    - [`Rule`][rule]
  - This change may require updates to consumer code that relies on `String`-specific APIs or type signatures.

[variation]: https://developers.kameleoon.com/feature-management-and-experimentation/web-sdks/rust-sdk/#variation
[variable]: https://developers.kameleoon.com/feature-management-and-experimentation/web-sdks/rust-sdk/#variable
[rule]: https://developers.kameleoon.com/feature-management-and-experimentation/web-sdks/rust-sdk/#rule

## 0.9.3 - 2026-05-21
### Features
* Added support for proxy hosts. You can provide a proxy via [`KameleoonClientConfig`][additional_config] or an external file.
* Updated evaluation and tracking logic to comply with GDPR requirements when consent is not given:
    - If behavior is **partially blocked**, the default variation will be returned.
    - If behavior is **completely blocked**, an exception will be thrown.
* Added the [`evaluate_audiences`][evaluate_audiences] method. This method iterates over all Audiences Explorer segments, evaluates each one, and tracks the segments for which the visitor is targeted using the [`TARGETINGSEGMENT`](https://developers.kameleoon.com/apis/data-api-rest/all-endpoints/post-visit-events/) event.
* Added support for variation simulation when feature flags are **inactive (OFF state)** across the following methods:
  - [`get_variation()`][get_variation]
  - [`get_varitions()`][get_variations]
  - [`is_feature_active()`][is_feature_active]

[evaluate_audiences]: https://developers.kameleoon.com/feature-management-and-experimentation/web-sdks/rust-sdk/#evaluate_audiences
[get_variation]: https://developers.kameleoon.com/feature-management-and-experimentation/web-sdks/rust-sdk/#get_variation
[get_variations]: https://developers.kameleoon.com/feature-management-and-experimentation/web-sdks/rust-sdk/#get_variations
[is_feature_active]: https://developers.kameleoon.com/feature-management-and-experimentation/web-sdks/rust-sdk/#is_feature_active
[get_datafile]: https://developers.kameleoon.com/feature-management-and-experimentation/web-sdks/rust-sdk/#get_datafile
[datafile]: https://developers.kameleoon.com/feature-management-and-experimentation/web-sdks/rust-sdk/#datafile
[additional_config]: https://developers.kameleoon.com/feature-management-and-experimentation/web-sdks/rust-sdk/#additional-configuration

## 0.9.2 - 2026-05-14
### Features
* Added a new `date_modified` property to the [`DataFile`](https://developers.kameleoon.com/feature-management-and-experimentation/web-sdks/rust-sdk/#datafile) object returned by the [`get_datafile`](https://developers.kameleoon.com/feature-management-and-experimentation/web-sdks/rust-sdk/#get_datafile) method.

## 0.9.1 - 2026-05-05
### Features
* Added support for **during the current visit** and **during any of the last visits** settings across the following targeting conditions:
  - Converted Goal
  - Feature Flag
  - Web Experiment
  - Personalization
  - Exclusive Campaign



## 0.9.0 - 2026-04-24
### Features
* Initial release of the `kameleoon-client` crate
