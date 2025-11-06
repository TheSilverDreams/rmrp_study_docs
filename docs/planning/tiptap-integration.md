# TipTap Integration Plan

## Goals
- Replace the current Markdown textarea (`MarkdownEditor`) in the admin docs editor with a richer WYSIWYG experience powered by TipTap.
- Keep parity with current features: version history, attachments, access tags, status transitions.
- Minimise disruption for existing documentation authors and readers.

## Constraints & Context
- Backend APIs currently accept and return document `content` as raw Markdown strings (`DocumentDetail.content`).
- Rendering on the admin/read side goes through `MarkdownContent`, which expects Markdown input and converts it to HTML with `react-markdown`/`remark-gfm`.
- Existing documents are already stored as Markdown; hundreds of entries would need to keep formatting after the change.
- Dark theme, Tailwind-based styling, and role/permission checks must remain untouched.
- We already depend on React 19 + Vite 7; TipTap (v2) is compatible.

## Implementation Outline
1. **Set up TipTap basics**
   - Add `@tiptap/react`, `@tiptap/starter-kit`, and required extensions (bold/italic/heading/list/code, undo/redo, history, tables if needed).
   - Create a new `RichTextEditor` component wrapping `EditorProvider` with our theme tokens.
   - Build toolbar UI (buttons, dropdowns, shortcuts) aligned with existing design. TipTap doesn't ship UI, so we copy/adapt from docs or craft our own.

2. **Content conversion strategy**
2. **Content conversion strategy**
   - Evaluate `tiptap-markdown` to convert Markdown -> ProseMirror JSON.
   - Map supported syntax: headings, lists, blockquotes, code, tables, task lists, inline links, bold/italic.
   - Document unsupported Markdown (e.g., HTML blocks, custom admonitions) and decide whether to block the editor or add extensions.
3. **DocumentEditor integration**
   - Swap `MarkdownEditor` usage for the new `RichTextEditor`.
   - On load: convert incoming Markdown to TipTap JSON; if conversion fails, fall back to plain text node with a warning banner.
   - On submit: serialize TipTap state back to Markdown before sending to API (so backend remains unchanged).
   - Expose helper text for authors about supported formatting and conversion limitations.

4. **Read-side rendering**
   - Option A: keep storing Markdown, keep `MarkdownContent` as-is. No change needed for viewers.
   - Option B (future): store HTML/JSON and render via TipTap/ProseMirror; requires backend change. For first iteration stick to Option A.

5. **Attachments, history, permissions**
   - Ensure version history still captures Markdown diffs (no backend change).
   - Verify attachments uploads remain unaffected (handled elsewhere).
   - Keep change summary textarea and metadata forms untouched.

6. **Testing plan**
   - Unit test conversion helpers with a corpus of real documents.
   - Snapshot tests for toolbar actions (bold/italic/lists) to ensure Markdown round-trips.
   - Manual QA: create/edit docs, verify preview (`DocumentDetailPanel`) displays identical output to old editor.
   - Regression test course modules or other consumers of `MarkdownEditor` before removing it entirely.

## Preserving Existing Documents
- **Complexity:** medium-high. Conversion libraries handle common Markdown, but edge cases (nested HTML, custom brackets, tables) may lose data.
- Mitigation steps:
  - Build an automated script that runs Markdown -> TipTap -> Markdown round-trip on all existing docs. Flag differences >0 length for manual review.
  - For problematic documents, either lock them to legacy editor temporarily or extend TipTap with custom nodes/marks.
  - Provide an escape hatch: if conversion fails runtime, present a fallback textarea with raw Markdown so edits are still possible.
- No backend migration required if we continue persisting Markdown, but we must commit to maintaining converter compatibility.

## Open Questions
- Do we need additional features (color, underline, callouts)? Each requires custom extension and potentially non-Markdown storage.
- Should we migrate other Markdown inputs (course modules, quiz explanations) to TipTap simultaneously for consistency?
- How do we communicate the change to admins and provide a short guide?

## Next Steps
1. Prototype `RichTextEditor` with basic formatting and Markdown round-trip locally.
2. Validate conversion accuracy on a sample of existing documents.
3. Finalize toolbar UX (decide on color picker, table support).
4. Implement phased rollout flag (feature toggle) so we can revert to the old editor if needed.

