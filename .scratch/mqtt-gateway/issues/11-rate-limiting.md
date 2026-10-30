# 11 — Rate limiting

**What to build:** One misbehaving caller cannot exhaust the gateway through the publish endpoint,
and a caller that is throttled is told how long to wait.

**Blocked by:** 04

**Status:** ready-for-agent

- [ ] Exceeding the configured rate returns a throttled response in the standard problem shape, with
      a retry hint
- [ ] A burst within the configured allowance is accepted — the limiter is a token bucket, because
      bursty publishing is the normal shape of this traffic
- [ ] Other endpoints are unaffected
- [ ] Limits are configurable without a rebuild
- [ ] Rejected requests do no work beyond being rejected
- [ ] Rejections are counted and exposed as a metric
