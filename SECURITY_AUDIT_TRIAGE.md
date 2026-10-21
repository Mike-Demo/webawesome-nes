# Security audit triage — 2026-10-21

Parent: https://github.com/Mike-Demo/webawesome-nes/issues/1

This is the requested fallback backlog because this session has no issue-creation tool. The three issue titles and bodies below are ready to file; no child issues have been created. No implementation, dependency, or workflow changes are included.

Scope: the parent reports two confirmed medium findings, six needs-validation leads, and one rejected candidate. This triage uses the supplied audit and static source inspection only; no component code was executed and no deployed endpoint was probed. Several fields in the parent itself end in `…`; missing-fact quotations below preserve that truncation exactly, not an inferred completion.

## Priority and grouping

| Order | Finding | Risk and reachability | Fix scope |
| --- | --- | --- | --- |
| 1 | `wa-markdown-unsanitized-html-injection` — medium | The component summary explicitly invites user-generated content, making lower-trust input a plausible use. Such input reaches HTML insertion without sanitization. Documentation already warns against unsanitized input; no actual lower-trust host binding is established here. | One shared injection closure can sanitize both synchronous and asynchronous parse results. Preserve normal markdown and extension behavior. |
| 2 | `wa-include-allowscripts-bypass-via-event-handlers` — medium | Active content survives the default `allowScripts = false` setting. A lower-trust caller must control included markup; both source and docs warn that included content must be trusted. The flag nevertheless incompletely enforces its apparent boundary. | Sanitize before insertion across full-file, fetched-fragment, and same-page fragment paths. More branches need coverage than the markdown fix. |

Both findings concern active HTML reaching the DOM, but they have separate root causes and insertion paths: missing markdown-output sanitization versus incomplete enforcement of include's script-control flag. Keep two remediation issues; do not merge the six unvalidated leads into either. Severity is unchanged from the audit. Reachability is conditional, not evidence of a compromised deployment; risk ranks first, with fix size/safety secondary.

## Ready-to-file issue 1

### Title

[security][medium] Sanitize wa-markdown output before DOM insertion

### Body

Parent audit: https://github.com/Mike-Demo/webawesome-nes/issues/1

Fingerprint: `wa-markdown-unsanitized-html-injection`

Priority: 1 of 2 confirmed findings. Severity: medium (as reported by the audit).

#### Evidence

Static source references at revision `b15a7a72c92aa711a5c745fa714290bba3dd939d`:

