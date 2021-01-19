# Exponential Backoff Retry Custom Policy

This is a implementation of the Exponential BackOff Retry algorithm that increases the backoff period for each retry attempt in case of app error. Applying this to your API or proxy you would be able to:

  - Define an initial interval whose first attempt should wait
  - Define a multiplier, used to increment the intervals between retries
  - Define a maximum interval, after which no increment on the interval will occur
  - Define a maximum elapsed time as a breaking point to stop retrying
  - Define expressions to validate the application's response. Expected responses can be configured in case no error occurs, or specific error expressions.  
  - Perform a dynamic retry strategy, abstracting from the complexity of retrying at application level

### Why?
The idea behind the Exponential Backoff Retry algorithm is to provide a greater degree of resilience in our services when performing a backoff that is not static in the event of an error event. In this way, under a persistent error scenario, we do not overload the impacted system/service, but we provide a graceful degradation mechanism increasing the waiting time between retries after each retry failure.


### How?
The logic uses the values ​​injected during the configuration of the policy to calculate the following variables:
- loopSize: this variable indicates the number of iterations to perform (used by the for-each). It is calculated at the beginning, and in the following way: ceil (ROUNDUP) of default_max_elapsed_time / 1000 / default_multiplier
- waitTime: this value indicates the time to wait. It is updated as follows:
	- At the beginning it is empty, because you have to try at least once to find out if there is an error
	- If there is an error the first time, it takes the value of default_initial_interval
	- If there are successive errors, it takes the value of waitTime * default_multiplier

Other variables are calculated during the process but are not specific to the algorithm, but are for internal use for flow control.
This policy does not have any dependency outside http-policy and mule-core components.

### Usage
After publishing to Exchange, follow these steps to apply the policy to an existing managed API (or proxy):

* Log into Anypoint Platform
* Enter API Manager
* Click on the API version for the application you want to apply the policy to
* Click on Policies
* Click on Apply New Policy
* Filter by 'Custom' category and select 'Exponential Backoff Retry'. Click on 'Configure Policy' button
* Give value to the policy's parameters:

| Parameter | Purpose |
| ------ | ------ |
| default_initial_interval | Indicates the initial wait time (when the first error happens) |
| default_max_elapsed_time | Indicates the maximum wait time. After this time, no retry will be performed. Similar to a timeout. |
| default_max_interval | Indicates the upper bound of the wait interval. Once this value is reached, the next retry will not be multiplied by the default_multiplier |
| default_multiplier | Indicates the increment value between repetitions. Between attempts, the calculation of the current wait time is given by previous interval value * multiplier |


#### Development

The following commands are required during development phase

| Task | Command |
| ------ | ------ |
| Package policy| mvn clean install |
| Publish to Exchange | mvn deploy - Make sure to update the pom.xml file with your org ID - |

### Limitations
This policy makes use of the for-each module. Since there is no break condition to exit the loop, this policy iterates through all the objects in the array (please see "How?" Section). This means that it is convenient to apply this policy with small values ​​of intervals and elapsed time, to keep the loop fast to iterate (please see Benchmark section below). Of course, the current logic could be replaced with custom code (java module, script execution). It is up to the user to implement it. Just keep in mind that one of the main motivations for this policy is not to have too many dependencies.

