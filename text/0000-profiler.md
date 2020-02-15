---
title: Low Impact LWC LifeCycle Profiler & Usage Counter
status: DRAFTED
created_at: 2020-01-08
updated_at: 2020-02-13
pr: (leave this empty until the PR is created)
---

# Low Impact LWC LifeCycle Profiler & Usage Counter

---- TODOs
Talk about profiler hooks replacing the current calls to log performance timing

Profiler hooks method signature to capture data

Clear profiler interface and what will then need to belong within the lwc engine if we're going to implement the profiler interface. 
---- TODOs

## Summary

Allow for selective collection of LWC lifecycle events timing in a lightweight manner in production. The collected events are asynchronously available for processing 

In addition to collecting lifecycle events, we will collect render counts of components

Each of these will be behind it's own feature flag. Access to this data will be available through an experimental api added to the engine.

## Basic example

These examples assume that the relevant feature flags have been turned on.
### Accessing profiler api
```js
let profilerApi;
// api can only be called once in the lifetime of the window
// to ensure that we have a single profiler consumer
engine.__unstable_registerProfilerConsumer(function(privProfilerApi)) {
    profilerApi = privProfilerApi;
}
```

### Component Counts
```js
// getting counts of components
console.log(profilerApi.getComponentUsage());
profilerApi.resetComponentUsage();
//
```

### Accessing LWC Lifecycle Events
The buffer for storage of events will be provided by the application hosting the lwc engine.
Because we're storing all data in a typed array, the we need to be able to map strings to integers
globally
```js
// The lifecycle events profiler collects events in a buffer that you need to 
// set from the outside. This is so that events from other sources (aura, lds) etc
// can be collected in this single performant SharedArrayBuffer / TypedArray
const buffer = new Float64Array(100000);
// fmMapStrignToInt is need to convert any strings to integers for storage 
// in the TypedArray
profilerApi.setBuffer(buffer, fnMapStringToInt);
```


## Motivation

The performance-timing rfc implementation made it easy to visualize lwc lifecycle events in chrome dev tools. Unfortunately the methodology (browser performance marks) of instrumenting these events was resource heavy, resulting in more than 25% degradation in lwc performance metrics. Thus this collection is only possible in DEV mode. 

The information collected in these lifecycle events that can be used to understand part of why a customer scenario is slow in production and to get back information on how much cpu was used by lwc lifecycle phases and what components were responsible for that. If we couple this information with the time spent servicing data requests for components, we can get a reasonable estimate of end to end loading performance of a component. 

Instead of pushing data to performance.mark and performance.measure api that can take significant time and limit you only to a single string to contain information on the component, we're going to use a TypedArray buffer to store the events. Using a TypedArray [is 10x faster](https://jsperf.com/perf-mark-vs-buffer) ([test source](#JSPerf-Test)) compared to using measure and mark APIs.

While we can theoretically get a count of the lwc components used in an application by traversing through the lifecycle events buffer, it is an expensive way if we simply want to count the number and types of components created. To allow for broader and efficient collection of component usage, we will have a separate api just for tracking and fetching component usage counts.

## Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with Lightning Web Components to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

The approach here is to replace the current instrumentation points that call into the browser's performance timing api with a call to log the operations into a dedicated profiler buffer instead. At that point we can choose to generate performance marks as well if we're not in production mode.

We have 2 choices with the profiler buffer. Does it remain with LWC or stay outside ? 

### Profiler API

```javascript
/**
 * @param {number} operationId 
 * @param {string} cmpName
 * @param {number} cmpId
 * @param {number} parentId 
 */
function logOperationStart(operationId, cmpName, cmpId, parentId){};
function logOperationStop (operationId, cmpName, cmpId, parentId){};


```


## Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people Lightning Web Components
- integration of this feature with other existing and planned features
- cost of migrating existing Lightning Web Components applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

* 

## Alternatives

What other designs have been considered? What is the impact of not doing this?

## Adoption strategy

This is transparent to LWC developers. Whether this is used by anyone, doesn't matter. 

# How we teach this

This is an experimental API at this point. This feature will not be documented externally.

# Unresolved questions

None

# Appendix

## JSPerf Test
In case jsperf isn't available, here's the breakdown
### Setup
```js
// Create N random strings to represent the name of what's being measured
function randomStringGen(n) {
let strings = [];
for (let i = 0; i < n; i++) {
    let s = Math.random().toString(36).substring(2, 15) + Math.random().toString(36).substring(2, 15);
    strings[i] = s;
}
return strings;
}

var perfNow = performance.now.bind(performance);
var perfMark = performance.mark.bind(performance);
var perfMeasure = performance.measure.bind(performance);
var perfClearMarks = performance.clearMarks.bind(performance);
var perfClearMeasures = performance.clearMeasures.bind(performance);


let stringMap = {};
let stringCount = 0;
// typed arrays can only store numbers, so we convert string to numbers
function stringToInt(s) {
    let i = stringMap[s];
    if (i === undefined) {
        i = stringCount++;
        stringMap[s] = i;
    }
    return i;
}

let iterations = 10000;
let strings = randomStringGen(iterations);
let buffer = new Float64Array(iterations * 10);
```

### Baseline performance mark & measure
```js
for (let i = 0; i < iterations; i++) {
    let str = strings[i];
    perfMark(str);
    perfMeasure(str, str);
    perfClearMarks(str);
    perfClearMeasures(str);
}
```

### Comparison TypedArray
equivalent of information collected through performance mark & measure
```js
let j = 0;
for (let i = 0; i < iterations; i++) {
    let str = strings[i];
    buffer[j] = i;
    buffer[j++] = i + 1;
    buffer[j++] = perfNow();
    buffer[j++] = stringToInt(str);
    buffer[j++] = perfNow();
    buffer[j++] = stringToInt(str);
}
```
