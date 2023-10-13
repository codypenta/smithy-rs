# RFC: User-configurable retry classification

> Status: Implemented
>
> Applies to: client

For a summarized list of proposed changes, see the [Changes Checklist](#changes-checklist) section.

This RFC defines the user experience and implementation of user-configurable
retry classification. Custom retry classifiers enable users to change what
responses are retried while still allowing them to rely on defaults set by SDK
authors when desired.
## Terminology

- **Smithy Service**: An HTTP service, whose API is modeled with the [Smithy
  IDL](https://www.smithy.io).
- **Smithy Client**: An HTTP client generated by smithy-rs from a `.smithy`
  model file.
- **AWS SDK**: A **smithy client** that's specifically configured to work with
  an AWS service.
- **Operation**: A modeled interaction with a service, defining the proper input
  and expected output shapes, as well as important metadata related to request
  construction. "Sending" an operation implies sending one or more HTTP requests
  to a **Smithy service**, and then receiving an output or error in response.
- **Orchestrator**: The client code which manages the request/response pipeline.
  The orchestrator is responsible for:
    - Constructing, serializing, and sending requests.
    - Receiving, deserializing, and (optionally) retrying requests.
    - Running interceptors *(not covered in this RFC)* and handling errors.
- **Runtime Component**: A part of the orchestrator responsible for a specific
  function. Runtime components are used by the orchestrator itself, may depend
  on specific configuration, and must not be changed by interceptors. Examples
  include the endpoint resolver, retry strategy, and request signer.
- **Runtime Plugin**: Code responsible for setting and **runtime components**
  and related configuration. Runtime plugins defined by codegen are responsible
  for setting default configuration and altering the behavior of **Smithy
  clients** including the **AWS SDKs**.

## How the orchestrator should model retries

A **Retry Strategy** is the process by which the orchestrator determines when
and how to retry failed requests. Only one retry strategy may be set at any
given time. During its operation, the retry strategy relies on a series of
**Retry Classifiers** to determine if and how a failed request should be
retried. Retry classifiers each have a **Retry Classifier Priority** so that
regardless of whether they are set during config or operation construction,
they'll always run in a consistent order.

Classifiers are each run in turn by the retry strategy:

```rust,ignore
pub fn run_classifiers_on_ctx(
    classifiers: impl Iterator<Item = SharedRetryClassifier>,
    ctx: &InterceptorContext,
) -> RetryAction {
    // By default, don't retry
    let mut result = RetryAction::NoActionIndicated;

    for classifier in classifiers {
        let new_result = classifier.classify_retry(ctx);

        // If the result is `NoActionIndicated`, continue to the next classifier
        // without overriding any previously-set result.
        if new_result == RetryAction::NoActionIndicated {
            continue;
        }

        // Otherwise, set the result to the new result.
        tracing::trace!(
            "Classifier '{}' set the result of classification to '{}'",
            classifier.name(),
            new_result
        );
        result = new_result;

        // If the result is `RetryForbidden`, stop running classifiers.
        if result == RetryAction::RetryForbidden {
            tracing::trace!("retry classification ending early because a `RetryAction::RetryForbidden` was emitted",);
            break;
        }
    }

    result
}
```

*NOTE: User-defined retry strategies are responsible for calling `run_classifiers_on_ctx`.*

Lower-priority classifiers run first, but the retry actions they return may be
overridden by higher-priority classifiers. Classification stops immediately if
any classifier returns `RetryAction::RetryForbidden`.

## The user experience if this RFC is implemented

In the current version of the SDK, users are unable to configure retry
classification, except by defining a custom retry strategy. Once this RFC is
implemented, users will be able to define and set their own classifiers.

### Defining a custom classifier

```rust,ignore
#[derive(Debug)]
struct CustomRetryClassifier;

impl ClassifyRetry for CustomRetryClassifier {
    fn classify_retry(
        &self,
        ctx: &InterceptorContext,
    ) -> Option<RetryAction> {
        // Check for a result
        let output_or_error = ctx.output_or_error();
        // Check for an error
        let error = match output_or_error {
            // Typically, when the response is OK or unset
            // then `RetryAction::NoActionIndicated` is returned.
            Some(Ok(_)) | None => return RetryAction::NoActionIndicated,
            Some(Err(err)) => err,
        };

        todo!("inspect the error to determine if a retry attempt should be made.")
    }

    fn name(&self) -> &'static str { "my custom retry classifier" }

    fn priority(&self) -> RetryClassifierPriority {
        RetryClassifierPriority::default()
    }
}
```

#### Choosing a retry classifier priority

Sticking with the default priority is often the best choice. Classifiers should
restrict the number of cases they can handle in order to avoid having to compete
with other classifiers. When two classifiers would classify a response in two
different ways, the priority system gives us the ability to decide which
classifier should be respected.

Internally, priority is implemented with a simple numeric system. In order to
give the smithy-rs team the flexibility to make future changes, this numeric
system is private and inaccessible to users. Instead, users may set the priority
of classifiers relative to one another with the `with_lower_priority_than` and
`with_higher_priority_than` methods:

```rust,ignore
impl RetryClassifierPriority {
    /// Create a new `RetryClassifierPriority` with lower priority than the given priority.
    pub fn with_lower_priority_than(other: Self) -> Self { ... }

    /// Create a new `RetryClassifierPriority` with higher priority than the given priority.
    pub fn with_higher_priority_than(other: Self) -> Self { ... }
}
```

For example, if it was important for our `CustomRetryClassifier` in the previous
example to run *before* the default `HttpStatusCodeClassifier`, a user would
define the `CustomRetryClassifier` priority like this:

```rust,ignore
impl ClassifyRetry for CustomRetryClassifier {
    fn priority(&self) -> RetryClassifierPriority {
        RetryClassifierPriority::run_before(RetryClassifierPriority::http_status_code_classifier())
    }
}
```

The priorities of the three default retry classifiers
(`HttpStatusCodeClassifier`, `ModeledAsRetryableClassifier`, and
`TransientErrorClassifier`) are all public for this purpose. Users may **ONLY**
set a retry priority relative to an existing retry priority.


#### `RetryAction` and `RetryReason`

Retry classifiers communicate to the retry strategy by emitting `RetryAction`s:

```rust,ignore
/// The result of running a [`ClassifyRetry`] on a [`InterceptorContext`].
#[non_exhaustive]
#[derive(Clone, Eq, PartialEq, Debug, Default)]
pub enum RetryAction {
    /// When a classifier can't run or has no opinion, this action is returned.
    ///
    /// For example, if a classifier requires a parsed response and response parsing failed,
    /// this action is returned. If all classifiers return this action, no retry should be
    /// attempted.
    #[default]
    NoActionIndicated,
    /// When a classifier runs and thinks a response should be retried, this action is returned.
    RetryIndicated(RetryReason),
    /// When a classifier runs and decides a response must not be retried, this action is returned.
    ///
    /// This action stops retry classification immediately, skipping any following classifiers.
    RetryForbidden,
}
```

When a retry is indicated by a classifier, the action will contain a `RetryReason`:

```rust,ignore
/// The reason for a retry.
#[non_exhaustive]
#[derive(Clone, Eq, PartialEq, Debug)]
pub enum RetryReason {
    /// When an error is received that should be retried, this reason is returned.
    RetryableError {
        /// The kind of error.
        kind: ErrorKind,
        /// A server may tell us to retry only after a specific time has elapsed.
        retry_after: Option<Duration>,
    },
}
```

*NOTE: `RetryReason` currently only has a single variant, but it's defined as an `enum` for [forward compatibility] purposes.*

`RetryAction`'s `impl` defines several convenience methods:

```rust,ignore
impl RetryAction {
    /// Create a new `RetryAction` indicating that a retry is necessary.
    pub fn retryable_error(kind: ErrorKind) -> Self {
        Self::RetryIndicated(RetryReason::RetryableError {
            kind,
            retry_after: None,
        })
    }

    /// Create a new `RetryAction` indicating that a retry is necessary after an explicit delay.
    pub fn retryable_error_with_explicit_delay(kind: ErrorKind, retry_after: Duration) -> Self {
        Self::RetryIndicated(RetryReason::RetryableError {
            kind,
            retry_after: Some(retry_after),
        })
    }

    /// Create a new `RetryAction` indicating that a retry is necessary because of a transient error.
    pub fn transient_error() -> Self {
        Self::retryable_error(ErrorKind::TransientError)
    }

    /// Create a new `RetryAction` indicating that a retry is necessary because of a throttling error.
    pub fn throttling_error() -> Self {
        Self::retryable_error(ErrorKind::ThrottlingError)
    }

    /// Create a new `RetryAction` indicating that a retry is necessary because of a server error.
    pub fn server_error() -> Self {
        Self::retryable_error(ErrorKind::ServerError)
    }

    /// Create a new `RetryAction` indicating that a retry is necessary because of a client error.
    pub fn client_error() -> Self {
        Self::retryable_error(ErrorKind::ClientError)
    }
}
```

### Setting classifiers

The interface for setting classifiers is very similar to the interface of
settings interceptors:

```rust,ignore
// All service configs support these setters. Operations support a nearly identical API.
impl ServiceConfigBuilder {
    /// Add type implementing ClassifyRetry that will be used by the RetryStrategy
    /// to determine what responses should be retried.
    ///
    /// A retry classifier configured by this method will run according to its priority.
    pub fn retry_classifier(mut self, retry_classifier: impl ClassifyRetry + 'static) -> Self {
        self.push_retry_classifier(SharedRetryClassifier::new(retry_classifier));
        self
    }

    /// Add a SharedRetryClassifier that will be used by the RetryStrategy to
    /// determine what responses should be retried.
    ///
    /// A retry classifier configured by this method will run according to its priority.
    pub fn push_retry_classifier(&mut self, retry_classifier: SharedRetryClassifier) -> &mut Self {
        self.runtime_components.push_retry_classifier(retry_classifier);
        self
    }

    /// Set SharedRetryClassifiers for the builder, replacing any that were
    /// previously set.
    pub fn set_retry_classifiers(&mut self, retry_classifiers: impl IntoIterator<Item = SharedRetryClassifier>) -> &mut Self {
        self.runtime_components.set_retry_classifiers(retry_classifiers.into_iter());
        self
    }
}
```

### Default classifiers

Smithy clients have three classifiers enabled by default:

- `ModeledAsRetryableClassifier`: Checks for errors that are marked as retryable
  in the smithy model. If one is encountered, returns
  `RetryAction::RetryIndicated`. Requires a parsed response.
- `TransientErrorClassifier`: Checks for timeout, IO, and connector errors. If
  one is encountered, returns `RetryAction::RetryIndicated`.  Requires a parsed
  response.
- `HttpStatusCodeClassifier`: Checks the HTTP response's status code. By
  default, this classifies `500`, `502`, `503`, and `504` errors as
  `RetryAction::RetryIndicated`.  The list of retryable status codes may be
  customized when creating this classifier with the
  `HttpStatusCodeClassifier::new_from_codes` method.

AWS clients enable the three smithy classifiers as well as one more by default:

- `AwsErrorCodeClassifier`: Checks for errors with AWS error codes marking them
  as either transient or throttling errors. If one is encountered, returns
  `RetryAction::RetryIndicated`. Requires a parsed response. This classifier
  will also check the HTTP response for an `x-amz-retry-after` header. If one is
  set, then the returned `RetryAction` will include the explicit delay.

The priority order of these classifiers is as follows:

1. *(highest priority)* `TransientErrorClassifier`
2. `ModeledAsRetryableClassifier`
3. `AwsErrorCodeClassifier`
4. *(lowest priority)* `HttpStatusCodeClassifier`

The priority order of the default classifiers is not configurable. However, it's
possible to wrap a default classifier in a newtype and set your desired priority
when implementing the `ClassifyRetry` trait, delegating the `classify_retry` and
`name` fields to the inner classifier.

#### Disable default classifiers

Disabling the default classifiers is possible, but not easy. They are set at
different points during config and operation construction, and must be unset at
each of those places. A far simpler solution is to implement your own classifier
that has the highest priority.

Still, if completely removing the other classifiers is desired, use the
`set_retry_classifiers` method on the config to replace the config-level
defaults and then set a config override on the operation that does the same.

## How to actually implement this RFC

In order to implement this feature, we must:
- Update the current retry classification system so that individual classifiers
  as well as collections of classifiers can be easily composed together.
- Create two new configuration mechanisms for users that allow them to customize
  retry classification at the service level and at the operation level.
- Update retry classifiers so that they may 'short-circuit' the chain, ending
  retry classification immediately.

### The `RetryClassifier` trait

```rust,ignore
/// The result of running a [`ClassifyRetry`] on a [`InterceptorContext`].
#[non_exhaustive]
#[derive(Clone, Eq, PartialEq, Debug)]
pub enum RetryAction {
    /// When an error is received that should be retried, this action is returned.
    Retry(ErrorKind),
    /// When the server tells us to retry after a specific time has elapsed, this action is returned.
    RetryAfter(Duration),
    /// When a response should not be retried, this action is returned.
    NoRetry,
}

/// Classifies what kind of retry is needed for a given [`InterceptorContext`].
pub trait ClassifyRetry: Send + Sync + fmt::Debug {
    /// Run this classifier on the [`InterceptorContext`] to determine if the previous request
    /// should be retried. If the classifier makes a decision, `Some(RetryAction)` is returned.
    /// Classifiers may also return `None`, signifying that they have no opinion of whether or
    /// not a request should be retried.
    fn classify_retry(
        &self,
        ctx: &InterceptorContext,
        preceding_action: Option<RetryAction>,
    ) -> Option<RetryAction>;

    /// The name of this retry classifier.
    ///
    /// Used for debugging purposes.
    fn name(&self) -> &'static str;

    /// The priority of this retry classifier. Classifiers with a higher priority will run before
    /// classifiers with a lower priority. Classifiers with equal priorities make no guarantees
    /// about which will run first.
    fn priority(&self) -> RetryClassifierPriority {
        RetryClassifierPriority::default()
    }
}
```

### Resolving the correct order of multiple retry classifiers

Because each classifier has a defined priority, and because
`RetryClassifierPriority` implements `PartialOrd` and `Ord`, the standard
library's [sort] method may be used to correctly arrange classifiers. The
`RuntimeComponents` struct is responsible for storing classifiers, so it's also
responsible for sorting them whenever a new classifier is added. Thus, when a
retry strategy fetches the list of classifiers, they'll already be in the
expected order.

## Questions and answers

- **Q:** Should retry classifiers be fallible?
  - **A:** I think no, because of the added complexity. If we make them fallible
    then we'll have to decide what happens when classifiers fail. Do we skip
    them or does classification end? The retry strategy is responsible for
    calling the classifiers, so it be responsible for deciding how to handle a
    classifier error. I don't foresee a use case where an error returned by a
    classifier would be interpreted either by classifiers following the failed
    classifier or the retry strategy.

## Changes checklist

- [x] Add retry classifiers field and setters to `RuntimeComponents` and `RuntimeComponentsBuilder`.
  - [x] Add unit tests ensuring that classifier priority is respected by `RuntimeComponents::retry_classifiers`, especially when multiple layers of config are in play.
- [x] Add codegen customization allowing users to set retry classifiers on service configs.
- [x] Add codegen for setting default classifiers at the service level.
  - [x] Add integration tests for setting classifiers at the service level.
- [x] Add codegen for settings default classifiers that require knowledge of operation error types at the operation level.
  - [x] Add integration tests for setting classifiers at the operation level.
- [x] Implement retry classifier priority.
  - [x] Add unit tests for retry classifier priority.
- [x] Update existing tests that would fail for lack of a retry classifier.

<!-- Links -->

[sort]: https://doc.rust-lang.org/stable/std/primitive.slice.html#method.sort
[forward compatibility]: https://en.wikipedia.org/wiki/Forward_compatibility