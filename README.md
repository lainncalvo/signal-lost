# Signal Lost — Crypto Shutdown Registry (2026)

**Live site → https://lainncalvo.github.io/signal-lost/**

A single-page, open, and free website that archives crypto projects, protocols and
chains that shut down during 2026. Anyone can browse every shutdown, jump straight to
each project's own announcement, and read simple stats (timeline by month, how each
project announced it, and which categories lost the most projects).

## Why this exists

The goal is simple: **help spread good, verifiable information across the crypto
ecosystem.** When a project winds down, the news scatters across X threads, Discord
servers, and forum posts — and then disappears. This is a calm, neutral, source-linked
record so anyone can see what happened and check the original announcement for
themselves. No hype, no speculation, no financial advice — just what was publicly
reported, with a link back to the source.

**This is meant to be a community effort. Anyone is welcome to collaborate.**

## Contributing

Spotted a shutdown that's missing, or an error to fix? Two ways to help:

- **Open an [issue](https://github.com/lainncalvo/signal-lost/issues)** with the project
  name, date, and a link to the announcement.
- **Open a pull request** editing the data directly.

All the content lives in one JavaScript array near the bottom of `index.html`, inside
the `<script>` tag — look for `var DATA = [`. Each project is one object:

```js
{ n:"Project Name",
  iso:"2026-07-25",   // shutdown date, YYYY-MM-DD
  day:true,           // false if only the month is known (shows "≈ Jul 2026")
  url:"https://x.com/.../status/...",  // announcement link, or null
  c:"x",              // how it was announced: "x" | "discord" | "docs" | "deleted"
  cat:"DeFi" }        // Wallet | DeFi | L2/Infra | Gaming | NFT/Social | Trading/CEX | AI | Other
```

Set `url` to `null` when a project only deleted its account or posted on Discord — the
page shows a badge instead of a broken link. Every stat, chart, filter and counter is
computed from this array at runtime, so they update automatically.

### A note on what to contribute

Please keep contributions to **public, on-the-record information** — a project's own
announcement, a dated post, an official statement. **Do not add private or personal
details** about the maintainer, contributors, or anyone else (emails, phone numbers,
real names, addresses, etc.). This project is about public ecosystem history, not
people's private data. PRs that add private information will be closed.

## Credits & data

- **Source data** compiled by [@0xvietnguyen](https://x.com/0xvietnguyen), current as of
  Jul 25, 2026.
- **Cross-referenced** against [RootData's 2026 Crypto Dead Projects List](https://www.rootdata.com/archives/detail/2026%20Crypto%20Dead%20Projects%20List)
  (current as of Jul 26, 2026). Every project added from it was individually researched
  to confirm its shutdown date and locate a real announcement; unverifiable entries were
  left out rather than listed on trust.
- **Site** built and maintained by [@lain_calvo](https://x.com/lain_calvo).
- Categories are best-effort labels, not official classifications.

### How entries are sourced

Most entries link to the project's own announcement (X post, blog, governance forum).
Where no primary announcement could be found, the entry links to press coverage instead
and is tagged **Press coverage**. A few announced a shutdown with no archivable link and
are marked **Link not archived**. The `c` field records which: `x`, `docs`, `news`,
`discord`, or `deleted`.

## Disclaimer

Everything here is presented as publicly reported and is **not financial advice**.
Always verify against the original announcement before drawing conclusions.

## License

[MIT](LICENSE) — free to use, adapt, and build on.
