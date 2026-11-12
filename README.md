# Smarter Search for Magento 2

Product and category autocomplete, typo-tolerant search, promoted products and editable stop words on top of
Magento's own OpenSearch search.

Magento Open Source with OpenSearch already handles full-text search, synonyms and search suggestions. What it
lacks is a useful search dropdown, a second chance for misspelt queries, a way to pin products to a search term,
and control over which words are ignored. This module adds exactly those parts and leaves the rest to core.

## Features

- **Product and category autocomplete**: the search dropdown shows matching products with image and price, and
  matching categories with their path ("Men › Tops › Jackets"), so same-named categories can be told apart.
  Clicking a row, or pressing Enter on it, opens that page.
- **Typo-tolerant fallback**: when a search finds nothing, it runs once more with fuzzy matching, so "jakcet" still
  finds jackets. Exact matching stays the default, so "shirt" doesn't turn into "short".
- **Misspellings stay out of suggestions**: a term rescued by the fallback is saved with 0 results, so the typo is
  never suggested to other shoppers. It shows in *Marketing > Search Terms* as a 0-result term, a candidate for a
  synonym.
- **Popular terms in the dropdown**: correctly spelt searches show your store's own popular terms with their result
  counts ("jacket (23)") before spelling corrections.
- **Promoted products**: list SKUs on a search term and those products appear first, in your order, even if they
  don't contain the search term. Visibility, category, price and layered-navigation filters still apply.
- **Stop words you control**: add your own stop words or keep built-in ones searchable (for example "it" for an IT
  store), per store view, with no reindex. Matching is case-insensitive, so "The" is treated like "the".
- **Built on core**: multi-store and multilingual search, synonyms, "Did you mean" suggestions and partial SKU
  search keep working as in core Magento.

## How it works

Shoppers type in the normal storefront search box. The dropdown shows search terms, then products and categories.
On the results page, promoted products come first, and an empty result triggers one fuzzy retry. If layered
navigation filters are what emptied the page, the module doesn't retry, so filters behave as shoppers expect.

Merchants manage promoted SKUs on the search term's edit page and the settings in store configuration. Saving a
term's promoted SKUs clears the cached result pages, so the change shows immediately.

## Good to know

- Promoted products apply to the storefront search page, not to GraphQL search (for example a PWA storefront).
- On a store with several languages, Magento caches one stop-word list for all locales, so for up to 15 minutes a
  store view may use another locale's built-in list. This is core behaviour the module does not change.
- Spelling corrections in the dropdown come from OpenSearch (core) and can be odd word stems ("lavend"). Turn them
  off with *Enable Search Suggestions = No*; the popular-term rows stay.
- Synonyms are core Magento: after editing them, the search index is rebuilt by cron or by a manual reindex.

## Requirements

- Magento Open Source 2.4.8 with OpenSearch
- PHP 8.3 or 8.4
- Magento modules: Search, CatalogSearch, AdvancedSearch, Elasticsearch, OpenSearch

## Installation

```
composer require tonyapps/module-search
bin/magento module:enable Tonyapps_Search
bin/magento setup:upgrade
```

## Configuration

- **Stores > Configuration > Catalog > Catalog > Catalog Search**: Typo-Tolerant Fallback Search (default Yes),
  Products in Autocomplete (default 5, 0 turns it off), Categories in Autocomplete (default 3, 0 turns it off),
  Additional Stop Words, Words to Keep. All settable per store view.
- **Marketing > SEO & Search > Search Terms > edit term**: Promoted SKUs.
- **Marketing > SEO & Search > Search Synonyms**: core synonyms.

## For developers

Architecture, decisions, data model and tests are in [AGENTS.md](AGENTS.md).
