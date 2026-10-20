# Sinq Course Preview Adapter — Implementation Plan

Status: design only, isolated branch, not deployed.

## Provenance
Upstream: https://github.com/3519130448wzr-alt/preview-course-in-figjam
License: MIT; retain upstream copyright and license in any adapted source distribution.
Reviewed upstream SKILL.md, README, LICENSE, directory structure, dependencies.
Do not copy FigJam automation wholesale into browser.

## Rationale
Sinq already implements course-scoped upload, PDF/PPTX text extraction, week inference, notes, vocabulary, and Supabase-backed AI requests.
Adapt upstream's distinct features: source-aligned notes, canonical bilingual glossary, concept links to source pages, checkpoint validation.

## Proposed user flow
1. Open course > Week > source document.
2. Choose '预习本周' (optional, module-scoped).
3. Extract a page manifest (index, page kind, source reference, text, content hash).
4. Generate bilingual summaries based on page content; mark uncertain claims rather than hallucinate.
5. Present an in-app split view (original slides left, notes right).
6. Generate weekly glossary and linked concept graph; clicking a term or graph node opens the exact source page.
7. Allow corrections and regenerating one page without rewriting the entire course.

## Data boundaries
- Scope all reads and writes by auth.user_id AND course_id, week, material_id.
- Generate stable page IDs from material ID plus slide number; revision hash detects changed source.
- Persistent job status is server side: queued/processing/needs_review/complete/failed. Idempotent page writes.
- Store originals only with explicit user permission and private authenticated access; never use public URLs by default.
- Respect upload size limits, retention/delete requests, and third-party courseware copyright.
- Keep lecture-derived claims and external material separately attributable.
- Never automatically send entire courseware to third-party FigJam.

## Minimum output contract
{
  "course_id":"uuid",
  "material_id":"uuid",
  "week":1,
  "pages":[{
    "page":1,
    "kind":"content|low_information|image_only|blank",
    "title":"",
    "summary_zh":"",
    "terms":[{"en":"","zh":"","source_page":1}],
    "concept_edges":[{"source":"","target":"","relation_zh":""}],
    "needs_review":false
  }]
}
JSON illustration only; validate all fields server side before storage.

## Reuse vs. redesign
- Adapt deck_manifest.py classification concept -> Sinq browser extraction manifest, avoiding an unnecessary Python service for V1.
- Adapt glossary_dedupe.py normalization rules -> shared JS/server logic.
- Adapt validate_note_payload.py source/term consistency checks -> deterministic server validation plus editor warnings.
- Adapt registry/checkpoint semantics -> existing Sinq Supabase course/material/job IDs, NOT a machine-local registry.json.
- Replace Figma node links -> Sinq deep links (#/course/{courseId}/week/{week}/material/{materialId}/page/{n}) after routing implementation.
- Replace FigJam concept maps -> Sinq's own interactive graph; FigJam export later as optional integration.
- Source image-only content: unlike upstream blanket skip, offer explicit opt-in visual analysis only when needed.

## Delivery phases
P0: Prototype with a single synthetic PDF/PPTX, 10 pages max; in-memory only; verify order, classification, bilingual consistency.
P1: Add course-scoped page summaries and precise return-to-slide links in Sinq UI.
P2: Deduplicated weekly glossary linked to existing word study flow.
P3: Interactive concept map; mobile UI; accessibility.
P4: Per-user quotas, audit log, retry, revision handling, privacy and load testing.
P5: Optional FigJam export after product usefulness is established.

## Acceptance tests
- 100% page numbers preserved even on blank/image-only pages.
- No cross-user or cross-course reads/writes.
- Clicking every term/reference returns to correct page.
- Identical input re-run does not duplicate entries.
- Changed file version never silently overwrites reviewed notes.
- Terminology translations are consistent across notes, glossary and graph.
- AI service failure leaves prior notes intact and retryable.
- No copyrighted uploads published to public storage or FigJam without user authorization.
- Original Sinq upload and flashcard functions remain unchanged.

## Deferred decisions
- Storage schema and AI endpoint implementation require review of the actual deployed Supabase functions and policies; these are excluded from the public frontend repository.
- Production release requires tests and explicit deployment approval.
