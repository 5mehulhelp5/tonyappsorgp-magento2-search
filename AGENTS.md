# Tonyapps_Search — notes for AI agents and developers

Read this before changing the module. Workspace-wide rules are in ../AGENTS.md.

Smarter storefront search for Magento 2.4.8 on OpenSearch. Package `tonyapps/module-search` 1.0.0.

## Status

- **2026-10-25: done.** Browser check passed (see Change history).
- 2026-09-29: first live verification in DDEV.

## Architecture

Core Magento plus OpenSearch already handles most of the feature list. This module adds only the parts core is
missing:

| Feature | How |
|---|---|
| Full-text, multi-store, multilingual | Core. One index per store view. Stemmers and stop words are set per locale in `Magento_Elasticsearch/etc/esconfig.xml`. |
| Synonyms | Core: **Marketing > SEO & Search > Search Synonyms**. Built into the index, so saving a group invalidates `catalogsearch_fulltext`. Cron's `indexer_reindex_all_invalid` rebuilds it, or run `bin/magento indexer:reindex catalogsearch_fulltext`. |
| **Stop words** | `Model\Preprocessor\Stopwords`, a preference on `Magento\Elasticsearch\SearchAdapter\Query\Preprocessor\Stopwords` (same CSV location as core: `Magento_Elasticsearch` / `stopwords`). Takes core's CSV list for the locale, adds **Additional Stop Words** and removes **Words to Keep**, both per store view. Also fixes core's case-sensitive matching, which let "The" through. Applied at query time, so no reindex. |
| "Did you mean" / related terms | Core: **Catalog > Catalog Search > Enable Search Suggestions / Recommendations** (on by default). |
| Partial SKU / name | Core `quick_search_container` already runs `match_phrase_prefix` on name and sku. |
| Long-tail queries | Core. Minimum Should Match is empty by default, so a query matches when any one of its words matches. |
| **Product + category autocomplete** | `Model\Autocomplete\{Product,Category}DataProvider` added to core `Magento\Search\Model\Autocomplete` `dataProviders` (keys 20, 30; the product provider uses `Magento\CatalogSearch\Model\ResourceModel\Fulltext\SearchCollectionFactory`). The `view/frontend/web/js/form-mini-mixin.js` mixin (on `Magento_Search/js/form-mini`, styles in `css/source/_module.less`) shows image and price per row; clicking a row (or Enter on it) opens that page. Category rows show their path below the store root ("Men › Tops › Jackets"). |
| **Typo-tolerant fallback** | `Plugin\Fallback`, one shared instance plugged into `Magento\Framework\Search\AdapterInterface`, `Magento\Elasticsearch\SearchAdapter\Query\Builder\MatchQuery` and `Magento\Search\Model\Query` (the adapter plugin flips the flags the other two read). When a quick search finds nothing, it runs the same search once more with `fuzziness: AUTO`. Exact matching stays the default, so "shirt" won't turn into "short". A term rescued this way is saved with **0 results**, so the typo is never suggested and shows in **Marketing > Search Terms** as a 0-result term (synonym candidate). With layered-navigation filters on, an empty page first checks the term alone (one 1-hit query): if the term matches without the filters, the filters emptied the page and there's no fuzzy retry; a misspelt term still retries with the filters kept. |
| **Search-term rows for correct words** | `Plugin\AddHistoryTerms` on `Magento\AdvancedSearch\Model\DataProvider\AutocompleteSuggestions`. With "Enable Search Suggestions" on, core only shows the OpenSearch suggester's corrections, so a correctly spelled "jacket" got no term row. Core's own search-history terms ("jacket (23)") now come first, then the corrections, deduplicated and capped at the Autocomplete Limit. |
| **Promoted products** | `Plugin\Promote` on `Magento\OpenSearch\SearchAdapter\Mapper` plus a new `search_query.promoted_skus` column. Admin field added by `Plugin\AddPromotedField` on `Magento\Search\Block\Adminhtml\Term\Edit\Form` (**Marketing > Search Terms > edit term**). Listed products come first, in list order, even if they don't contain the search term. Visibility, category, price and layered-navigation filters still apply. |
| Page cache on promotion change | `Observer\CleanPageCacheOnPromotedSkus` on `search_query_save_after` (adminhtml): cleans `full_page` when Promoted SKUs changed. |

## Data model

