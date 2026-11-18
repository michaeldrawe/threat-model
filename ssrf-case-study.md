# Case study: server-side request forgery in a website scanner

[SiteScope](https://github.com/michaeldrawe/Website-analyzer) is my public website scanner. A visitor types a web address, and the server fetches that site and reports on it. While reviewing it, I found that the server would fetch **any** address it was given.

## The flaw

The scanner passed the visitor's address straight to the server's `fetch`, and followed redirects automatically. So a visitor could make the server request:

- `http://127.0.0.1/` or `http://localhost/`, which are services on the server itself;
- `http://10.0.0.5/` or `http://192.168.1.1/admin`, which are machines on a private network;
- `http://169.254.169.254/latest/meta-data/`, which is a cloud provider's metadata service and can hold credentials;
- a public site that **redirects** to any of the above, which slips past a check that only looks at the first address.

This is server-side request forgery (CWE-918, part of OWASP's A10:2021). The scanner's report then showed the attacker what came back.

## The fix

I didn't write the guard from scratch. My claims platform already had one for fetching county web pages, so I moved it into the scanner. Both projects now use it.

1. **Check the address's shape first.** Only `http` and `https` are allowed, only on ports 80 and 443, and the address can't contain a user name or password. Internal names (`localhost`, `.local`, `.internal`, names with no dot) are refused.
2. **Resolve the name, then check every address it gives.** If any one of them is non-public, the request is refused. A name that looks up to both a public and a private address is refused too. The check covers IPv4 and IPv6, including the tricks that hide an IPv4 address inside an IPv6 one: IPv4 written as IPv6, NAT64 and 6to4.
3. **Pin the connection to the checked addresses.** Without this, the name could be looked up a second time when the connection is made and return a different, private answer. That attack is called DNS rebinding.
4. **Never follow redirects automatically.** Each redirect target goes through steps 1 to 3 again, up to five times.
5. **Fail in plain words.** The visitor sees "That address points inside a private network, so it can't be analyzed," and nothing about what lies behind it.

## How it's tested

The tests use a stand-in name lookup and a stand-in fetch, so they never touch the network:

- 18 cases for the address rules, including each IPv6 form that hides an IPv4 address;
- loopback, private and metadata addresses typed in directly are refused before any request is made;
- a public-looking name that looks up to a private address is refused;
- a public site that redirects to the metadata service is refused, and the stand-in fetch shows nothing was sent there;
- a chain of too many redirects is stopped;
- a normal public site works, and the request goes only to the address that was checked.

## What I took from it

- **Every feature that fetches a URL supplied by a user is an SSRF risk.** That includes a URL that came from a page the user pointed at.
- **Checking only the first address isn't enough.** Redirects and DNS rebinding both get around a check made only at the start.
- **When a guard already exists elsewhere in your code, reuse it, with its tests.** A second guard written from scratch would have had its own mistakes.
