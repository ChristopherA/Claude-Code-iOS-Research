# Claude iOS App: Clickable URLs, Deep Links, and URL Scheme Research

**Date:** 2026-02-09
**Status:** Research findings based on publicly available information. No official Anthropic documentation directly addresses most of these questions, so findings are assembled from reverse-engineering analyses, security research, developer reports, and iOS platform knowledge.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Markdown Link Rendering in the Claude iOS App](#markdown-link-rendering)
3. [iOS URL Schemes and Deep Links](#ios-url-schemes)
4. [Security Restrictions and Sandboxing](#security-restrictions)
5. [Claude Code on iOS (Web vs. Native)](#claude-code-ios)
6. [Artifact Rendering and Deep Links](#artifacts)
7. [Claude's Own URL Scheme (`claude://`)](#claude-url-scheme)
8. [Practical Implications and What Works](#practical-implications)
9. [Sources](#sources)

---

## 1. Executive Summary <a name="executive-summary"></a>

| Question | Answer |
|---|---|
| Does Claude iOS render markdown links as clickable? | **Very likely yes** — Claude's responses use markdown, and both the native iOS app and web interface render formatted output. Web search citations are confirmed clickable inline. |
| Can bare URLs like `https://example.com` become tappable? | **Very likely yes** — Standard iOS text rendering (`UITextView`/SwiftUI `Text`) auto-detects URLs via `NSDataDetector` or native Markdown support (iOS 15+). |
| Can you tap a link and have it open Safari or another app? | **Yes for HTTPS links** — Standard iOS behavior opens HTTPS links in Safari or the default browser. Universal Links may intercept and open the relevant app. |
| Do custom URL schemes (`github://`, `shortcuts://`) work? | **Unknown/Unlikely in native app** — No evidence that the Claude iOS native app allows tapping non-HTTPS URL schemes. The app may sanitize or ignore them. |
| Can artifacts embed clickable deep links? | **No** — Artifacts run in a heavily sandboxed iframe that explicitly blocks external navigation, popups, and link-outs. |
| Does Claude have its own `claude://` URL scheme? | **Not publicly documented** — No `claude://` scheme has been confirmed. iOS Shortcuts integration uses App Intents, not URL schemes. |

---

## 2. Markdown Link Rendering in the Claude iOS App <a name="markdown-link-rendering"></a>

### 2.1 How Claude Formats Responses

Claude's responses are formatted in Markdown. On all platforms (web, desktop, iOS, Android), the client application is responsible for parsing and rendering this Markdown into rich text with formatting, code blocks, lists, and links.

### 2.2 Native iOS App Rendering

Anthropic's Claude iOS app (App Store ID: `6473753684`) is a native iOS application, not a thin web wrapper. The app renders Claude's Markdown responses into native UI components.

On iOS, there are standard mechanisms for rendering clickable links in text:

- **SwiftUI `Text` with Markdown (iOS 15+):** SwiftUI natively supports inline Markdown in `Text` views, including tappable links. A `[text](url)` pattern renders as a blue, tappable hyperlink.
- **`UITextView` with `dataDetectorTypes`:** UIKit's `UITextView` can automatically detect and make tappable: URLs, phone numbers, addresses, and other data types. Setting `dataDetectorTypes = .link` enables this.
- **`NSAttributedString` with `.link` attribute:** Custom link styling can be applied to attributed strings, making any text span tappable with a URL destination.

**Assessment:** Given that rendering Markdown links as clickable hyperlinks is trivially supported in both SwiftUI and UIKit, and that Claude's web search feature explicitly produces "inline links" that are "clickable in context" (per BrightEdge's analysis of Claude Search), it is highly probable that the Claude iOS native app renders both:

- **Markdown-formatted links** (`[text](https://example.com)`) as tappable hyperlinks
- **Bare URLs** (`https://example.com`) as auto-detected tappable links

### 2.3 What Happens When You Tap a Link

When a user taps a link in an iOS app, the default behavior depends on how the developer implemented it:

1. **Open in Safari (external):** The link opens in the system Safari browser. This is the default for `UIApplication.shared.open(url)`.
2. **Open in `SFSafariViewController` (in-app browser):** The link opens in an in-app Safari view that looks like a mini browser. This is Apple's recommended approach for keeping users in the app.
3. **Open in `WKWebView` (custom in-app browser):** The app renders the page in a custom web view.

**No public documentation or user reports specify which method the Claude iOS app uses.** However, the standard iOS pattern for AI chat apps (ChatGPT, Gemini, etc.) is to use `SFSafariViewController` for HTTP/HTTPS links, keeping the user in the app while still leveraging Safari's engine.

### 2.4 Universal Links Behavior

If a tapped HTTPS URL matches a Universal Link registered by an installed app (e.g., `https://github.com/user/repo` opens in the GitHub app), iOS will intercept the tap and open the target app instead of Safari. This happens at the OS level regardless of how the Claude app handles the tap — as long as the URL is opened through standard iOS URL-handling APIs.

**Key point:** Universal Links work even from in-app browsers (`SFSafariViewController`), but they do NOT work from `WKWebView` for navigations triggered by the web view itself (they require a user tap and a standard URL open call).

---

## 3. iOS URL Schemes and Deep Links <a name="ios-url-schemes"></a>

### 3.1 How Custom URL Schemes Work on iOS

iOS allows apps to register custom URL schemes (e.g., `github://`, `shortcuts://`, `x-callback-url://`). When any app calls `UIApplication.shared.open(URL(string: "github://repo/user/project")!)`, iOS checks if any installed app handles that scheme and opens it.

Common iOS URL schemes:

| Scheme | App | Example |
|---|---|---|
| `https://` | Safari / Universal Links | `https://github.com/user/repo` |
| `github://` | GitHub | `github://repo/user/project` (unofficial) |
| `shortcuts://` | Apple Shortcuts | `shortcuts://run-shortcut?name=MyShortcut` |
| `x-callback-url://` | Various apps | `x-callback-url://action?param=value` |
| `prefs://` or `App-prefs://` | iOS Settings | `App-prefs://` (unreliable, undocumented) |
| `maps://` | Apple Maps | `maps://?q=coffee` |
| `tel://` | Phone | `tel://5551234567` |
| `sms://` | Messages | `sms://5551234567` |
| `mailto://` | Mail | `mailto://user@example.com` |

### 3.2 Can Claude Output These Schemes?

Claude (the language model) can output any text, including custom URL schemes. The question is whether the Claude iOS app's Markdown renderer makes them tappable.

**For standard schemes (`https://`, `mailto:`, `tel:`):**
- iOS `NSDataDetector` automatically detects `https://`, `http://`, `mailto:`, `tel:`, and similar standard schemes.
- These are almost certainly tappable in the Claude iOS app.

**For custom schemes (`github://`, `shortcuts://`, etc.):**
- `NSDataDetector` does **not** detect custom URL schemes by default. It only detects standard schemes.
- Custom URL schemes would only be tappable if:
  1. They appear inside a Markdown link (`[Open GitHub](github://repo/user/project)`) AND the Markdown renderer preserves the URL, OR
  2. The app uses a custom URL detection regex that includes non-standard schemes.

**Assessment:** Custom URL schemes in Claude's responses are **unlikely to be automatically tappable** in the native iOS app unless they are wrapped in Markdown link syntax, and even then, the app may sanitize non-HTTP URLs for security reasons.

### 3.3 Security Considerations for Custom Schemes

Apps that allow arbitrary URL scheme opening expose users to risks:
- **Phishing:** A malicious `tel://` link could initiate a phone call.
- **Data exfiltration:** Custom schemes can pass data to other apps.
- **App exploitation:** Some apps have vulnerabilities in their URL scheme handlers.

For this reason, many apps sanitize links in user-facing content to only allow `https://` URLs. It is reasonable to expect that Anthropic applies similar sanitization in the Claude iOS app.

### 3.4 Universal Links as the Best Path

Universal Links (`https://` URLs that iOS intercepts and opens in an app) are the most reliable way to open another app from Claude's output because:

1. They use standard `https://` scheme, which is always rendered as tappable.
2. iOS handles the interception at the OS level — the Claude app doesn't need to do anything special.
3. If the target app isn't installed, the link falls back to opening in Safari.

**Examples that should work:**
- `https://github.com/user/repo` → Opens in GitHub app (if installed)
- `https://twitter.com/user` or `https://x.com/user` → Opens in X/Twitter app
- `https://www.youtube.com/watch?v=ID` → Opens in YouTube app
- `https://maps.apple.com/?q=coffee` → Opens in Apple Maps

**Caveat:** Universal Links are suppressed in some contexts (e.g., when the link is in the same domain, when pasted into Safari's address bar, or when opened in a `WKWebView` without standard navigation). If the Claude iOS app opens links in an `SFSafariViewController`, Universal Links should work. If it uses a `WKWebView`, they may not.

---

## 4. Security Restrictions and Sandboxing <a name="security-restrictions"></a>

### 4.1 Claude iOS App (Native)

There is **no public documentation** about the Claude iOS app's internal URL handling, sanitization, or security policies. The following is inferred from iOS development best practices and observed behavior:

**Likely restrictions:**
- URL scheme allowlisting: The app likely only opens `https://`, `http://`, `mailto:`, and `tel:` schemes, blocking arbitrary custom schemes.
- No `javascript:` or `data:` URLs rendered.
- Links in Claude's responses likely go through a sanitization layer before being made tappable.

**Likely permissions:**
- HTTPS links are rendered as tappable hyperlinks.
- Tapping an HTTPS link opens it via `SFSafariViewController` (in-app) or Safari (external).
- Universal Links work for HTTPS URLs if the OS intercepts them.

### 4.2 Claude Web Interface (claude.ai)

The web interface at `claude.ai` renders Claude's Markdown responses in HTML. Links are rendered as standard `<a href="...">` tags. In a browser on iOS:

- HTTPS links work as standard web hyperlinks.
- Custom URL schemes in `<a>` tags may or may not work depending on the browser:
  - **Safari on iOS:** Generally allows custom URL schemes from `<a>` tags (with a confirmation dialog for some schemes). Safari prompts: "Open this page in [App]?" before opening.
  - **In-app browsers (WKWebView/SFSafariViewController):** May block custom schemes.
- `claude.ai` likely does **not** sanitize URLs in the same way the native app does, since the browser handles link clicks natively.

### 4.3 Known Claude.ai CSP Headers

From reverse-engineering analyses (Reid Barber, Simon Willison), the claude.ai website serves strict Content Security Policies. The artifact sandbox on `claudeusercontent.com` has:

- `default-src https://www.claudeusercontent.com`
- `script-src 'unsafe-eval' 'unsafe-inline'` with allowlisted CDNs
- `frame-ancestors` limited to `claude.ai`, `preview.claude.ai`, `claude.site`, `feedback.anthropic.com`
- Restricted `connect-src` and `form-action`

These CSP restrictions apply to **artifacts**, not to the main chat interface where Claude's text responses with links appear.

---

## 5. Claude Code on iOS (Web vs. Native) <a name="claude-code-ios"></a>

### 5.1 Architecture

When using Claude Code from the iOS app, you are using a **web-based interface** (`claude.ai/code`) rendered within the iOS app. The iOS app acts as a thin client connecting to Anthropic's cloud infrastructure. Each coding task runs on an ephemeral Ubuntu VM.

### 5.2 Link Behavior in Claude Code Output

Claude Code's output (terminal-style) is different from the standard Claude chat interface:

- **File paths:** Displayed as plain text. Known to NOT be clickable (this is a documented pain point — see GitHub issue #13008 requesting OSC 8 hyperlinks).
- **URLs in terminal output:** Likely rendered as plain text, not clickable.
- **Links in Claude Code's conversational responses:** May be rendered as clickable if the response section uses Markdown rendering (as opposed to terminal/code-block rendering).

### 5.3 Web Interface vs. Native App Differences

| Feature | Native iOS App | Web (Safari on iOS) |
|---|---|---|
| Chat message links | Rendered via native iOS components | Rendered via HTML `<a>` tags |
| Custom URL schemes | Likely blocked/sanitized | May work (Safari shows confirmation) |
| Universal Links | Work if using standard URL open APIs | Work from Safari navigation |
| Artifacts | Same sandbox restrictions | Same sandbox restrictions |
| Claude Code output | Terminal-style, limited link support | Same (web-based rendering) |

---

## 6. Artifact Rendering and Deep Links <a name="artifacts"></a>

### 6.1 Artifact Sandbox Architecture

Claude Artifacts render user-generated HTML/CSS/JavaScript inside a sandboxed iframe on `claudeusercontent.com`. This sandbox is the most restricted environment in the Claude ecosystem:

**Confirmed restrictions:**
- **No external navigation:** Artifacts cannot "link out to other pages" (confirmed by Simon Willison).
- **No API calls:** Artifacts cannot make `fetch()` calls to external endpoints.
- **No form submissions:** Artifacts cannot submit forms to external URLs.
- **No popups:** The sandbox likely does not include `allow-popups` in the iframe sandbox attribute.
- **No top navigation:** The sandbox does not include `allow-top-navigation`, preventing the artifact from navigating the parent frame.
- **Strict CSP:** `connect-src`, `form-action`, and `default-src` are locked down to `claudeusercontent.com` and allowlisted CDN domains.

### 6.2 Can Artifacts Embed Clickable Deep Links?

**No.** Given the documented restrictions:

1. An `<a href="github://repo/user/project">` link inside an artifact would be blocked by the sandbox. The iframe cannot navigate to external URLs or open popups.
2. An `<a href="https://github.com/user/repo">` link would also be blocked because the sandbox prevents top-level navigation.
3. Even `window.open()` calls are blocked without `allow-popups` in the sandbox.
4. JavaScript `location.href` changes are blocked without `allow-top-navigation`.

**The only partial workaround is MCP integration:** Artifacts can now interact with external services through the Model Context Protocol (MCP), but this is a controlled server-side integration, not a client-side link-clicking mechanism.

### 6.3 Artifacts on iOS Specifically

Artifacts render in the same sandboxed iframe on both the web and iOS app. The restrictions are identical. The iframe sandbox is enforced by the browser engine (WebKit on iOS), which applies the same security model regardless of whether the iframe is displayed in Safari, `SFSafariViewController`, or `WKWebView`.

---

## 7. Claude's Own URL Scheme (`claude://`) <a name="claude-url-scheme"></a>

### 7.1 Is There a `claude://` URL Scheme?

**Not publicly documented.** Extensive searching found no official documentation, community discussion, or reverse-engineering analysis that confirms a `claude://` URL scheme for the Claude iOS app.

### 7.2 Alternatives for Opening the Claude App

Anthropic provides integration with iOS through **App Intents**, not URL schemes:

- **"Ask Claude" App Intent:** Can be used in iOS Shortcuts to send a prompt to Claude.
- **Spotlight:** Typing "Ask Claude" in Spotlight search opens the Claude app with a query interface.
- **Share Sheet:** Selecting text and sharing it with Claude sends it to the app.
- **Siri:** Saying "Ask Claude" triggers the App Intent via Siri.
- **Widgets:** The Claude widget on the Home Screen provides quick access to chat and voice.
- **Lock Screen Controls:** "Analyze Photo with Claude" can be added as a Lock Screen control.

These App Intents are the modern iOS replacement for URL schemes. Apple has been pushing developers toward App Intents and away from URL schemes for inter-app communication since iOS 16.

### 7.3 Claude Code Deep Linking

There is a feature request (GitHub issue #10366) for deep linking into Claude Code sessions via `vscode://anthropic.claude-code/chat/{chatId}`, but this is for the VS Code extension, not the iOS app. This feature does not exist yet.

---

## 8. Practical Implications and What Works <a name="practical-implications"></a>

### 8.1 What Almost Certainly Works

1. **HTTPS links in Claude's responses** → Tappable, opens in Safari or in-app browser.
2. **Universal Links** (HTTPS URLs for apps like GitHub, YouTube, Twitter) → Tappable, iOS may open the relevant app if installed.
3. **`mailto:` links** → Likely tappable, opens Mail or default email app.
4. **`tel:` links** → Likely tappable, initiates a phone call.
5. **Web search citations** → Confirmed clickable inline links.

### 8.2 What Probably Does Not Work

1. **Custom URL schemes** (`github://`, `shortcuts://`, etc.) in plain text → Not auto-detected by iOS.
2. **Custom URL schemes in Markdown links** → May be sanitized by the Claude iOS app.
3. **Artifacts with embedded links** → Blocked by iframe sandbox.
4. **`javascript:` URLs** → Blocked.
5. **`data:` URLs** → Blocked.
6. **`app-settings://` or `prefs://`** → Not standard, not auto-detected, likely blocked.

### 8.3 Workarounds

**For opening other apps from Claude's output:**
1. **Use Universal Links:** Ask Claude to provide standard `https://` URLs for services. iOS will intercept and open the relevant app if installed. Example: `https://github.com/user/repo` will open in GitHub.app.
2. **Copy-paste:** Copy a custom URL scheme from Claude's response and paste it into Safari's address bar. Safari will attempt to open the corresponding app.
3. **iOS Shortcuts integration:** Use the "Ask Claude" App Intent within a Shortcut that then opens another app or URL scheme as a subsequent step.
4. **Apple Maps links:** `https://maps.apple.com/?q=...` links should open in Apple Maps.

**For automation workflows:**
1. Create an iOS Shortcut that: asks Claude a question via App Intent, parses the response for URLs, and opens them using the "Open URLs" action (which supports custom schemes).
2. This is the most reliable way to chain Claude's output into opening other apps.

### 8.4 Testing Recommendations

Since no definitive documentation exists, the following tests should be performed on a physical iPhone with the Claude iOS app installed:

1. Ask Claude: "Give me a markdown link to https://github.com" — Verify the link is tappable and observe where it opens (Safari vs. in-app browser).
2. Ask Claude: "Give me these URLs as clickable links: github://repo/anthropics/claude-code and shortcuts://run-shortcut?name=Test" — Test if custom schemes are rendered as tappable.
3. Ask Claude to create an artifact with `<a href="https://google.com">Click me</a>` — Verify the link is blocked by the sandbox.
4. Ask Claude to provide a Universal Link like `https://www.youtube.com/watch?v=dQw4w9WgXcQ` — Verify iOS intercepts and opens YouTube.
5. Test the same scenarios on claude.ai in mobile Safari and compare behavior.

---

## 9. Sources <a name="sources"></a>

### Official Anthropic Documentation
- [Using Claude with iOS Apps](https://support.claude.com/en/articles/11869619-using-claude-with-ios-apps)
- [Using Claude App Intents, Shortcuts, and Widgets on iOS](https://support.claude.com/en/articles/10263469-using-claude-app-intents-shortcuts-and-widgets-on-ios)
- [What are artifacts and how do I use them?](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them)
- [Claude is producing links that don't work](https://support.claude.com/en/articles/8241188-claude-is-producing-links-that-don-t-work-and-falsely-claiming-that-it-has-sent-emails-or-produced-external-documents-what-s-going-on)
- [Making Claude Code more secure and autonomous (Sandboxing)](https://www.anthropic.com/engineering/claude-code-sandboxing)
- [Claude Code Sandboxing Docs](https://code.claude.com/docs/en/sandboxing)

### Reverse-Engineering and Security Research
- [Reverse engineering Claude Artifacts — Reid Barber](https://www.reidbarber.com/blog/reverse-engineering-claude-artifacts)
- [Everything I built with Claude Artifacts this week — Simon Willison](https://simonwillison.net/2024/Oct/21/claude-artifacts/)
- [Simon Willison on Claude Artifacts (tag)](https://simonwillison.net/tags/claude-artifacts/)
- [Bulk Exploitation of Verified Users in the Claude.ai Artifact System — sobele.com](https://www.sobele.com/en/blogs/security-breach-analysis/bulk-exploitation-of-verified-users-in-the-claudeai-artifact-system)
- [How Anthropic built Artifacts — Gergely Orosz / Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/how-anthropic-built-artifacts)

### Claude Code Link Issues (GitHub)
- [#13008 — OSC 8 Hyperlinks for Clickable File Paths](https://github.com/anthropics/claude-code/issues/13008)
- [#10846 — Markdown links in VS Code extension not clickable](https://github.com/anthropics/claude-code/issues/10846)
- [#16056 — Markdown file links not clickable on macOS](https://github.com/anthropics/claude-code/issues/16056)
- [#10366 — Deep linking support for Claude Code](https://github.com/anthropics/claude-code/issues/10366)

### iOS Deep Linking General Reference
- [iOS Deep Linking: URL Schemes vs Universal Links — byby.dev](https://byby.dev/ios-deep-linking)
- [Deep linking and URL scheme in iOS — Benoit Pasquier](https://benoitpasquier.com/deep-linking-url-scheme-ios/)
- [iOS App URL Schemes for Shortcuts — GitHub Gist](https://gist.github.com/roachhd/c5d9c9dee45c73568daff94b343f5170)
- [Understanding iOS Universal Links — Josh Dick](https://joshdick.net/2017/01/08/understanding_ios_universal_links.html)

### iOS Development (Text/Link Rendering)
- [How to make tappable links in NSAttributedString — Hacking with Swift](https://www.hackingwithswift.com/example-code/system/how-to-make-tappable-links-in-nsattributedstring)
- [Hyperlinks in SwiftUI Text — Swift UI Recipes](https://swiftuirecipes.com/blog/hyperlinks-in-swiftui-text)
- [dataDetectorTypes — Apple Developer Documentation](https://developer.apple.com/documentation/uikit/uitextview/1618607-datadetectortypes)

### Iframe Sandbox and CSP Reference
- [Content-Security-Policy: sandbox directive — MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/sandbox)
- [iframe sandbox attribute — MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe)
- [Play safely in sandboxed IFrames — web.dev](https://web.dev/articles/sandboxed-iframes)
