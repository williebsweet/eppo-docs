import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# PHP

Eppo's open source PHP SDK can be used for both feature flagging and experiment assignment:
- [GitHub repository](https://github.com/Eppo-exp/php-sdk)
- [Package](https://packagist.org/packages/eppo/php-sdk)

## 1. Install the SDK

Run

```
composer require eppo/php-sdk
```

## 2. Initialize the SDK

Initialize the SDK with an API key, which can be generated in the Eppo interface. Initialization should happen when your application starts up to generate a singleton client instance, once per application lifecycle:

```php
use Eppo\EppoClient;

require __DIR__ . '/vendor/autoload.php';

$eppoClient = EppoClient::init(
   "<your_api_key>",
   "<base_url>", // optional, default https://eppo.cloud/api
   $assignmentLogger, // optional, must be an instance of Eppo\LoggerInterface
   $cache // optional, must be an instance of PSR-6 CacheInterface. If not passed, FileSystem cache will be used
);
```

To make the experience of using the library faster, there is an option to start a background polling for randomization params.
This way background job will start calling the Eppo api, updating the config in the cache.

For this, create a file, e.g. `eppo-poller.php` with the contents:

```php
$eppoClient = EppoClient::init(
   "<your_api_key>",
   "<base_url>", // optional, default https://eppo.cloud/api
   $assignmentLogger, // optional, must be an instance of Eppo\LoggerInterface
   $cache // optional, must be an instance of PSD-16 SimpleInterface. If not passed, FileSystem cache will be used
);

$eppoClient->startPolling();
```

after this, run this script by:

`php eppo-poller.php`

This will start an indefinite process of polling the Eppo-api.

### Define an assignment logger (experiment assignment only)

If you are using the Eppo SDK for experiment assignment (i.e randomization), pass in a callback logging function to the `EppoClient::init` function on SDK initialization. The SDK invokes the callback to capture assignment data whenever a variation is assigned.

The code below illustrates an example implementation of a logging callback using Segment. You could also use your own logging system, the only requirement is that the SDK receives a class with `logAssignment` method. Here we define an implementation of the `Eppo\Logger\LoggerInterface` interface containing a single function named `logAssignment`:

```php
<?php

use Eppo\Logger\LoggerInterface;

class Logger implements LoggerInterface {
  public function logAssignment(
    string $experiment,
    string $variation,
    string $subject,
    string $timestamp,
    array $subjectAttributes = []
  ) {
    var_dump(
      json_encode([
        'experiment' => $experiment,
        'variation' => $variation,
        'subject' => $subject,
        'timestamp' => $timestamp,
      ]);
    );
  }
}
```

The SDK will invoke the `logAssignment` method with the following parameters:

| Field | Description | Example |
| --------- | ------- | ---------- |
| `experiment` (string) | An Eppo experiment key | "recommendation_algo" |
| `subject` (string) | An identifier of the subject or user assigned to the experiment variation | UUID |
| `variation` (string) | The experiment variation the subject was assigned to | "control" |
| `timestamp` (string) | The time when the subject was assigned to the variation | 2021-06-22T17:35:12.000Z |
| `subjectAttributes` (map) | A free-form map of metadata about the subject. These attributes are only logged if passed to the SDK assignment function | `{ "country": "US" }` |

:::note
More examples of logging (with Segment, Rudderstack, mParticle, and Snowplow) can be found in the [event logging](/experiments/prerequisites/event-logging/) page.
:::

## 3. Assign variations

Assigning users to flags or experiments with a single `GetAssignment` function:

```php
<?php

use Eppo\EppoClient;

require __DIR__ . '/vendor/autoload.php';

$eppoClient = EppoClient::init(
   "<your_api_key>",
   "<base_url>", // optional, default https://eppo.cloud/api
   $assignmentLogger, // optional, must be an instance of Eppo\LoggerInterface
   $cache // optional, must be an instance of PSR-6 CacheInterface. If not passed, FileSystem cache will be used
);

$subjectAttributes = [];
$variation = $eppoClient->getAssignment('subject-1', 'experiment_5', $subjectAttributes);

if ($variation === 'control') {
    // do something
}

```

The `getAssignment` method takes two required and one optional input to assign a variation:
- `subjectKey` - The entity ID that is being experimented on, typically represented by a uuid.
- `flagOrExperimentKey` - This key is available on the detail page for both flags and experiments.
- `targetingAttributes` - An optional map of metadata about the subject used for targeting. If you create rules based on attributes on a flag/experiment, those attributes should be passed in on every assignment call.

### Handling the empty assignment

We recommend always handling the empty assignment case, when the SDK returns `""`. Here are some examples illustrating when the SDK returns `""`:

1. The **Traffic Exposure** setting on experiments/allocations determines the percentage of subjects the SDK will assign to that experiment/allocation. For example, if Traffic Exposure is 25%, the SDK will assign a variation for 25% of subjects and `""` for the remaining 75% (unless the subject is part of an allow list).

2. If you are using Eppo for experiment assignments, you must start the experiment in the user interface before `GetAssignment` returns variations. It will return `""` if the experiment is not running, both before and after.

  ![start-experiment](../../../../static/img/connecting-data/StartExperiment.png)

<br />

:::note
It may take up to 5 minutes for changes to Eppo experiments to be reflected by the SDK assignments.
:::

