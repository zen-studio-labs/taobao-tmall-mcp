---
name: taobao
description: Taobao (淘宝) and Tmall (天猫) product data through the taobao MCP tools. Use when the user wants Taobao or Tmall products searched by keyword with prices, sales and shop ratings, a product's full details with its SKU variants, prices, stock and attributes, buyer reviews (买家评价) of a product, or everything a Taobao or Tmall shop (店铺, 旗舰店) sells, for price comparison (比价), product sourcing, cross-border buying (代购) or brand monitoring.
---

# Taobao (淘宝) and Tmall (天猫)

Three Taobao tools plus Apify's run and dataset tools, served by the `taobao` MCP server. Tmall listings are part of Taobao: the same tools, links and item IDs cover both. Each Taobao tool starts a paid run on the user's Apify account and returns run metadata. The rows arrive later in a dataset. Every request follows the **run flow** below.

| Tool (client prefixes vary) | Use for |
|---|---|
| `zen-studio--taobao-search-scraper` | Keyword product search: price, discounted price, sales band, shop, shop scores; optional full details and reviews per product |
| `zen-studio--taobao-detail-scraper` | Full details for known product links, share links or item IDs: SKU variants with price and stock, attributes, images, shop; optional buyer reviews |
| `zen-studio--taobao-seller-products-scraper` | Everything a shop sells, from a product link from the shop (preferred), a Taobao shop link or a seller handle such as `inman` |
| `get-actor-run` | Wait for and check a run |
| `get-dataset-items` | Read the rows |
| `fetch-actor-details` | Full input reference and README of any of the three |
| `get-key-value-store-record` | Read a record a run saved in its key-value store |
| `abort-actor-run` | Stop a run the user no longer needs |

If only Apify's generic `call-actor` is available, call it with `actor: "zen-studio/<name>"` (for example `zen-studio/taobao-search-scraper`) and the same input; the run flow is identical.

## Run flow

A tool call returns after at most 45 seconds, whether or not the run has finished. Many runs take longer, so treat the first response as a receipt, not the result.