- `search_query.promoted_skus` (text, nullable): comma-separated SKUs pinned to the top for this term.

## Configuration and ACL

**Stores > Configuration > Catalog > Catalog > Catalog Search** (`catalog/search/*`, default/website/store):

| Path | Label | Default |
|---|---|---|
| `catalog/search/tonyapps_fallback` | Typo-Tolerant Fallback Search | 1 (Yes) |
| `catalog/search/tonyapps_autocomplete_products` | Products in Autocomplete | 5 (0 turns it off) |
| `catalog/search/tonyapps_autocomplete_categories` | Categories in Autocomplete | 3 (0 turns it off) |
| `catalog/search/tonyapps_stopwords_add` | Additional Stop Words | empty; comma or newline separated |
| `catalog/search/tonyapps_stopwords_remove` | Words to Keep | empty; comma or newline separated |

No own ACL or menu.

## Decisions and constraints

- Add only what core is missing; everything else stays core (see table).
- Exact matching is the default; fuzzy is only a one-shot fallback on 0 hits.
- Fallback-rescued terms are saved with 0 results on purpose, so typos are never suggested.

## Known limitations and open points

- Core caches the stop word list under one cache key for all locales. On a store with several languages, the first
  locale cached is used for up to 15 minutes. Not fixed here.
- Promotions are read from the `q` request parameter through `QueryFactory`, so they don't apply to GraphQL search.
- Correction rows from OpenSearch's suggester are core and can be odd stems ("lavend", "lap"). Turn them off with
  **Enable Search Suggestions = No**; the history terms stay.

## Tests

```
vendor/bin/phpunit -c dev/tests/unit/phpunit.xml.dist app/code/Tonyapps/Search/Test/Unit
```

Test files: `Observer/CleanPageCacheOnPromotedSkusTest`, `Plugin/{AddHistoryTerms,AddPromotedField,Fallback,PromotePlugin,QueryRewrite}Test`.
Browser checks so far: headless Chromium against DDEV, desktop and 390px wide.

## Change history

- **2026-11-16** — Released as free software under GPL-3.0-or-later (owner decision): `LICENSE` added,
  `composer.json` license changed from `proprietary`, License section in the README.
- **2026-11-13 package metadata** — `composer.json` requires advanced-search, open-search, catalog, config and store,
  which the code uses.
- **2026-10-30 review fixes** — On 2.4.8, core caches the result pages of popular search terms in the full page
  cache (`Result\Index::getCacheableResult`). Saving a search term whose Promoted SKUs changed now cleans the
  `full_page` cache, so the change shows straight away.
- **2026-10-25 status: done.** Browser check (headless Chromium, desktop and 390px wide):
  - ArrowDown to a product row fills the input and selects the row. Enter opens that product.
  - Enter on a core term row, or with nothing selected, runs a normal search.
  - Escape closes the dropdown. Clicking a category row opens the category. No JS errors.
  - At 390px the dropdown fits the viewport with no sideways scroll.
  - Fixed during the check: "jacket" showed two identical "Jackets" category rows (Men and Women). Rows now show
    the full path.
  - Added after the check: history term rows for correctly spelled words ("jacket" → "jacket (23)", "24-" →
    "24-MB0 (8)"), and typos rescued by the fallback are saved with 0 results ("jakcet" still finds 13 products on
    the page but is no longer suggested). 15 unit tests pass, and the browser run passes again, including Enter on
    a term row.
- **2026-09-29 status** — verified live in DDEV on Luma with sample data:
  - Autocomplete returns category and product rows with image and price.
  - The typo fallback works: "jakcet" returns 23 results, the same as "jacket".
  - Partial SKU works: "24-MB0" finds the bags.
  - Promoted `24-MB01,MJ04-XS-Black` on "jacket": the duffle bag is first (the total goes from 23 to 24). The
    not-visible child SKU stays hidden.
  - Stop words: adding "gym" takes "lando gym" from 27 results to 1. Keeping "the" takes "The lando" from 1 result
    to 172.
  - Core synonyms: "parka,jacket" takes "parka" from 1 result to 17 after a reindex.
  - The admin Promoted SKUs field renders with the saved value.
  - The frontend mixin and its CSS are served.
  - No exceptions were logged.
  - Unit tests: 9 pass (fallback retry: once, only on 0 hits, flag reset even on error; query rewrites; stop words).
