# Cookies-over-HTTP Bad

## A Problem

Cookies sent over plaintext HTTP are visible to anyone on the network. This visibility exposes substantial amounts of
data to network attackers (passive or active). We know, for example, that long-lived and stable cookies have enabled
[pervasive monitoring][1] in the past (see [Google's PREF cookie][2]), and we know that HTTPS provides significant
confidentiality protections against this kind of attack.

It would be good to encourage developers to stop creating these monitoring opportunities (and to remove themselves as
a mixed content barrier to other sites) by migrating to HTTPS.

## A Proposal

TL;DR: Expire cookies early rather than sending them over non-secure connections.

When building the `Cookie` header for an outgoing request to an non-secure URL, we'll first check each cookies'
creation date. If that date is older than some arbitrary cutoff (let's start with twelve months), we won't add the
cookie to the outgoing `Cookie` header. Instead, we'll delete the cookie. Over time, we'll reduce the cutoff to
something suitably small (say, zero months).

We'll also slightly modify the creation-time algorithm in [step 11.3 of section 5.3 of RFC6265][3] by persisting
the creation time only for cookies whose value doesn't change (which turns out to be a no-op for Chrome, as that's
its current behavior).

That's it.

_Note that this proposal does not aim to prevent third-party tracking, except insofar as that tracking is done over
non-secure connections. It aims to set HTTPS as the minimum bar for third-party tracking capabilities (which,
hopefully we'd all agree are "powerful") in the hopes of ensuring that the third-party in question is the only one
with whom you're conversing._

## FAQ

### Won't this break things?

Cookies are somewhat fragile, and can be [evicted][4] at any time for reasons outside developers' control, so there
is unlikely to be a high compatibility cost: users are not likely to see breakage. On the other hand, services that
use long-lived non-secure cookies are likely to be unhappy, which is good. There are distinct risks to sending
cookies over non-secure channels, especially when done at scale as part of an advertising network.

Developers responsible for affected services have a few options:

1.  They can migrate to HTTPS, which is good for everyone.

2.  They can adopt a system similar to DoubleClick's rotation of its `ID`, whose value is re-encrypted ~daily. Folks
    who need to maintain state in areas of the world where HTTPS is difficult to implement may be forced to stick
    with this option longer than they'd like.

3.  They could migrate away from cookies as identifiers, towards something like `localStorage`, or, much worse,
    fingerprinting. Our goal should be to ensure that the friction involved with rebuilding their entire
    infrastructure to use a different identification mechanism will be higher than the pain introduced as we roll
    out this change.

4.  They could trivially modify the cookie value as a perversion of the approach in #2 above. That is,
    `Set-Cookie: name=[value]-1`, followed by `Set-Cookie: name=[value]-2`, followed by
    `Set-Cookie: name=[value]-3`, and so on. If this is the route developers choose, browsers would be well-served
    to consider countermeasures. Given past experience, this doesn't seem likely: for example, only a very small
    number of sites switched away from `<input type="password">` when Chrome started showing the "Not Secure" chip
    for password fields on non-secure pages.

The most worrisome group for unintentional breakage are Enterprisey intranet single-sign on servers. This doesn't
seem terribly concerning, because it seems useful to encourage even enterprises to migrate to HTTPS for critical
services like SSO. If it turns out that we ought to be more concerned about this, browsers could consider adding
an enterprise policy to unblock old cookies for a given set of sites (though it seems valuable to avoid doing so
if possible).

### What's a reasonable cutoff point to start with?

Chrome's added metrics to measure the age of the oldest cookie in each first-/third-party request sent to a
non-secure endpoint. As of March, 2018, the percentile buckets break down as follows (ages in ~days):

| | First-Party | Third-Party |
|-|-------------|-------------|
| 20% | 0-1 | 2-3 |
| 40% | 3-4 | 33-37 |
| 60% | 37-42 | 75-84 |
| 80% | 120-135 | 171-192 |
| 95% | 437-492 | 437-492 |
| 99% | 701-789 | 701-789 |

Squinting a bit, it seems reasonable to start at somewhere around a year. That would have a one-time effect on
~7% of cross-site requests, and ~6% of same-site requests. It's a compromise between a short-enough lifetime to
have a real impact on pervasive monitoring and tracking in general, while at the same time not actually breaking
things like SSO on an ongoing basis (being forced to reauthenticate once a year doesn't seem like a massive
burden).

### Why base this on creation date rather than limiting expiration?

We could limit the expiration time for cookies set over HTTP (that's what Martin Thompson's ahead-of-its-time
[omnomnom][5] proposal suggests). We'd have a hard time doing the same for cookies set over HTTPS, however, and
those cookies can also be sent over HTTP if they lack the `Secure` attribute. [Mozilla's research on the topic][6]
(and Chrome's data) suggests that that happens far too often to be easily adjustable.

The approach proposed in this doc allows a request-time decision which targets only those cookies which would
actually be sent over HTTP, which seems like a net that's just wide enough to catch the cookies we care about,
while not attacking those which _could_ be a problem, but _aren't_ in practice.

### Why base this on creation date rather than the Secure attribute?

Developers who are already serving their sites over HTTPS can avoid any impact of this proposal by annotating
their cookies with the `Secure` attribute, which is a robust protection against being delivered or modified
over HTTP. It would be lovely if more folks used that attribute: perhaps we should encourage them to do so by
modifying this proposal to affect all cookies that lack that attribute?

That approach might be simpler to communicate to developers, and it would go beyond the current protections
against pervasive monitoring to include a potential defense against DNS poisoning.

These are reasonable arguments. On the other hand, only something like 7.5% of cookies use the `Secure`
attribute according to Chrome's data. Since we'd be expiring cookies regardless of the _actual_ risk, and
since [something like 70% of users' navigations are to secure pages][7] that the current proposal excludes,
we'd end up affecting significantly more requests.

Still, this would be a more robust defense for users and developers. We're adding more metrics to Chrome to
see what the impact of this kind of approach might look like if we take more things into account (HSTS with
`includeSubdomains` obviates the needs for the `Secure` attribute, for example).

## Open Questions

1.  **All or nothing**? Is it better to treat a request's cookies as a monolithic set by deleting _all_ of
    them if _any_ cookie is expired due to age? That is, in the 2017-09-01 example above, should
    `http://A.com` receive no cookies at all?
     
    This is fairly draconian, but could be justified by noting that browsers don't understand cookies'
    intent, so ensuring a consistent state is important. Sending one cookie but not another might confuse
    a server in a dangerous way; as an extreme example, `doSecurityChecks=1` might be set once a year,
    while `sessionID` might be set daily.

    Counterpoints in favor of the current proposal include:
    
    1.  A server shouldn't be punished for old cookies it's forgotten about and isn't using
    2.  Cookies are fragile today, and older cookies will be expired first regardless.

2.  **Should we delete cookies? Or simply hide them?** A variant of this proposal would not remove the cookies
    when they'd be sent over non-secure transport, but instead treat them as though they'd been marked as
    `Secure`, at least for the purposes of building the `Cookie` header. That approach raises a few questions
    which we discussed in an earlier version of this proposal. For example:

    1.  Should we allow the server to overwrite the cookie we didn't send? (e.g. if `name=value` is older than
        the cutoff, would we accept `Set-Cookie: name=other-value` in the response?)

    2.  Should we inform the server that there's a hidden cookie they're not getting (perhaps by sending the
        cookie name and a blanked-out value, like `name=[value-omitted]`).

3.  **Do we need to care deeply about bypasses?** We discuss trivially bypassing this mechanism in #4 above.
    It seems that we could come up with heuristics to poke at that if it becomes common. Is that important?
    Should we care more than this document suggests that we should?

4.  **Should we special-case the cookie value "OPT_OUT"?** It would be unfortunate indeed if removing old cookies
    meant that users who had opted out of interest-based advertising started being targeted again. Perhaps
    excluding the special value `OPT_OUT` (and asking advertisers to standardize on it?) is justifiable.

[1]: https://tools.ietf.org/html/rfc7258
[2]: https://www.eff.org/deeplinks/2013/12/nsa-turns-cookies-and-more-surveillance-beacons
[3]: https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis#section-5.3
[4]: https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis#section-5.4
[5]: https://tools.ietf.org/html/draft-thomson-http-omnomnom-00
[6]: https://bugzilla.mozilla.org/show_bug.cgi?id=1160368
[7]: https://transparencyreport.google.com/https/overview
