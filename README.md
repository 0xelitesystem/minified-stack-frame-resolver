# Minified stack frame resolver

Paste a production stack trace and its source map, get the original file, line and column for every frame, with the off by one column correction applied and every frame it changes flagged.

**Live demo:** https://0xelitesystem.github.io/minified-stack-frame-resolver/

The tool always outputs the original file, line and column for every frame it can resolve. It outputs the original source line text only when the map embeds `sourcesContent`, and many maps do not carry it, in which case the tool says so in words rather than showing an excerpt. The live repo `stack-trace-cleaner` filters trace text by frame origin across six languages (Node, Python, Rust, Java, Ruby, Go) and reads no map file, so it cannot resolve a minified frame to an original position; this repo is a separate tool, not a companion to it, an improvement on it, or a successor to it.

## Use

1. Paste the stack trace on the left, exactly as you copied it from a browser console, a Node crash or an error dashboard. Lines that are not frames are listed back to you verbatim instead of being dropped.
2. Supply the `.map` file on the right: pick it, drop it, or paste the JSON. All three routes fill the same box. Nothing is fetched and no `sourceMappingURL` is followed, so the map has to come from you.
3. Press **Resolve frames**. You get one row per frame: the frame as reported, the original file, the original line, the original column, the original name at that token when the map carries one, and a verdict saying whether the correction changed the answer for that frame.
4. Press **Run self test** to run the assertions shipped inside the page, including a positive control that is designed to fail.

Every failure is a worded refusal attached to the frame or shown as a banner: an invalid map, an index map with a `sections` array, a version that is not 3, a character outside the Base64 alphabet, a generated line with no mappings, a column before the first mapping on its line, a segment that carries no original mapping, a map whose indices point past its own `sources` or `names`, and a map whose `sources` or `names` entry at the resolved index is not a string. No path in the code produces a best guess.

## What the correction is

V8 reports a 1-based column. A source map stores 0-based generated columns. So the lookup is a binary search for the greatest generated column less than or equal to the reported column minus one. That minus one is applied to the column before the search, and it is the whole correction.

The corrected and uncorrected lookups diverge if and only if the map contains a mapping segment at a generated column exactly equal to the reported 1-based column. Landing exactly on a segment start is the safe case, not the failing one. An uncorrected lookup overshoots and can land on the mapping segment for the following token; it can never land on an earlier one, because feeding a larger target column to a greatest-less-than-or-equal search returns the same segment or a later one.

The name column is the original identifier at the mapped token, taken from the map's `names` array. It does not name the enclosing function, and a frame can legitimately resolve with no name at all: a segment with only 4 fields carries no name index. This tool prints "no name in map" for those, and never substitutes the minified identifier, walks back to a nearby named segment, or infers a name from source text.

When an uncorrected lookup is wrong it is confidently wrong, and the answer it gives can be a different original line, not just a different column.

### Receipts you can recompute

Reference fixture: `underscore-umd-min.js.map`, shipped inside the npm package `underscore` version 1.13.8. Install that package, then paste its map into this tool alongside this trace:

```
Error: forced
    at underscore-umd-min.js:6:7876
    at underscore-umd-min.js:6:13792
    at underscore-umd-min.js:6:8187
```

Decoding that map reports 4276 mapping segments across 6 generated lines, 159 sources, 373 names, and no `sourcesContent`, and the three frames resolve like this:

| Frame | With the correction | Without the correction | Verdict |
| --- | --- | --- | --- |
| `underscore-umd-min.js:6:7876` | `modules/find.js` 7:50 name `findKey` | `modules/find.js` 8:23 name `obj` | changed, and it changed the line, not only the column |
| `underscore-umd-min.js:6:13792` | `modules/isEqual.js` 148:23 name `a` | `modules/isEqual.js` 148:25 name `key` | changed |
| `underscore-umd-min.js:6:8187` | `modules/map.js` 13:22 name `iteratee` | same segment | no change |

