# Capacity Estimation Cheat Sheet

## Traffic

### Average RPS

```text
RPS = requests per day / 86,400
```

### Peak RPS

Use an explicit peak multiplier based on the problem's traffic pattern.

## Storage

```text
Storage = objects × average object size
```

## Bandwidth

```text
Bandwidth = requests/sec × average response size
```

## Useful Units

```text
1 KB ≈ 10^3 bytes
1 MB ≈ 10^6 bytes
1 GB ≈ 10^9 bytes
1 TB ≈ 10^12 bytes
1 day = 86,400 seconds
1 month ≈ 30 days
1 year ≈ 365 days
```

## Interview Habit

Always state your assumptions before calculating.
