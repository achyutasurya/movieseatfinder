# Movie Language Identification

This document describes how Movie Seat Finder identifies the spoken language for a Fandango showtime and applies the optional language filter.

## Source data

Fandango returns language information inside each movie variant's `amenityGroups`. Each group contains an `amenities` array with a `name` and optional `description`, alongside its showtimes. The implementation lives in `src/backend/application.py`.

## Resolution rules

`showtime_language(group)` returns one canonical label. It checks explicit labels across all amenities before checking descriptions, so an unrelated amenity description cannot override an explicit language:

1. `X with Y Subtitles` → `X` (for example, `Hindi with English Subtitles` → `Hindi`).
2. `X Language` → `X` (for example, `Japanese Language` → `Japanese`).
3. `X Dubbed` → `X` (for example, `English Dubbed` → `English`).
4. Descriptions containing `spoken in X`, `presented in X`, or `dubbed in X` → `X` when `X` is a known language. This restriction prevents format descriptions such as `presented in IMAX` or `presented in high definition` from becoming language options. Add new language names to `LANGUAGE_LABELS` when needed.
5. Subtitle/caption-only labels such as `English Subtitles`, `Spanish subtitled`, `Closed caption`, and `Open caption` do not identify spoken language.
6. If no spoken-language evidence exists, the showtime defaults to `English`. This is a compatibility assumption for unlabeled Fandango groups, not confirmation of their spoken language.

Labels are normalized to title case for known languages, including Arabic, Cantonese, English, Hindi, Japanese, Korean, Mandarin, Nepali, Spanish, Tamil, and Telugu. Unknown language names are retained after whitespace cleanup so new Fandango values remain usable.

## Filtering flow

Movies load without a separate language request. `/api/movies` returns the movie list and language options together, using one traversal of the existing Fandango showtime payloads. The **Language · Any** text disclosure is always visible below Movie and uses the same styling as **Which format is best?**, without counts. It is muted and disabled until a valid ZIP code or browser location loads the movie response. Once available, clicking the text reveals the language dropdown; it stays collapsed until opened. The text reflects an active language filter, including when no matching showtimes remain. Missing language metadata leaves Any language visible but disabled; an explicit filter remains editable so it can be cleared.

Choosing a language filters the movie list, formats, and seat-search results. Language options always describe the full location/date/theatre scope, so filtering cannot hide the other choices. The selected movie stays selected if it remains available; otherwise it is cleared with a prompt to choose a movie in the chosen language. Dependent formats refresh and incompatible selections are cleared. Changing movies does not reset the language.

Saved searches, shared links, and criteria refreshes preserve the selected language. If a language no longer has showtimes, the selection remains visible with a `no showtimes` label; the search is never silently broadened to all languages. Omitting the optional `language` parameter preserves the previous API and URL behavior (`any`).

Saved language names match available options without regard to case, so `hindi` selects `Hindi` without adding a duplicate option. `ANY` selects `Any language`. Backend comparisons preserve non-Latin names so distinct languages cannot match merely because they use non-ASCII characters.

The language control never gates movie selection or searching with Any language. If a valid movie response omits language metadata or supplies malformed metadata, movies remain usable; an explicit filter stays visible and continues to be applied. Empty language metadata means there are no available language options, while missing metadata means availability is unknown. Responses from superseded movie and format requests cannot replace current options, including during the criteria-refresh debounce.

## HTTP API

`GET /api/movies` returns `{"movies": [{"title": "Movie title"}], "languages": ["English", "Hindi"]}`. The existing movie structure is unchanged. Language options are sorted and deduplicated across matching theatres and dates, including duplicate entries for the same title, and include only available, non-expired showtimes. The optional language filter affects `movies`, not `languages`.

`/api/movies`, `/api/formats`, and `/api/search` accept an optional, case-insensitive `language` parameter, defaulting to `any`. With an explicit language filter, formats come only from available, non-expired matching showtimes. Seat-search matches include a `language` field. Existing validation, upstream-error handling, and rate limiting still apply. The separate `/api/languages` endpoint has been removed.

## Testing and maintenance

Parser cases are in `tests/test_application.py`; the language text disclosure, movie selection, saved searches, and stale-response handling are in `tests/browser.integration.spec.mjs`. When Fandango introduces a new pattern, add a fixture and test before changing the parser. Prefer explicit spoken-language evidence and never infer a spoken language from subtitle or accessibility metadata alone.
