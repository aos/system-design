# API Rate Limiter

Throttle users based upon the number of requests they are sending.

## Requirements & Goals

**Functional**:
1. Limit # of requests an entity can send to an API within a timing window
2. APIs are accessible through a cluster, rate limit should be considered
   across different servers

**Non-functional**:
1. Highly available
2. Should not introduce substantial latency

## Types of throttling
- **Hard**: # of API requests cannot exceed throttle limit
- **Soft**: # of requests can exceed up to a certain percentage
  - 10% exceed limit: 100 * 1.10 = 110
- **Elastic/Dynamic**: if free resources available, allow for exceeding
  threshold

## Types of algorithms used for rate limiting
- **Fixed window**: time window considered from the start of the time-unit to
  the end of the time-unit. Example: a period considered 0-60 seconds for a
  minute, irrespective of time frame at which API request has been made.

If rate limiting 2 messages per second, m5 will be throttled below.
```
        fixed window           fixed window
        2 messages             3 messages
   __________|_________   __________|_________
  |                    | |                    |

--|----------|-*--*-----|----*---*-|-----*----|--
 0.0s          m1 m2   1.0s  m3  m4      m5  2.0

             |_____________________|
                        |
                   rolling window
                   4 messages
```

- **Rolling window**: time window considered from fraction of time at which
  request is made plus time window length

## High level design
Separate service that the web server first asks the rate limiter. If not
throttled, it'll be passed to the API server.

## System design and algorithm
Create a hashtable with:
- Key: UserID
- Value: { Count, StartTime }
