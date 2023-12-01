# slog for logging

* **Status**: Open
* **Author**: Nithya (@nithyatsu)

## Background

Radius has been using logr/zapr for generating logs, which has been satisfactory so far. However, there is room for improvement.

Zap is an external library that we adopted when Go did not have a native solution for structured logging. Now, Go 1.21 introduces the slog library, which offers both high performance and rich features, such as structured logging, context awareness and log levels. 

We wanted to use nested logs to avoid empty fields in some cases and organize log structure better, but zap made it difficult to achieve. As a result, Radius adopted a flat log structure, which reduces the readability of the logs in elastic search/ log UI. slog makes nested log structures easy with slog.Group.

We have also encountered some issues with zapr/logr levels, This is primarily since the approach to setting levels on logs is non-intuitive.

Since Go now has its own structured logging library, it would be beneficial to switch from zapr/logr to slog/slogr for Radius logging and improve on the above points.

## Terms and definitions

## Objectives

### Goals

1. Define log structure for Radius services. The fields should be the same as ones currently present, but with introduction of nested structure for better organization.

2. Implement the log structure 

### Non-Goals

NA



## Design

### library
Go's slog library 

### Log structure
We will retain current logging fields. Some of these will be nested into "additionalProperties" structure when they appear. 

|Field name| Field description|
|---|---|


## References

| Title | Links |
|---|---|

