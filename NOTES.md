# NOTES — Where I expect the agent to get CONTRACT.md wrong

Based only on reading `CONTRACT.md`; I have not read the code or the tests.

## Prediction 1 — The order of checks in `POST /v1/orders` (§4.2)

**Sentences from the contract:**

> 3. **The key has been used before** (see *Idempotency* below): replay, or `422 idempotency-key-reused`, or `409 idempotency-key-in-flight`.
> 4. Every field follows the table above, else `400 validation` listing **every** broken field.

(together with the sentence "The server checks, **in this order**, and stops at the first step that fails")

**Why:** The familiar way to write this handler is to parse the JSON, validate the body, check the menu items, and only then look up the idempotency table. An agent can easily put step 3 after step 4, because "validate the body first, then look up the key" feels natural. The contract says the opposite, so two situations will go wrong:

- The same key is sent again with a different body, and the new body is invalid. The contract requires `422 idempotency-key-reused`, but the agent's code will return `400 validation`.
- The exact stored body is sent again. The agent re-runs validation and the menu item check before replaying. If a menu item was deleted in the meantime, the replay is wrong: it should return the stored `201` response.

The happy-path test will still pass; only a test about the order will expose the bug.

## Prediction 2 — `nextCursor` is `null` only when there is no more data (§4.4)

**Sentence from the contract:**

> `nextCursor` is `null` **if and only if** there are no more orders after this page. A client never receives a cursor that leads to an empty page.

**Why:** The common approach is `LIMIT :limit`, then return a `nextCursor` if `items.length === limit` and `null` otherwise. This is wrong when the total number of orders is an exact multiple of `limit`, for example exactly 20 orders with `limit=20`: the last page is full but nothing follows it, so the client receives a cursor that leads to an empty page, which violates "if and only if". The correct approach is to query `limit + 1` rows, treat an extra row as proof that there is a next page, and drop that extra row before returning. Agents often miss this detail because it only shows up at the boundary case.

A related mistake to watch for: the cursor takes the `id` of the extra row instead of the last row on the page, or the `tableId` filter is not applied again when the cursor is used.