### Benchmark
The policy has been tested using a Mule application that returns error or ok at random, based on a number generated with the random() Dataweave function (https://docs.mulesoft.com/mule-runtime/4.3/dw-core-functions-random). If it is odd, it returns an error (MULE:EXPRESSION, "org.mule.weave.v2.runtime.core.exception.DivisionByZeroException: Division by zero"), otherwise the result is satisfactory.  The purpose of this test was to emulate a transient error, i.e. network failure or server overload, to check effectiveness around error rate and overall performance. Please see the following results.

#### Application config

| Deployment Model | Runtime Version | Worker Size | Workers |
| ------ | ------ | ------ | ------ |
| CloudHub (US East - Ohio - ) | 4.3.0 | 0.1 | 1 |


#### Test #0 -  NO policy applied -

##### Test suite configuration

| # of request | # of threads | # of total concurrent requests | 
| ------ | ------ | ------ |
| 100 | 2  | 200 | 

##### Results

| Avg  | Median | Min | Max | Error Rate | Throughput |
| ------ | ------ | ------ | ------ | ------ | ------ |
| 398 ms | 378 ms | 349 ms | 1410 ms | %46.50 | 4.9/sec |

#### Test #1

##### Policy configuration

| default_initial_interval  | default_max_elapsed_time | default_max_interval | default_multiplier |
| ------ | ------ | ------ | ------ |
| 2000 ms | 10000 ms | 10000 ms | 1.5 |

##### Test suite configuration

| # of request | # of threads | # of total concurrent requests | 
| ------ | ------ | ------ |
| 100 | 2  | 200 | 

##### Results

| Loop Size | Avg  | Median | Min | Max | Error Rate | Throughput |
| ------ | ------ | ------ | ------ | ------ | ------ | ------ |
| 7 | 7813 ms | 3391 ms | 369 ms | 41822 ms | %00.00 | 15.1/min |


#### Test #2

##### Policy configuration

| default_initial_interval  | default_max_elapsed_time | default_max_interval | default_multiplier |
| ------ | ------ | ------ | ------ |
| 2000 ms | 10000 ms | 5000 ms | 1.5 |

##### Test suite configuration

| # of request | # of threads | # of total concurrent requests | 
| ------ | ------ | ------ |
| 100 | 2  | 200 | 

##### Results

| Loop Size | Avg  | Median | Min | Max | Error Rate | Throughput |
| ------ | ------ | ------ | ------ | ------ | ------ | ------ |
| 7 | 4584 ms | 2416 ms | 369 ms | 39133 ms | %00.00 | 25.4/min |

#### Test #3

##### Policy configuration

| default_initial_interval  | default_max_elapsed_time | default_max_interval | default_multiplier |
| ------ | ------ | ------ | ------ |
| 2000 ms | 10000 ms | 3500 ms | 1.5 |

##### Test suite configuration

| # of request | # of threads | # of total concurrent requests | 
| ------ | ------ | ------ |
| 100 | 2  | 200 | 

##### Results

| Loop Size | Avg  | Median | Min | Max | Error Rate | Throughput |
| ------ | ------ | ------ | ------ | ------ | ------ | ------ |
| 7 | 2019 ms | 1120 ms | 367 ms | 20428 ms | %00.00 | 55.4/min |

#### Test #4

##### Policy configuration

| default_initial_interval  | default_max_elapsed_time | default_max_interval | default_multiplier |
| ------ | ------ | ------ | ------ |
| 2000 ms | 600000 ms | 3500 ms | 1.5 |

##### Test suite configuration

| # of request | # of threads | # of total concurrent requests | 
| ------ | ------ | ------ |
| 100 | 2  | 200 | 

##### Results

| Loop Size | Avg  | Median | Min | Max | Error Rate | Throughput |
| ------ | ------ | ------ | ------ | ------ | ------ | ------ |
| 400 | 3579 ms | 1572 ms | 365 ms | 37890 ms | %00.00 | 32.6/min |

#### Test #5

##### Policy configuration

| default_initial_interval  | default_max_elapsed_time | default_max_interval | default_multiplier |
| ------ | ------ | ------ | ------ |
| 2000 ms | 600000 ms | 120000 ms | 1.5 |

##### Test suite configuration

| # of request | # of threads | # of total concurrent requests | 
| ------ | ------ | ------ |
| 500 | 1  | 500 | 

##### Results

| Loop Size | Avg  | Median | Min | Max | Error Rate | Throughput | 95% percentile |
| ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ |
| 400 | 2772 ms | 1385 ms | 361 ms | 299453 ms | %00.20 | 21.6/min | 5615 |

#### Test #6

##### Policy configuration

| default_initial_interval  | default_max_elapsed_time | default_max_interval | default_multiplier |
| ------ | ------ | ------ | ------ |
| 2000 ms | 600000 ms | 60000 ms | 1.5 |

##### Test suite configuration

| # of request | # of threads | # of total concurrent requests | 
| ------ | ------ | ------ |
| 500 | 1  | 500 | 

##### Results

| Loop Size | Avg  | Median | Min | Max | Error Rate | Throughput | 95% percentile |
| ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ |
| 400 | 1953 ms | 479 ms | 365 ms | 49785 ms | %00.00 | 30.7/min | 5220 |

### Contribution

Want to contribute? Great!

* For public contributions - Just fork the repo, make your updates and open a pull request!
* For internal contributions - Use a simplified feature workflow following these steps:
   - Clone the repo
   - Create a feature branch using the naming convention feature/name-of-the-feature
   - Once it's ready, push your changes
   - Open a pull request for a review