- [Component summary, lines 10–13](https://github.com/Mike-Demo/webawesome-nes/blob/b15a7a72c92aa711a5c745fa714290bba3dd939d/packages/webawesome/src/components/markdown/markdown.ts#L10-L13) advertises user-generated content.
- [Parsing and injection, lines 146–187](https://github.com/Mike-Demo/webawesome-nes/blob/b15a7a72c92aa711a5c745fa714290bba3dd939d/packages/webawesome/src/components/markdown/markdown.ts#L146-L187): `sharedMarked.parse(dedented)` feeds `inject()`, which passes `renderedHtml` directly to `createContextualFragment()` and appends the result. Both string and Promise results use that closure; no sanitization occurs.
- [Existing documentation warning, lines 41–43](https://github.com/Mike-Demo/webawesome-nes/blob/b15a7a72c92aa711a5c745fa714290bba3dd939d/packages/webawesome/docs/docs/components/markdown.md#L41-L43) already says not to use unsanitized user input.

Marked supports raw HTML. Consequently, handler attributes and executable URLs in lower-trust markdown can reach the host DOM. The warning mitigates intended use but does not sanitize output or reconcile the summary's claim. This is source-established; browser execution and a real lower-trust deployment were not demonstrated by this audit.

#### Root cause

The markdown-to-HTML conversion is treated as safe DOM content without an allowlist sanitization boundary. Parsing markdown is not HTML sanitization.

#### Smallest fix

Sanitize parsed HTML with a maintained allowlist sanitizer (for example, DOMPurify) inside the shared `inject()` closure, before `createContextualFragment()`, covering both synchronous and asynchronous Marked results. Keep generation checks and source-script preservation intact. Until sanitization is implemented, remove the user-generated-content claim from the component summary; that wording change alone is not remediation.

#### Acceptance criteria

- Rendered raw HTML cannot introduce executable handlers, dangerous URL schemes, or other active content through either synchronous or asynchronous parsing.
- Ordinary markdown, permitted HTML, existing extensions, and rerender/stale-result handling retain their supported behavior.
- Browser regression tests cover handler attributes and dangerous links for both result types.
- Component summary and documentation describe the resulting trust boundary consistently.

## Ready-to-file issue 2

### Title

[security][medium] Enforce wa-include allow-scripts=false before insertion

### Body

Parent audit: https://github.com/Mike-Demo/webawesome-nes/issues/1

Fingerprint: `wa-include-allowscripts-bypass-via-event-handlers`

Priority: 2 of 2 confirmed findings. Severity: medium (as reported by the audit).

#### Evidence

Static source references at revision `b15a7a72c92aa711a5c745fa714290bba3dd939d`:

- [Trust warning and default flag, lines 26–42](https://github.com/Mike-Demo/webawesome-nes/blob/b15a7a72c92aa711a5c745fa714290bba3dd939d/packages/webawesome/src/components/include/include.ts#L26-L42): `allowScripts` defaults to false.
- [Fragment cloning, lines 52–62](https://github.com/Mike-Demo/webawesome-nes/blob/b15a7a72c92aa711a5c745fa714290bba3dd939d/packages/webawesome/src/components/include/include.ts#L52-L62) imports cloned children without removing handler attributes or active URLs.
- [Same-page fragment insertion, lines 74–85](https://github.com/Mike-Demo/webawesome-nes/blob/b15a7a72c92aa711a5c745fa714290bba3dd939d/packages/webawesome/src/components/include/include.ts#L74-L85) inserts the clone and returns before the fetched-content script branch.
- [Fetched insertion and script gate, lines 107–124](https://github.com/Mike-Demo/webawesome-nes/blob/b15a7a72c92aa711a5c745fa714290bba3dd939d/packages/webawesome/src/components/include/include.ts#L107-L124): fragment content is cloned; otherwise `this.innerHTML = file.html`. Only explicit re-execution of `<script>` elements is gated by `this.allowScripts`.
- [Documentation warning, lines 23–25](https://github.com/Mike-Demo/webawesome-nes/blob/b15a7a72c92aa711a5c745fa714290bba3dd939d/packages/webawesome/docs/docs/components/include.md#L23-L25) warns about untrusted content but describes scripts as running when `allow-scripts` is set.

Inline event handlers and `javascript:` URLs survive these insertion paths even with the flag unset. Exploitation requires control over included content; the trust warning does not make the flag complete. The audit establishes the source path, not a live-browser reproduction or an affected deployment.

#### Root cause

The flag controls explicit script-element re-execution, not all active content. Neither full-file insertion nor fragment cloning sanitizes the markup before connecting it to the live DOM.

#### Smallest fix

When `allowScripts` is false, sanitize a detached parsed include before insertion: remove script elements, every `on*` attribute, and `javascript:` URLs. Apply the same boundary to full-file includes and both fetched and same-page fragment/template clones, without mutating the original same-page source. Use robust HTML/URL handling rather than raw-string substitutions; ensure other active-content forms cannot bypass the documented boundary. Document that `allow-scripts` governs active content, not only `<script>` elements. Preserve the intended opt-in behavior when it is true.

#### Acceptance criteria

- With the default flag, full-file, fetched-fragment, and same-page fragment/template paths block scripts, handlers, and executable URLs before connection to the live DOM.
- Regression tests exercise those paths, including nested template content and encoded/obfuscated dangerous URLs.
- Safe markup, fragment selection, request/error/load behavior, and supported `allow-scripts=true` behavior remain intact.
- Documentation explains the boundary and retains appropriate source-trust guidance.

## Ready-to-file issue 3

### Title

[security][needs-validation] Audit leads to validate

### Body

Parent audit: https://github.com/Mike-Demo/webawesome-nes/issues/1

These are leads, not confirmed vulnerabilities. No severity is assigned. Record evidence for each missing fact before promoting or dismissing a lead. Text under “Exact missing fact” is copied from the parent, including its existing `…` truncations; obtain the untruncated audit text where necessary rather than assuming the missing qualifications. Validate only with authorized host/deployment evidence or controlled tests.

- [ ] **Autoloader takes its code-loading base path (and the FA kit code) from the first arbitrary DOM element carrying the attribute**
  - Fingerprint: `autoloader-base-path-from-any-element`
  - Source lead: first-match attribute lookup and caching can influence dynamic component loading.
  - Exact missing fact:
    > Whether a real host using the autoloader (1) lets a lower-trust content author insert arbitrary elements/attributes (data-webawesome / data-fa-kit-code) without already having script capability, and (2) exposes a same-origin path writable by that author from which attacker-controlled wa-* component modules could be imported; neither condition is d…
  - Resolution evidence: host content permissions and a corresponding attacker-writable same-origin module path; recover the truncated qualification.

- [ ] **Claude Code workflow feeds untrusted PR content to an agent job holding contents:write / pull-requests:write; only a soft prompt says not to modify files**
  - Fingerprint: `claude-workflow-untrusted-content-agent-write`
  - Source lead: write job permissions and no explicit tool allowlist; actual review-mode action behavior is external.
  - Exact missing fact:
    > Whether anthropics/claude-code-action@v1, as invoked here (no explicit tool allowlist in claude.yml), internally downscopes available tools or the token for review-style invocations so file/PR writes are impossible, or whether the write-capable GITHUB_TOKEN and agent tools remain fully available to prompt-injected instructions; that behavior lives…
  - Resolution evidence: the resolved action implementation and effective tool/token permissions for this invocation; recover the truncated qualification.

- [ ] **Docs build reads files via path.join(baseDir, src) with src taken verbatim from content elements (no confinement)**
  - Fingerprint: `docs-code-examples-unconfined-path-read`
  - Source lead: two verbatim-attribute file reads; publication and build-environment exposure remain unknown.
  - Exact missing fact:
    > Whether untrusted contributor PRs trigger a docs build (production or Vercel preview) in an environment where BASE_DIR-relative ../ traversal can reach sensitive files AND the generated output exposing the read file contents is visible to the contributor; the repo shows only the build command (vercel.json/vercel.sh), not preview-build exposure or …
  - Resolution evidence: PR build triggers, accessible build filesystem boundaries, and contributor visibility of generated output, without retrieving actual secrets; recover the truncated qualification.

- [ ] **Default icon library interpolates the name attribute verbatim into the fetch URL (path traversal / query manipulation, kit token in query)**
  - Fingerprint: `wa-icon-name-unencoded-url-construction`
  - Source lead: icon URL construction lacks encoding or an allowlist; attribute ownership is deployment-dependent.
  - Exact missing fact:
    > Whether any host application binds untrusted/lower-trust data (user input, CMS fields) to wa-icon's name/family/variant attributes; the repo contains no such binding, and without it the value is set by the same developer principal who owns the page.
  - Resolution evidence: a concrete lower-trust input-to-attribute binding and its authorization boundary.

- [ ] **wa-icon inlines fetched SVG into the page without stripping script, event handlers, or active content**
  - Fingerprint: `wa-icon-unsanitized-svg-adoption`
  - Source lead: fetched SVG is parsed, re-parsed, and adopted into the page without active-content removal.
  - Exact missing fact:
    > Whether any supported/deployment use of wa-icon (src, custom library resolver, or self-hosted icon path) delivers SVG files authored by a principal lower-trust than the host developer who chose that source (e.g. user-uploaded or third-party-contributed icon sets) — the repo itself documents src as developer-trusted content (icon.ts:143-144) and co…
  - Resolution evidence: SVG provenance and an actual trust boundary for a supported host use; recover the truncated qualification.

- [ ] **wa-page interpolates mobile-breakpoint verbatim into an unsafeHTML &lt;style&gt; block (CSS/HTML injection)**
  - Fingerprint: `wa-page-mobile-breakpoint-style-breakout`
  - Source lead: nonnumeric breakpoint text passes through `toLength()` into the unsafe style insertion.
  - Exact missing fact:
    > Whether any host application binds untrusted/lower-trust data (theme/config/CMS/user input) to wa-page's mobile-breakpoint attribute; in this repo the attribute is only ever set to developer-authored literals (default '768px').
  - Resolution evidence: a concrete lower-trust input-to-breakpoint binding and its authorization boundary.

Do not re-raise `release-workflow-mutable-third-party-action`: the parent records it as rejected during independent verification. Reconsider only with new evidence.
