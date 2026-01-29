# Changes to Infrastructure Documentation

This document tracks changes made to the shared infrastructure documentation to remove project-specific content and make it more generic and reusable across projects.

## Date: 2025-01-XX

### OOP.md - Generalization

**Changed:** Section 4 "Current State vs Target" → "Common Patterns & Anti-Patterns"

**Rationale:** The original Section 4 contained project-specific file names and examples (e.g., `reconciliationController.ts`, `ReviewService`, `ReconciliationModel`, `ArtistCleanupController`, etc.) that were specific to the stubhub-recon-gui project. This made the documentation less reusable for other projects.

**Changes Made:**

1. **Section 4.1 Controllers:**
   - Removed project-specific file references (`reconciliationController.ts`, `venueRunController.ts`, `diceDupsController.ts`, `artistCleanupController.ts`)
   - Replaced with generic anti-pattern/pattern examples
   - Added code examples showing bad vs good controller patterns

2. **Section 4.2 Services:**
   - Removed project-specific service references (`reviewService.ts`, `routingService.ts`, `learningService.ts`, `crosswalkService.ts`, `queryBuilder.ts`, `eventIdentityResolver.ts`)
   - Replaced with generic anti-pattern/pattern examples
   - Added code examples showing bad vs good service patterns with dependency injection

3. **Section 4.3 Models:**
   - Removed project-specific model reference (`ReconciliationModel.ts`)
   - Replaced with generic pattern guidance

4. **Section 2 Examples:**
   - Changed examples from project-specific (`ReviewService`, `LearningService`, `rankCandidates()`, `ReconciliationRecord`, `ArtistCleanupController`) to generic (`UserService`, `OrderService`, `calculateScore()`, `User`, `UserController`)

5. **Section 5 Summary:**
   - Removed project-specific references (`ReviewService`, `ReconciliationModel`)
   - Made language more generic

**Result:** The documentation now provides generic, reusable guidance that can be applied to any Node/Express project, while project-specific analysis has been moved to the stubhub-recon-gui project's local documentation.

---

### uiux/FRONTEND_FEATURES.md - Removed

**Rationale:** This file was entirely project-specific, documenting the Event Reconciliation Dashboard frontend features. It contained detailed information about StubHub/Dice.fm reconciliation workflows, which is not applicable to other projects.

**Action:** Moved to `stubhub-recon-gui/docs/uiux/FRONTEND_FEATURES.md` (project-specific location).

### uiux/TANSTACK_GRID_GUIDELINES.md - Generalization

**Changed:** Removed project-specific "Artist Cleanup" case study references

**Rationale:** The document contained specific references to the `ArtistCleanup` page and project-specific filter names (`review_type`, `match_type`, `disable_type`). These were replaced with generic examples.

**Changes Made:**

1. **Case Study Section:**
   - Removed specific "Artist Cleanup case study" section
   - Replaced with generic "Common Performance Issues" section
   - Changed from specific example to general pattern description

2. **Code Examples:**
   - Changed `queryKey: ['artistCleanup', ...]` to `queryKey: ['myGrid', ...]`
   - Changed filter examples from `match_type`, `review_type`, `disable_type` to generic `status`, `category`, `search`
   - Changed console log examples from `[ArtistCleanup]` to `[MyGrid]`

**Result:** The documentation now provides generic, reusable guidance applicable to any TanStack grid implementation.

**Note:** The project-specific case study has been preserved in `stubhub-recon-gui/docs/uiux/ARTIST_CLEANUP_CASE_STUDY.md` for reference.

---

## Notes

- Project-specific content extracted from shared docs has been preserved in the stubhub-recon-gui project's local documentation:
  - `docs/plans/oop-refactoring/CURRENT_STATE.md` - OOP refactoring analysis
  - `docs/uiux/FRONTEND_FEATURES.md` - Frontend features documentation
  - `docs/uiux/ARTIST_CLEANUP_CASE_STUDY.md` - Performance case study
- This ensures no information is lost while making the shared documentation more maintainable and reusable
