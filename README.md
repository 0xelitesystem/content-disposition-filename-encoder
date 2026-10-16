# Content-Disposition Filename Encoder

Type any download filename and get a Content-Disposition header that survives every receiver: an ASCII filename fallback placed first, plus an RFC 8187 filename*=UTF-8''... value, with each step explained. A second mode parses a pasted header the way RFC 6266 tells recipients to, and flags what a real parser may do with the same header.

**Live demo:** https://0xelitesystem.github.io/content-disposition-filename-encoder/

## Use

**Build.** Type the filename, any Unicode. Pick `attachment` or `inline` and how the ASCII fallback is made: transliterate (NFKD, drop combining marks U+0300 to U+036F, replace the rest with `_` or a character you choose), replacement only, or your own fallback. The page shows each step with its rule:

1. Sanitise, per the recipient advice in RFC 6266 section 4.3 applied on the sending side: keep only the last path segment, strip control characters, trim leading and trailing whitespace, and warn on `.`, `..`, `~` and `|`. Each change is shown and can be switched off.
2. Decide whether `filename*` is needed: only when the plain `filename` cannot carry the name faithfully.
3. Build the fallback: printable ASCII only, token form when every character allows it, quoted-string otherwise. A `"` or `\` is written as a quoted-pair with a warning, and a `%` followed by two hex digits gets a warning and an offer to replace the `%`.
4. Build `filename*`: `UTF-8''` followed by the name, with every character outside attr-char written as its UTF-8 octets in uppercase `%XX`. Never quoted.
5. Put `filename` first and `filename*` second.

You get the header line, copy buttons for the value and the full header, plain string snippets for Node's `res.setHeader`, Starlette, Django and an S3 presigned URL's `ResponseContentDisposition` parameter (the value only; nothing is claimed about how S3 handles it), the value's length in bytes, and a panel showing what `encodeURIComponent` would have produced and why that is wrong here.

**Parse.** Paste a header, with or without `Content-Disposition:`. The page shows the RFC 6266 recipient result (filename* when it decodes, otherwise filename) next to a reading of Chromium's parser, then a list of findings: duplicate parameters, a quoted `filename*`, `filename*` placed before `filename`, a `%` plus two hex digits or raw non-ASCII in the plain filename (with a live preview of the UTF-8 bytes read as ISO-8859-1), path separators, surrounding whitespace, malformed escapes and other charsets.

### Worked examples

- `€ rates.pdf` gives `filename="_ rates.pdf"; filename*=UTF-8''%E2%82%AC%20rates.pdf`.
- `naïve résumé.pdf`, transliterated, gives `filename="naive resume.pdf"; filename*=UTF-8''na%C3%AFve%20r%C3%A9sum%C3%A9.pdf`.
- `report (final)*'v2'.pdf` is faithful as a quoted plain filename. Its RFC 8187 form is `report%20%28final%29%2A%27v2%27.pdf`, while `encodeURIComponent` gives `report%20(final)*'v2'.pdf`, which leaves `'`, `(`, `)` and `*` raw.
- `100%25 done.txt` warns about the `%25` in the fallback and adds `filename*=UTF-8''100%2525%20done.txt`.
- `漏洞.txt` gives `filename="__.txt"; filename*=UTF-8''%E6%BC%8F%E6%B4%9E.txt`.
- `../../etc/passwd` is sanitised to `passwd`, with the change shown.
- Parsing `attachment; filename*=UTF-8''%E2%82%AC.txt; filename="x.txt"` gives `€.txt` and an order warning.
- Parsing `attachment; filename*="UTF-8''a.txt"; filename="b.txt"` gives `b.txt`: the quoted `filename*` is invalid, and Chromium's reading rejects it too.

### When this is useful

- Users report downloads saved under the wrong name or with garbled characters, and you need a header that carries the name for both old and new receivers.
- You are writing the download endpoint in a framework that leaves this header to you and want a value you can paste in.
- You are reviewing a header a backend already sends and want to know which filename a recipient ends up with.
- You want to check a library's output for the `encodeURIComponent` mistake.

## The rules