1. **Start.** Call the Taobao tool with the recipe input and `"waitSecs": 45`. Keep the `runId` from the response. Done when you hold a `runId`.
2. **Poll.** Read `status`. While it is `READY` or `RUNNING`, call `get-actor-run` with `runId` and `waitSecs: 45`, and tell the user the run is still going between calls. Done when `status` is terminal: `SUCCEEDED`, `FAILED`, `ABORTED` or `TIMED-OUT`. A response that is already `SUCCEEDED` skips straight to step 3. Read `statusMessage` on every terminal run, `SUCCEEDED` included: it carries free-plan caps and how many products failed.
3. **Fetch.** Call `get-dataset-items` with `datasetId` from `storages.datasets.default.id` (the response's `nextStep` also names it). Pass `limit` (20 to 50) and a `fields` projection from **Reading results**; Taobao rows carry well over a hundred fields, and `skus`, `descriptionHtml` and `reviews` are large. Dotted fields come back flattened: `fields: "shop.shopName"` returns a key named `"shop.shopName"`, not a nested object. Page with `offset` until you have what the question needs. Done when every row the answer depends on is read.
4. **Answer.** Summarise for the user: prices in yuan, sales, shop, product links (`url`), and anything filtered out. Translate Chinese titles, options and reviews when the user writes in another language.

One run per request. A run that is still `RUNNING` keeps working and billing; starting a second run for the same question pays twice. Resume polling the same `runId` instead, including after a client timeout or a new turn.

Terminal states other than `SUCCEEDED`: `TIMED-OUT` and `ABORTED` can hold partial rows, so fetch them and say the result is partial. `FAILED`: report `statusMessage`. Change the input only when the message names invalid input. Otherwise keep the input, fetch any rows the run saved, and retry later only if the user agrees; never start a paid retry on your own. A `SUCCEEDED` run whose item count reads 0 can lag for a few seconds; fetch once before concluding it is empty. A row with an `error` field (`free_tier_exhausted` or `free_tier_unavailable`) and no `price` is a notice, not data: it carries the free-plan message in `message` (see **Cost and limits**).

## Chaining the tools

- **Search, then details.** Pass a search row's `url` (`https://item.taobao.com/item.htm?id=<itemId>`) or its `itemId` to Product details for SKU variants, stock levels, attributes and, with `fetchReviews`, reviews. Batch the products you need into one details run.
- **Search or details, then the shop.** Pass the product `url` to Shop products (preferred); it finds the shop that sells it. Prefix bare item IDs: `itemId:844075013790`. A bare number is read as a seller ID, which the tool cannot resolve on its own; use a link instead. Use `shop.shopUrl` from details rows only when it is non-empty and its host is `shop<id>.taobao.com` or `<name>.taobao.com`; otherwise pass the product `url`. `{"sellerId": "<shop.sellerId>", "shopId": "<shop.shopId>"}` also works when both are filled.
- **Shop, then details.** Shop products rows carry `url` and `itemId`; pass them to Product details for SKUs and reviews.
- **User links.** Product details accepts `item.taobao.com`, `detail.tmall.com`, short share links (`e.tb.cn/...`) and affiliate links (`s.click.taobao.com`). Tmall shop links (`brand.tmall.com`) do not work in Shop products; ask for a product link from that shop.

## Input recipes

Start small (10 to 30 products) unless the user asks for more; every returned product, details row and review is billed.

**Product search** (`taobao-search-scraper`):

```json
{ "keyword": "机械键盘", "maxItems": 20, "sort": "sales",
  "minPrice": 200, "maxPrice": 600, "tmallOnly": false,
  "enrichWithDetails": false, "fetchReviews": false }
```

- `keyword`: one keyword per run (required). Chinese keywords match how sellers title their listings; brand and model names work as they are. Run one search per keyword.
- `maxItems`: 10 to 1,000, default 100; always set it. Values below 10 become 10.
- `sort`: `relevance` (default), `sales` (best sellers first), `highest_price`. There is no lowest-price sort: use `maxPrice` and sort the rows yourself.
- `minPrice` and `maxPrice` are in yuan. `tmallOnly: true` keeps Tmall listings only.
- `enrichWithDetails` is **on by default** and adds the Product details fields (`isTmall`, `sales`, `skus`, `attributes`, `brandName`, full `shop`) to every row, at a separate charge per product. Set it to `false` for price and sales comparisons; turn it on, or chain into Product details, when the user needs SKUs or exact sales.
- `fetchReviews` with `maxReviewsPerItem` (1 to 1,000, default 100) nests reviews on each row, billed per review. Keep it off unless the user wants review text.

**Product details** (`taobao-detail-scraper`):

```json
{ "items": ["https://item.taobao.com/item.htm?id=844075013790", "1003783113480"],
  "fetchReviews": false }
```

- Product links, Tmall links, share links, affiliate links or numeric item IDs, up to 1,000 per run. Batch them: the start fee is charged once per run. Entries that cannot be read are skipped and not billed; the rest continue.
- `fetchReviews: true` with `maxReviewsPerItem` (default 100; always set it) adds `reviews[]`. Each review is billed.
- `skus[]` holds each returned variant's own price, stock and option names. On some listings it holds fewer variants than the product page shows; when the user needs every option, say the list may be incomplete. `propsList` and `propsImages` map option IDs to names and swatches.

**Shop products** (`taobao-seller-products-scraper`):

```json
{ "sellers": ["https://item.taobao.com/item.htm?id=844075013790"],
  "maxItemsPerSeller": 30, "sort": "best_selling" }
```

- `sellers`: a product link from the shop (most reliable), a Taobao shop link (`shop<id>.taobao.com` or `<name>.taobao.com`), a seller handle such as `inman` (Chinese shop names such as 迈从官方旗舰店 are not accepted), `itemId:<id>`, or `{"sellerId": "...", "shopId": "..."}` with both IDs. A bare seller ID on its own is not enough.
- `maxItemsPerSeller`: 1 to 5,000, default 100; always set it.
- `sort`: `recommended` (default), `best_selling`, `newly_listed`, `lowest_price`, `highest_price`.

## Reading results

Project fields with `get-dataset-items` `fields` (dot paths allowed, returned flattened). Good starting sets:

- Product search: `error,message,searchRank,itemId,url,title,price,discountedPriceFromSearch,salesSignal,shop.shopNameFromSearch,sellerGoodrat,miaoshuDsr,fuwuDsr,fahuoDsr,sellerLoc`. With details on, add `priceFromSearch,isTmall,sales,skuCount,brandName,shop.shopName,couponPrice,priceVerified`.
- Product details: `error,message,itemId,url,isTmall,title,price,originalPrice,couponPrice,priceVerified,sales,stock,skuCount,brandName,categoryCrumbs,shop.shopName,shop.shopUrl,location,detailLevel`. Add `skus` for variants, `attributes` for specifications, `pictures` for images and `reviews` for review text; read those in small pages.
- Shop products: `error,message,catalog.rank,shopName,itemId,url,title,price,discountPrice,soldCount30Day,orderCount30Day,commentCount,freeShipping`. Its `isTmall` can read `false` for a Tmall store; use `isTmall` from Product details (or search with details on) to tell Tmall from Taobao shops. A count Taobao shows as a band arrives as its lower bound (`100+` as `100`, `1万+` as `10000`); say "at least" when you quote `soldCount30Day`, `orderCount30Day` or `commentCount`.

Read `error` and `message` first: a row with `error` and no `price` is a notice (see **Cost and limits**), not a product. Prices are strings in yuan (`"499.00"`) and indicative. In details rows, `price` is the current selling price with active promotions and `originalPrice` the pre-discount price when there is one. In search rows, `price` is the search-result price only when `enrichWithDetails` is off; with details on it is the details price, and `priceFromSearch` keeps the search-result price. `discountedPriceFromSearch`, and shop `price` and `discountPrice`, are the prices shown in those results; they can differ from the details `price` of the same product, so name which one you quote. `couponPrice` (券后价) is often empty; when filled, it is the price after public coupons. `priceVerified: true` marks a confirmed `price`; with `false`, present the price as indicative and possibly lower at checkout. `salesSignal` is a band (`1000+`, `1万+`, where 万 is 10,000), not an exact count. `sellerGoodrat` is the shop's positive-rating rate in percent; `miaoshuDsr`, `fuwuDsr` and `fahuoDsr` are its description, service and shipping scores out of 5. `null` means Taobao did not show the value, not zero. In details rows, `sales` of `0` can mean no count was shown; `totalSold` carries the lifetime count when Taobao shows one. A details row with `detailLevel: "partial"` has empty `stock`, `sales`, `location`, shipping fees and `video`; say so instead of reporting them as zero. Review `user.nick` is masked by Taobao. `fetch-actor-details` returns the full field reference when the user asks for something outside these sets.

These tools save no files. `get-key-value-store-record` (`keyValueStoreId` from the run's `storages.keyValueStores.default.id`, `recordKey` such as `INPUT`) reads a run's key-value records when you need to check the input a run used.

## Cost and limits

- Pricing is per result on each tool's Apify Store page (`https://apify.com/zen-studio/<name>/pricing`) and is billed to the user's Apify account. Quote the page rather than a remembered number. Search detail enrichment and reviews have separate charges.
- On Apify's free plan each tool allows a small preview:
  - Product search: 5 runs in total, one page of 10 products per run, full details for the first product only, no reviews. Do not spend these on tests.
  - Product details: 10 runs in total, the first product only, no reviews.
  - Shop products: 10 runs in total, up to 20 products per run.
- A capped search or details run says so in its `statusMessage` ("Free trial: returning 1 of 5 products. Paid runs process up to 1000 products per run."); Shop products stops at 20 rows without a notice. A free run with `fetchReviews` on returns its products without reviews and ends with "Free trial: customer reviews are included on paid plans." When the allowance is used up, the run returns one row with `"error": "free_tier_exhausted"` and a message such as "Free plan allows 10 runs. You have used all 10. Upgrade your plan for unlimited usage." A free run of search or details can also return `"error": "free_tier_unavailable"` with "The free preview is temporarily unavailable. Please try again shortly, or run on a paid plan for uninterrupted access." Neither row is billed as a result. Tell the user the result is partial or did not run, and that any paid Apify plan removes the caps. For `free_tier_unavailable`, say only that the free preview is temporarily unavailable: try again later, or use any paid plan.
- `maxItems`, `maxItemsPerSeller`, `maxReviewsPerItem`, `enrichWithDetails` and `fetchReviews` are the cost controls. State the caps you chose in the answer.
