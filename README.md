# Exponential Backoff Retry Custom Policy

This is a implementation of the Exponential BackOff Retry algorithm that increases the backoff period for each retry attempt in case of app error. Applying this to your API or proxy you would be able to:

  - Define an initial interval whose first attempt should wait (TO-DO)
  - Define a multiplier, used to increment the intervals between retries
  - Define a maximum interval, after which no increment on the interval will occur (TO-DO)
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
| Publish to Exchange | mvn deploy |

### Limitations
This policy makes use of the for-each module. Since there is no break condition to exit the loop, this policy iterates through all the objects in the attempt array (please see "How?" Section). This means that it is convenient to apply this policy with small values ​​of intervals and elapsed time, to keep the loop fast to iterate. Of course, the current logic could be replaced with custom code (java module, script execution). It is up to the user to implement it. Just keep in mind that one of the main motivations for this policy is not to have too many dependencies.


### Contribution

Want to contribute? Great!

* For public contributions - Just fork the repo, make your updates and open a pull request!
* For internal contributions - Use a simplified feature workflow following these steps:
   - Clone your repo
   - Create a feature branch using the naming convention feature/name-of-the-feature
   - Once it's ready, push your changes
   - Open a pull request for a review