# Canvas Poster Attribute

## Authors

- Jonathan Kingston

## Participate

- [Issue tracker](https://github.com/jonathan-kingston/canvas-poster-attribute/issues)
- Prototype patches: [Chromium](patches/chromium-canvas-poster.patch), [Firefox](patches/firefox-canvas-poster.patch)
- [Test cases](test-cases/poster.html)
<!--
- [Discussion forum]
-->

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [User-Facing Problem](#user-facing-problem)
  - [HTML Can Capture State, Except Canvas](#html-can-capture-state-except-canvas)
  - [Canvas Is Inherently Imperative](#canvas-is-inherently-imperative)
  - [Goals](#goals)
  - [Non-goals](#non-goals)
- [User research](#user-research)
- [What Exists Today](#what-exists-today)
- [Proposed Approach](#proposed-approach)
  - [Rendering Semantics](#rendering-semantics)
  - [Why Poster Doesn't Return After Clearing](#why-poster-doesnt-return-after-clearing)
  - [Interaction with Fallback Content](#interaction-with-fallback-content)
  - [Dependencies on non-stable features](#dependencies-on-non-stable-features)
  - [Solving web archiving with this approach](#solving-web-archiving-with-this-approach)
  - [Solving print stylesheets with this approach](#solving-print-stylesheets-with-this-approach)
  - [Solving progressive enhancement with this approach](#solving-progressive-enhancement-with-this-approach)
  - [Solving Core Web Vitals (LCP) with this approach](#solving-core-web-vitals-lcp-with-this-approach)
  - [Solving social sharing with this approach](#solving-social-sharing-with-this-approach)
- [API Design: What Should `canvas.poster` Return?](#api-design-what-should-canvasposter-return)
  - [Alternative A: Live Bitmap as Data URL](#alternative-a-live-bitmap-as-data-url)
  - [Alternative B: Normal Attribute, Archival Tools Populate (Recommended)](#alternative-b-normal-attribute-archival-tools-populate-recommended)
  - [Explicit Capture Pattern](#explicit-capture-pattern)
- [Alternatives considered](#alternatives-considered)
  - [A `static-src` Attribute](#a-static-src-attribute)
  - [Extending Fallback Content Semantics](#extending-fallback-content-semantics)
  - [A CSS Property](#a-css-property)
  - [Automatic Bitmap Serialisation](#automatic-bitmap-serialisation)
  - [Just use fallback content with an image](#just-use-fallback-content-with-an-image)
  - ["Canvas is for dynamic content, if you want static, use `<img>`"](#canvas-is-for-dynamic-content-if-you-want-static-use-img)
  - ["Just run JavaScript during archiving/printing"](#just-run-javascript-during-archivingprinting)
- [Implementation Considerations](#implementation-considerations)
  - [For Browser Vendors](#for-browser-vendors)
  - [Specification Changes](#specification-changes)
  - [CSS Styling](#css-styling)
- [Accessibility, Internationalization, Privacy, and Security Considerations](#accessibility-internationalization-privacy-and-security-considerations)
  - [Accessibility](#accessibility)
  - [Internationalization](#internationalization)
  - [Privacy](#privacy)
  - [Security](#security)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

The HTML `<canvas>` element has a gap that its sibling `<video>` solved years ago: there's no declarative way to provide a static visual representation. When JavaScript hasn't run, can't run, or has already run and the page is being archived, canvas content vanishes. A `poster` attribute could fix this.

## User-Facing Problem

### HTML Can Capture State, Except Canvas

One of HTML's strengths is that the rendered state of a document is largely recoverable from its serialised form. When you save a web page, most of what you see can be reconstructed:

- **Text content** lives in the DOM. What you see is what's in the markup.
- **Images** have `src` attributes pointing to retrievable resources, or inline data URLs.
- **Form state** can be captured via `value` attributes, `checked` states, `selected` options.
- **Video** has `poster` for a static frame and `src` for the media resource.
- **SVG** is entirely declarative; the visual output *is* the markup.
- **CSS styling** lives in stylesheets and inline styles, fully serialisable.

Even dynamically modified DOM is capturable. If JavaScript adds elements, changes text, or modifies attributes, the resulting DOM can be serialised and the visual state reconstructed. The browser's "Save As" function, web archive formats, and archival tools all rely on this property.

Canvas breaks this model. The `<canvas>` element in the DOM is just a rectangle with dimensions. The actual visual content (the pixels) exists only in memory, produced by JavaScript execution. There's no attribute, no child element, no CSS property that captures what's been drawn. Serialise the DOM, and you get:

```html
<canvas width="400" height="300"></canvas>
```

That's it. The chart, the game frame, the visualisation: gone.

### Canvas Is Inherently Imperative

You get a bitmap, you draw to it with JavaScript, and the result appears on screen. This works brilliantly for interactive graphics, games, and visualisations. But it fails completely in several important contexts:

**Web archiving.** When you save a page (via "Save As", Safari's Web Archive, WARC capture, or tools like Webrecorder), the canvas bitmap exists only in memory at the moment of capture. The saved HTML contains just an empty `<canvas>` tag. Replay the archive with JavaScript disabled (or with scripts that depend on now-dead APIs), and you get nothing.

**Print rendering.** The spec acknowledges this partially: if a canvas "has been previously associated with a rendering context," it renders the current bitmap when printing. But if you're printing a page where scripts haven't executed, or where the canvas was drawn after the print stylesheet was applied, you get the fallback content or a blank rectangle.

**No-JavaScript contexts.** Privacy-focused browsers, aggressive content blockers, corporate environments with script restrictions, and accessibility tools that operate on static DOM all encounter blank canvases. The element's entire visual content is locked behind JavaScript execution.

**Progressive loading.** Unlike `<img>` with its `src` or `<video>` with its `poster`, canvas offers no way to show a meaningful placeholder while scripts load. The best you can do is a CSS background image, which doesn't participate in the element's intrinsic sizing or get captured by standard serialisation methods.

**Search engines and social cards.** Crawlers that don't execute JavaScript (or execute it with limited fidelity) can't see canvas content. Open Graph scrapers grabbing preview images find nothing to grab.

### Goals

- Enable web archiving tools to capture meaningful static representations of canvas content
- Provide a print-friendly fallback for canvas elements
- Allow progressive enhancement with immediate visual content while JavaScript loads
- Improve Core Web Vitals (LCP) for canvas-heavy pages
- Support social sharing / link preview extraction for canvas content
- Enable meaningful display for no-JavaScript browsing contexts
- Support offline PWA loading states

### Non-goals

- Replacing or deprecating canvas fallback content (they serve different purposes)
- Automatic real-time synchronisation between canvas bitmap and poster attribute
- Providing animation or video-based fallback content
- Solving canvas accessibility beyond visual static representation (screen readers should continue using fallback content)

## User research

Canvas is used extensively for:

- Data visualisations (Chart.js, D3 bindings, trading platforms)
- Image editing (Photoshop-like web apps, filters, cropping tools)
- Games (everything from casual to complex)
- Document rendering (PDF.js, CAD viewers)
- Signatures and drawing (e-signature platforms, whiteboards)
- Maps (rendering layers, custom overlays)

All of this content becomes a black hole when archiving. The Internet Archive's Wayback Machine, national library preservation efforts, legal discovery, and compliance archives all struggle with canvas-heavy pages because there's no declarative way to capture what was rendered.

Canvas is the one major HTML element where rendered state and serialised state have completely diverged.

## What Exists Today

Canvas has fallback content: whatever you put between the opening and closing tags.

```html
<canvas width="400" height="300">
  Your browser doesn't support canvas.
</canvas>
```

This was designed for graceful degradation when canvas wasn't universally supported. The fallback only renders when:

1. The browser doesn't support canvas at all (increasingly rare)
2. JavaScript is disabled AND the canvas hasn't been touched by script
3. The context is non-visual media

Critically, fallback content serves a different purpose than a static visual representation. Consider an interactive chart:

```html
<canvas id="sales-chart" width="600" height="400">
  <table>
    <caption>Quarterly Sales Data</caption>
    <tr><th>Q1</th><td>$2.4M</td></tr>
    <tr><th>Q2</th><td>$3.1M</td></tr>
    <tr><th>Q3</th><td>$2.8M</td></tr>
    <tr><th>Q4</th><td>$4.2M</td></tr>
  </table>
</canvas>
```

The table provides accessible, structured data for screen readers. But it's not a visual representation of the chart; it's semantic content that conveys the same *information* through a different modality. If you wanted to also provide a static image for print/archive contexts, where would it go? Nested inside the table? Replacing it? The model doesn't accommodate both needs cleanly.

You could try a CSS background image:

```css
#sales-chart {
  background-image: url('sales-chart-snapshot.png');
  background-size: 100% 100%;
}
```

This works visually in some contexts, but:

- Background images don't affect intrinsic sizing
- They layer behind the canvas content, causing visual conflicts if the canvas is semi-transparent
- Archive serialisation may or may not capture them depending on implementation
- They're not semantically associated with the canvas content
- Swapping the background off when JavaScript draws requires additional coordination

## Proposed Approach

Add a `poster` attribute to `<canvas>`, mirroring the existing attribute on `<video>`:

```html
<canvas width="400" height="300" poster="chart-snapshot.png">
  <table><!-- accessible data representation --></table>
</canvas>
```

### Rendering Semantics

The poster image would display when the canvas has no rendered content:

1. **Initial state.** Before any JavaScript draws to the canvas, the poster image renders, sized to fill the canvas dimensions (respecting `object-fit` if we want to allow that).

2. **After first draw.** Once `getContext()` is called and any draw operation occurs, the poster is replaced by the live canvas bitmap. This mirrors how video poster disappears when playback begins.

3. **In static/print media.** If the canvas hasn't been drawn to (scripts didn't run or haven't run yet), the poster image is used for print rendering and static serialisation.

4. **When archiving.** Archive formats (WARC, Safari Web Archive, browser "Save As", etc.) could capture either the current bitmap (if drawing has occurred) or fall back to the poster URL/image. The poster provides a guaranteed static representation that doesn't depend on script execution during capture.

### Why Poster Doesn't Return After Clearing

Once the canvas has been drawn to, the poster is permanently hidden for that page load. This matches video behaviour: pausing or ending a video doesn't restore its poster. It also avoids practical problems. Animation loops clear and redraw every frame; responsive layouts resize canvases (which clears the bitmap). Re-showing the poster in these cases would cause distracting flicker.

The poster represents the initial state before JavaScript takes control, not a fallback for "empty." Authors who want explicit reset behaviour can manage it themselves by toggling the poster attribute or overlaying elements.

### Interaction with Fallback Content

The poster and fallback content serve different purposes and would coexist:

| Context | What Renders |
|---------|--------------|
| Visual, JS enabled, canvas drawn | Canvas bitmap |
| Visual, JS enabled, canvas not yet drawn | Poster image |
| Visual, JS disabled | Poster image |
| Visual, canvas unsupported | Fallback content |
| Non-visual (screen readers) | Fallback content |
| Print, canvas was drawn | Canvas bitmap |
| Print, canvas not drawn | Poster image |

This separation is important. Fallback content is for accessibility and semantic equivalence. The poster is for visual static representation. A chart needs both: a data table for screen readers and an image for printing.

### Dependencies on non-stable features

None. This proposal relies only on existing, stable web platform features.

### Solving web archiving with this approach

Archival tools could capture pages with canvas content that remains viewable when replayed offline or years later:

```html
<canvas id="dashboard" width="800" height="600" poster="dashboard-snapshot.png">
  <!-- Accessible fallback content -->
</canvas>
```

The Internet Archive, national libraries, and legal/compliance archives would all benefit. Currently, archived pages with significant canvas usage (dashboards, visualisations, games) are effectively broken.

### Solving print stylesheets with this approach

Designers could provide print-optimised versions of canvas graphics without requiring JavaScript to execute during print rendering:

```html
<canvas id="dashboard" poster="dashboard-print.png">
  <!-- A higher-contrast, simplified version for print -->
</canvas>
```

### Solving progressive enhancement with this approach

Pages could display meaningful content immediately while JavaScript loads:

```html
<canvas id="interactive-map" poster="static-map.png">
  <img src="static-map.png" alt="Map of downtown area">
</canvas>
```

The user sees the static map instantly, then it becomes interactive once scripts load.

### Solving Core Web Vitals (LCP) with this approach

Canvas-heavy pages have a Core Web Vitals problem. A blank canvas contributes nothing to Largest Contentful Paint (LCP). The browser is waiting for JavaScript to execute and draw before it has any content to measure.

A poster attribute directly addresses this. The poster image loads and renders like any other image, becoming an LCP candidate immediately:

```html
<canvas id="hero-chart" width="1200" height="800" poster="hero-chart.png">
</canvas>
```

Consider a dashboard with multiple canvas charts. Without poster, LCP waits for:

1. HTML parse
2. JavaScript download
3. JavaScript parse and execute
4. Data fetch (often)
5. Canvas draw operations

With poster, LCP can fire as soon as step 1 completes and the poster images decode. The difference can easily be seconds on slower connections or devices.

### Solving social sharing with this approach

Open Graph scrapers and other link preview tooling could extract the poster image as a representative image of the canvas content:

```html
<canvas id="infographic" poster="infographic.png">
</canvas>
```

Tools that generate social cards would have a reliable image source without needing to execute JavaScript and screenshot the result.

## API Design: What Should `canvas.poster` Return?

An important design question arises around the IDL attribute.

### Alternative A: Live Bitmap as Data URL

One approach would have the getter return the current canvas bitmap:

```js
canvas.poster // → "data:image/png;base64,..." (current bitmap)
```

This seems convenient: the poster is always "up to date" with what's been drawn. But it has significant problems:

**Performance.** Encoding a bitmap to PNG/base64 isn't free. If code accesses `canvas.poster` repeatedly (logging, debugging, frameworks that enumerate properties), you're encoding the bitmap each time.

**Tainted canvases.** If the canvas has drawn cross-origin images without CORS, `toDataURL()` throws a security error. Would `poster` also throw? That breaks the expectation that attributes are just strings.

**Semantic confusion.** The video `poster` attribute is explicitly *not* a frame from the video; it's a separate resource the author provides.

**Update timing.** When does the getter snapshot the bitmap? Every draw call? Lazily on property access?

### Alternative B: Normal Attribute, Archival Tools Populate (Recommended)

The cleaner approach treats `poster` as a standard content attribute:

```js
canvas.poster = "fallback.png"  // author sets it
canvas.poster // → "fallback.png"  // getter returns what was set
```

Archival tools (browser "Save As", Safari Web Archive, WARC capture, Wayback Machine) would then handle bitmap capture during serialisation:

```js
// Pseudocode for archival serialisation
if (canvas has been drawn to && canvas is origin-clean) {
  if (!canvas.hasAttribute('poster')) {
    canvas.setAttribute('poster', canvas.toDataURL())
  }
}
```

This approach wins on multiple fronts:

**Simple semantics.** The attribute behaves exactly like video's `poster` or img's `src`.

**Author control.** Authors can provide a curated image: optimised for file size, adjusted for print, different aspect ratio, simplified for clarity.

**Archival captures live state when needed.** The complexity of "snapshot the current bitmap" lives in the archival path, where it belongs.

**Tainted canvases degrade gracefully.** A cross-origin-tainted canvas simply doesn't get an auto-captured poster during archival.

### Explicit Capture Pattern

For authors who want to capture at a specific moment:

```js
// After drawing is "complete"
canvas.poster = canvas.toDataURL()

// Or with format control
canvas.poster = canvas.toDataURL('image/webp', 0.8)
```

## Alternatives considered

### A `static-src` Attribute

Instead of reusing the `poster` name, we could introduce `static-src` or `fallback-src`. But `poster` has established semantics from video that map well to canvas, and using a familiar name aids understanding.

### Extending Fallback Content Semantics

We could redefine fallback content to render in more contexts (not just when canvas fails). But this would break existing pages that use fallback for accessibility purposes and don't expect it to render visually.

### A CSS Property

Something like `canvas-poster: url(snapshot.png)` would work but feels like the wrong layer. This is fundamentally about the element's content, not its presentation. HTML attributes are the right place for content-related fallbacks.

### Automatic Bitmap Serialisation

Browsers could automatically serialise the canvas bitmap when saving pages. Some do this already for printing. But this fails when scripts haven't run, requires execution for archiving, and doesn't help with progressive loading.

### Just use fallback content with an image

You *can* put an `<img>` in the fallback content:

```html
<canvas width="400" height="300">
  <img src="snapshot.png" width="400" height="300" alt="Chart">
</canvas>
```

But this conflates two purposes. The fallback content exists for when canvas isn't supported or for non-visual media (screen readers). Adding a visual representation there muddies the semantics. What if you need *both* an accessible data table *and* a visual snapshot? The image would need to go inside or replace the table.

The rendering model also differs. Fallback content flows as normal HTML, not as a replaced element sized to the canvas.

### "Canvas is for dynamic content, if you want static, use `<img>`"

This misses the reality that dynamic content often needs static representation. A video is dynamic content, but we don't say "if you want a preview frame, don't use video." We provide `poster` because static and dynamic are complementary, not exclusive.

### "Just run JavaScript during archiving/printing"

Running JavaScript for archiving or printing creates its own problems:

- Non-determinism: the canvas might render differently each time
- External dependencies: scripts might fetch data from APIs that no longer exist
- Performance: executing arbitrary JavaScript is slow and potentially dangerous
- Timeouts: how long do you wait for the canvas to be "done"?

A declarative poster avoids all of these issues.

## Implementation Considerations

### For Browser Vendors

**Parsing.** Trivial: just another URL attribute on HTMLCanvasElement, exactly like `poster` on HTMLVideoElement.

**Image Loading.** Reuse existing image loading infrastructure. The poster image loads like any other image resource, with standard caching, CORS handling, and decode behaviour.

**CSP Integration.** The poster image should fall under the `img-src` directive, matching how video poster is handled.

**Resource Timing.** The poster fetch should create a `PerformanceResourceTiming` entry with `initiatorType: "canvas"`.

**Rendering.** The main complexity is managing the transition from poster to bitmap:

1. Track whether any draw operation has occurred on the canvas
2. When rendering, check if the canvas has been drawn to
3. If not, render the poster image (if present) scaled to canvas dimensions
4. If yes, render the canvas bitmap as normal

**Memory.** Potentially two bitmaps in memory briefly (poster + canvas bitmap during transition). But the poster image can be released once drawing begins.

**Serialisation (archiving/printing).** When serialising a canvas:
- If the canvas has been drawn to, serialise the current bitmap
- If not, serialise the poster image URL (or embed the image data)

### Specification Changes

The HTML specification would need:

1. A new `poster` content attribute on the canvas element
2. A corresponding `poster` IDL attribute on HTMLCanvasElement
3. Updated rendering rules in the "in a visual medium" section
4. Clarification of serialisation behaviour for archival formats

The changes are localised and don't affect the canvas drawing APIs, context handling, or security model.

### CSS Styling

The poster image should respect the canvas element's `object-fit` and `object-position` properties:

```css
canvas {
  object-fit: contain;
  object-position: center top;
}
```

There's no need for a pseudo-element like `canvas::poster`. Video doesn't have one either.

## Accessibility, Internationalization, Privacy, and Security Considerations

### Accessibility

The poster attribute complements but does not replace fallback content. Screen readers and other assistive technologies should continue to access the fallback content for semantic information. The poster provides a visual static representation only.

### Internationalization

No specific i18n considerations. The poster is an image URL with no text content.

### Privacy

No new privacy concerns. The poster image follows the same loading and caching behaviour as other images.

### Security

The poster image follows standard image loading rules:

- Same CORS handling as `<img src>`
- Tainting rules apply if the poster is cross-origin
- No new attack surface beyond what `<img>` already provides

One consideration: should a cross-origin poster affect the canvas's origin-clean flag? Probably not, since the poster is conceptually separate from the canvas bitmap and isn't accessible via `getImageData()`. But this would need specification.

## Stakeholder Feedback / Opposition

[Pending outreach to browser vendors and web archiving community]

- Chromium: No signals
- Mozilla: No signals  
- WebKit: No signals
- Internet Archive: No signals

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- [Pending]

Thanks to the following for their work on related problems:

- The `<video>` element's `poster` attribute, which establishes the pattern this proposal follows
- Web archiving initiatives (Internet Archive, national libraries) that highlight the importance of static representations
- PDF.js and other canvas-heavy applications that demonstrate the scope of the problem