Frame level receipt on the same fixture, measured on 2026-09-09: forcing a throw through 34 calls into 33 different `underscore` 1.13.8 public APIs under Node v25.9.0, then taking every distinct `underscore-umd-min.js` position the resulting stack traces contain, gives 39 frames (n=39). 2 of those 39 resolve differently under the two lookups, and one of the 2 is `underscore-umd-min.js:6:7876`, the first row of the table above. The other 37 all land exactly on a segment start and all 37 resolve identically both ways, which is what makes landing on a boundary the safe case.

In that same map, 1238 of 4276 segments (29.0 percent) are 4 field segments carrying no name index at all. That map has no `sourcesContent`, so all three rows above also show the worded "source line text not available" refusal rather than an excerpt.

## The worked example

Press **Load the worked example** and the page fills both boxes with a small hand written map and a matching trace, then resolves them. The expected output, so the example is checkable:

| Frame | Result | Verdict |
| --- | --- | --- |
| `app.min.js:1:11` | `src/app.js` 7:50 name `user` | correction changed the answer; without it, `src/app.js` 8:23 name `total` |
| `app.min.js:1:41` | `src/util.js` 12:4, no name in map | no change |
| `app.min.js:2:26` | `src/util.js` 21:8 name `format` | no change |
| `app.min.js:2:6` | the nearest segment marks a generated position with no original mapping | no change |
| `app.min.js:2:2` | no mapping at or before this column | no change |
| `app.min.js:9:5` | generated line 9 has no mappings | no change |

The first line of the pasted text is an error message, not a frame, so it appears in the "lines that were not read as frames" block. The example map has no `sourcesContent`, so every resolved row also carries the worded refusal in place of an excerpt.

**Run self test** runs 68 assertions over that example and over the refusal paths, then one positive control that must fail. The report prints the control separately and calls the whole run unreliable if the control ever passes, which is how a self test that quietly stopped part way through becomes visible. The same button also sweeps every 1-based column position of the example map: 99 positions compared, the uncorrected lookup lands on a later segment 6 times and on an earlier segment 0 times.

## Why this exists

A minified frame is a position, and resolving it correctly is arithmetic that a person cannot do by eye. The closest existing offline tool, the source map visualization page at https://evanw.github.io/source-map-visualization/, takes files: its own instruction text reads "You can either drop a single JavaScript/CSS file with an inline source map comment, or a JavaScript/CSS file and a separate source map file together." It has no field for pasting a stack trace. The difference is the input artifact. This tool starts from the trace you already have in front of you, and answers one question per frame.

It is one HTML file with inline CSS and inline JavaScript. No CDN, no framework, no web font, no analytics, no telemetry, no network requests of any kind. Nothing to install, nothing to sign up for, no account, no upload. MIT licensed, so you can read every line, keep a copy, and put it on your own machine.

## Privacy

Everything runs in your browser. The stack trace and the source map you paste never leave your machine: there is no network code in the page, no external script, no font request and no analytics. Production maps are sensitive, since they carry your original file names and often your original source, which is exactly why this tool never sends one anywhere and never fetches one either.

## Run locally

```
git clone https://github.com/0xelitesystem/minified-stack-frame-resolver.git
cd minified-stack-frame-resolver
```

Open `index.html` in a browser. It works from `file://` with no server. If you prefer to serve it:

```
python -m http.server 8000
```

Then open `http://localhost:8000/`.

## Build

There is no build step and there are no dependencies. `index.html` is the whole tool.

## Third-party notices

No third party code is reproduced in this repository. The Base64 VLQ decoder and the mappings grammar were reimplemented from the published Source Map specification; behaviour was verified against the specification text and against real maps, and no code was copied.

`underscore` is named only as a measurement fixture: its published map file is a convenient real world artifact you can obtain yourself and check these numbers against. `stack-trace-cleaner` and `span-offset-debugger` are other repos in this account. This tool is independent of and unaffiliated with the underscore project, with Ecma International, and with the source map visualization page named above. Naming them is description, not endorsement, in either direction.

## Related

- https://github.com/0xelitesystem/stack-trace-cleaner
- https://github.com/0xelitesystem/span-offset-debugger
- https://github.com/0xelitesystem/base64-url-encoder-decoder

## License

MIT. Copyright (c) 2026 0xelitesystem.