From [RFC 6266](https://www.rfc-editor.org/rfc/rfc6266.txt):

- 4.3: "Therefore, when both "filename" and "filename*" are present in a single header field value, recipients SHOULD pick "filename*" and ignore "filename"."
- 4.3, on paths: recipients should never trust folder names, "by stripping all but the last path segment and only considering the actual filename (where 'path segments' are the components of the field value delimited by the path separator characters "\" and "/")."
- 4.1: "Content-Disposition header field values with multiple instances of the same parameter name are invalid."
- Appendix D: "Avoid including the percent character followed by two hexadecimal characters (e.g., %A9) in the filename parameter, since some existing implementations consider it to be an escape character, while others will pass it through unchanged."
- Appendix D: "Although most existing implementations will decode them as ISO-8859-1, some will apply heuristics to detect UTF-8, and thus might fail on certain names."
- Appendix D: "When a "filename" parameter is included as a fallback (as per above), "filename" should occur first, due to parsing problems in some existing implementations."

From [RFC 8187](https://www.rfc-editor.org/rfc/rfc8187.txt), which defines the ext-value used by `filename*`: RFC 6266 cites RFC 5987 for it, and RFC 8187 (September 2017) obsoletes RFC 5987. The rfc-editor metadata, checked when this page was built, lists no document that obsoletes or updates RFC 8187 or RFC 6266.

- The grammar:

  ```
  attr-char     = ALPHA / DIGIT
                / "!" / "#" / "$" / "&" / "+" / "-" / "."
                / "^" / "_" / "`" / "|" / "~"
                ; token except ( "*" / "'" / "%" )
  ```
- "Producers MUST use the "UTF-8" ([RFC3629]) character encoding."
- The encoding "deviates from common parameter syntax in that the quoted-string notation is not allowed."
- For a malformed or undecodable value, the RFC lists three acceptable recipient strategies (ignore the parameter, strip the octets, substitute U+FFFD) without picking one. The parse mode ignores the parameter and says so.

Why not `encodeURIComponent`: [ECMA-262's Encode](https://tc39.es/ecma262/#sec-encode) says "Let alwaysUnescaped be the string-concatenation of the ASCII word characters and "-.!~*'()"." Of those, `*` and `'` are excluded from attr-char and `(` and `)` are not token characters, and the unescaped `'` collides with the delimiter in `UTF-8''`. In Node 25.9.0 and in the page's own self test, `encodeURIComponent` also throws `URIError` on a lone surrogate, which this page reports as "cannot be encoded as UTF-8".

What browsers do with percent escapes in the plain `filename`: [MDN's Content-Disposition page](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition) (retrieved 2026-10-16) reports that Firefox and Chrome decode percent escapes in plain filename and Safari does not. That is MDN's report, not a measurement made for this page.

Raw non-ASCII bytes in the plain `filename` are also fragile in transit; see nodejs/node#58240 (auto-closed as stale).

## The Chromium column

The parse mode's second column models `net/http/http_content_disposition.cc` from the [Chromium source](https://source.chromium.org/chromium), as read when this page was built. It is a reading of the code, not a test of any browser, and nothing is claimed about Firefox or Safari internals. What it models:

- `HttpContentDisposition::Parse` keeps the first `filename` and the first `filename*` that decode to something, and uses `filename*` when it is non-empty.
- `DecodeExtValue` rejects a `filename*` value that contains a double quote, and clears one whose value is not ASCII.
- `DecodeWord` percent-unescapes the plain filename and keeps the result only if it is valid UTF-8; its comment reads "What IE6/7 does: %-escaped UTF-8."
- For raw non-ASCII in the plain filename, the comment on `DecodeFilenameValue` reads "Non-ASCII strings are interpreted based on |referrer_charset|." The code tries UTF-8 first; the referrer charset case is not modelled, and the page says so.

## Self test

The page runs its self test on load and prints a live pass and fail count. Every worked example is built and then parsed back and must round-trip to the exact NFC name; the 95 printable ASCII characters are each checked as literal or `%XX` exactly per the attr-char rule; the `encodeURIComponent` contrast is computed live; and a lone surrogate, duplicate parameters, `ISO-8859-1'en'%E9`, a malformed `%E2%82`, case variants and a CRLF or NUL in the paste each have a case. The last row is a positive control written to fail.

## Why this exists

The sender follows RFC 6266 Appendix D, the receiver follows RFC 6266 section 4.3 and its own legacy behaviour, and the bytes in between are where filenames break. Getting both parameters right by hand means knowing which characters each one allows, which order to put them in, and which encoder is wrong for the job. This page does it in front of you and shows its working.

Single file, no build step, no network call, no tracking, MIT licensed.

## Privacy

Everything runs in your browser. The page makes no network call of any kind: no code, font, analytics or telemetry request, and nothing you type is transmitted. A Content-Security-Policy in the page forbids network connections. The only thing it stores is your light or dark theme choice, in your browser's local storage. Save the file and it works offline from `file://`.

## Run locally

```
git clone https://github.com/0xelitesystem/content-disposition-filename-encoder.git
cd content-disposition-filename-encoder
```

Open `index.html` in a browser. Or serve it:

```
python -m http.server 8000
```

then open `http://localhost:8000/`.

## Build

No build. One self-contained `index.html` with inline CSS and JavaScript. No dependencies, no package manager, no bundler, no CDN, no web font.

## Independence and provenance

This is an independent tool. It is not affiliated with or endorsed by the IETF, Google, Mozilla, Apple or MDN. Behaviour was implemented from the RFC text; Chromium source was read for the comparison column; no code is reproduced here.

## Third-party notices

**RFC 6266** (https://www.rfc-editor.org/rfc/rfc6266.txt). Holder line, copied from its Copyright Notice: "Copyright (c) 2011 IETF Trust and the persons identified as the document authors. All rights reserved." **RFC 8187** (https://www.rfc-editor.org/rfc/rfc8187.txt). Holder line, copied from its Copyright Notice: "Copyright (c) 2017 IETF Trust and the persons identified as the document authors. All rights reserved." Both are subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (https://trustee.ietf.org/license-info). What was taken: the short rule sentences quoted on the page and in this README, and the attr-char grammar line.

**Chromium** (https://source.chromium.org/chromium). Holder line, copied from https://raw.githubusercontent.com/chromium/chromium/main/LICENSE: "Copyright 2015 The Chromium Authors". Licensed BSD-3-Clause. What was taken: two source comments from `net/http/http_content_disposition.cc`, quoted above: "What IE6/7 does: %-escaped UTF-8." and "Non-ASCII strings are interpreted based on |referrer_charset|." Function names are cited as references; no code is reproduced.

**ECMA-262** (https://tc39.es/ecma262/). What was taken: the one sentence defining alwaysUnescaped, quoted above.

**MDN Web Docs**. Nothing is quoted. MDN's report on percent escapes is paraphrased and attributed, with a link.

## Related

- https://github.com/0xelitesystem/filename-and-path-limits-reference
- https://github.com/0xelitesystem/mime-type-lookup
- https://github.com/0xelitesystem/http-header-analyzer

## License

MIT. Copyright (c) 2026 0xelitesystem. See [LICENSE](LICENSE).
