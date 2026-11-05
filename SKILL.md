---
name: nidin
description: Drive the 你訂 (Nidin) group-ordering platform from the command line - find stores, browse menus, open a group buy (揪團) and share its link, add drinks with sweetness/ice/size choices, watch live totals, track who has paid, and submit an order for LINE Pay. Use whenever the user wants to start, join, monitor, settle, or pay for a food order on order.nidin.shop.
---

# nidin

Zero-dependency Python CLI wrapping the undocumented REST API behind
`order.nidin.shop`. Run `nidin --help` for the full surface.

**Always pass `--json`** when parsing output. It works before or after the
subcommand and wraps everything in a stable envelope:

```json
{"ok": true, "data": ...}
{"ok": false, "error": {"message": "...", "code": 3, "detail": {...}}}
```

Exit codes: `0` ok · `1` error · `2` usage · `3` auth · `4` network.
On exit `3` the user must log in again — do not retry.

## Auth

```bash
nidin login --phone 09xxxxxxxx --password-stdin   # reads password from stdin
nidin whoami
nidin doctor --json                               # connectivity + token check
nidin logout
```

Token lives in `~/.config/nidin/state.json` (mode 0600), or the macOS Keychain
with `--keychain`. It **auto-renews**: the API sets `token_need_replace` and the
CLI transparently calls `/user/{uid}/replaceToken`, so one login lasts
indefinitely. `NIDIN_TOKEN` + `NIDIN_UID` env vars override the state file —
the easiest way to run in a sandboxed runtime.

Never print the token. Never ask the user to paste it into chat.

## Group buy (揪團) — the primary workflow

```bash
# 1. find the store (distance is in METRES, nearest first)
nidin stores --lat 25.0330 --lng 121.5654 --keyword 延吉 --json
nidin stores --count 20                     # no coordinates? default list

# 2. 開團 — creates the empty group cart AND opens it AND prints the link
nidin group create <store_id> --json
# -> {"order_id", "share_code", "share_link": ".../gb/menu/<store_id>/<code>",
#     "delivery_type": 3, "delivery_type_name": "外送"}
#    defaults to 外送 (delivery_type 3); a store with no delivery falls back to
#    自取 and sets "delivery_fallback": true
nidin group create <store_id> --delivery-type 1   # force 自取
#    --dry-run resolves brand_id/menu_id/stamp without creating anything

# 3. share the link, then watch it fill up
nidin group watch <order_id> --interval 30  # prints only on change
nidin group watch <order_id> --once         # single poll
nidin group status <order_id> --json        # totals + per-member paid state
nidin group who <order_id>                  # who ordered what, grouped by person

# 4. settle up
nidin group collect <order_id> <sub_order_id> --paid
nidin group collect <order_id> <sub_order_id> --unpaid
```

Followers live in the **top-level** `group_buy_follower_order_list` of
`/order/{id}/info` — there is no `GET subOrderCollectStatus` (that path is
PUT-only and GETs return 1001). A follower object carries no total of its own:
sum its `product[]` lines. Its `contact_method_remark` is the person's phone.

`group status` returns `group_buying_name`, `share_link`, `total_amount`,
`item_count`, `member_count`, `paid_count`, `unpaid_total`, and `members[]` of
`{sub_order_id, name, amount, paid}` — everything for "現在總共多少錢" and
"誰還沒付".

Also: `nidin group open <order_id>` converts an *existing solo* cart into a
group; `nidin group link <store_id> <share_code>` rebuilds a link;
`nidin group close <order_id> --yes` is 收單: nobody can join or change their
order after it. Followers' orders stay. The web's owner page toggles the same
field back (`--status 10` reopens), but that has not been tried from the CLI, so
confirm with the user before closing.

### Joining someone else's group (跟團)

```bash
nidin group peek <share link>                       # store, opener, open/CLOSED, menu - no login
nidin group join <share link> <product_id> --pick 甜度=三分糖 --dry-run
nidin group join <share link> <product_id> --pick 甜度=三分糖   # into your sub-order, NOT sent
nidin group send <order_id> --yes                   # hand it to the opener
```

`join` adds to your sub-order only; the opener doesn't get it until `send`.
**Confirm the drink and the total with the user before `send`.** The follower
pays the opener directly; there is no online payment. Run `join` again to add
another drink. It finds your existing sub-order through `/groupBuy/{code}/info`.

## Ordering

```bash
nidin menu <store_id> --keyword 紅茶           # find a product_id
nidin product <store_id> <product_id>          # STEP 1: see its combines
nidin product <store_id> <product_id> --combine 北冰   # STEP 2: valid choices
nidin cart add <order_id> <product_id> --combine 北冰 \
      --pick 尺寸=L --pick 溫度=微冰 --pick 甜度=微糖 --amount 2
nidin cart items <order_id>                    # lines + their item_set_uid
nidin cart rm <order_id> <item_set_uid>
nidin cart delete <order_id> --yes             # destroys the WHOLE cart
nidin cart show <order_id>                     # order summary
nidin carts                                    # all your carts
```

**Combine names differ per product, even in the same store.** One drink offers
`冰/熱`, another `北冰/北熱`, merchandise `一般商品`. Never assume — run
`nidin product <store> <id>` with no `--combine` first; it lists them. (If a
product has exactly one combine, `cart add` picks it automatically.)

**Then run `product --combine <x>` before `cart add`.** A product's adjust groups
nominally list every option in the store; only those the chosen combine allows
are legal, and an illegal pick returns 40011. The `product` command applies that
filter, so the options it prints are exactly the valid ones.

`--pick` accepts either the option name or its numeric id, and repeats. Missing a
required group produces a usage error naming what is missing.

When the user does not specify sweetness/ice/size, prefer matching what is
already in their cart (`cart items` shows it) over inventing a default, and say
which you used.

## Delivery (外送) and timing

```bash
nidin addresses                                  # your saved delivery addresses
nidin order delivery <order_id> --index 0        # use saved address N
nidin order delivery <id> --address "..." --lat 25.05 --lng 121.54 \
      --city-code 2 --area-code 11 --note "14F"
nidin order delivery <id> --distance 2400        # override the distance estimate
nidin order set <order_id> --delivery-type 1     # back to 自取

nidin order time <order_id>                      # list slots + show current
nidin order time <order_id> --date 2026-08-26    # slots for another day
nidin order time <order_id> --time 18:30         # set the slot
nidin order time <order_id> --asap               # ASAP instead of a slot
```

`delivery_type`: 1=自取 2=內用 3=外送 4=外帶 21=店家外送. **`group create`
defaults to 3=外送** (falling back to 自取 only when the store has no 外送 tier),
so a new group still needs an address — run `order delivery` before `order pay`,
or switch it back with `order set --delivery-type 1`.

### When 外送 is not possible, say so

`group create`, `order delivery`, `order set --delivery-type 3` and `order time`
all run the same check and report every reason they find, as a `[無法外送]` /
`[注意]` list on stderr and as `delivery_blockers[]` (`{code, level, message}`)
in `--json`. Never bury these — relay them to the user verbatim.

| code | level | meaning |
| --- | --- | --- |
| `not_offered` | hard | no `delivery_type: 3` row in `store.delivery_info` |
| `store_closed` | hard | `is_close` — the store is not taking orders |
| `off_day` | hard | `is_off_day` — closed today |
| `out_of_range` | hard | distance exceeds `store.max_delivery_distance_m` |
| `no_slot_config` | warn | no 外送 `available_time` in `delivery_time_info` |
| `below_minimum` | warn | order total is under the distance tier's minimum |
| `exception_calendar` | warn | `delivery_exception_calendar["3"]` is non-empty, printed literally — the day encoding is unverified, so check it by hand |

A hard reason makes `order delivery` / `order set --delivery-type 3` exit 2
without writing; `group create` falls back to 自取 (`not_offered` only) or, when
you passed `--delivery-type 3` yourself, warns and does what you asked.
`order set --field delivery_type=3` is the escape hatch that skips the check.
`out_of_range` has no override: the distance is a straight-line estimate, so the
real routed distance can only be longer.

**Delivery has a tiered minimum spend** keyed on distance
(`store.delivery_info[]`: `order_min_distance_m` -> `order_min_price`).
`order delivery` reports whether the order clears it and by how much it falls
short. The distance it uses is a **straight-line estimate** — the real routed
distance is longer and can push the order into a higher (more expensive) tier, so
treat a near-miss as a likely miss. Pass `--distance` if you know the real one.

Slots come from `store.delivery_time_info` (lead time + slot width per delivery
type) bounded by `store.opening_times_weekly`, up to `order_tolerance_day` days
ahead. `order time` computes them the same way the site does.

## Payment

```bash
nidin order set <order_id> --method line_pay      # trade_type 707
nidin order set <order_id> --delivery-type 1
nidin order set <order_id> --field key=value      # any other order field
nidin order pay <order_id>                        # preview: what it would send
nidin order pay <order_id> --yes                  # SUBMITS — returns trade_url
nidin order pay <order_id> --yes --name 楊 --phone 09xxxxxxxx   # other 訂購人
nidin order void <order_id>                       # cancel a pending payment
nidin pay-methods                                 # trade_type codes
```

A 揪團 checkout, in order: `group close --yes` (收單) → `order delivery`
→ `order time --time HH:MM` → `order set --method line_pay` → `order pay`
→ `order pay --yes`.

`order pay` checks out the way the web does, in two calls:

1. `POST /order/productCouponFlow` **submits the order to the shop**
   (訂購成功, status 0 → 1). It carries the 訂購人 name and phone. They default
   to the account's own; pass `--name/--phone` for someone else.
2. `POST /order/{id}/startTrade` returns `trade_info.trade_url`, the LINE Pay
   link (status → 2 付款中).

Without `--yes` it sends nothing. It prints the store, total, method,
delivery address, slot and 訂購人, and `--json` adds the exact `would_send`
bodies. Show that to the user before you pass `--yes`.

**The payment link expires about 10 minutes after it is created**
(`pay_before`, from `online_pay_exp_time`). Give it to the user right away.
Paid orders move to status 110 已接單. Re-running `order pay` on an order in
status 1 or 2 only retries `startTrade`; past 2 it refuses. Cash (`--method
cash`) stops after step 1.

For 外送, set a real slot with `order time --time`. The web always sends one and
never uses `is_asap`; the only verified 外送 checkout used a slot.

Never pass `--yes` unless the user asked to pay for that specific order in that
message. Staging an order and handing back the payment link is the norm; firing
it unattended is not.

## Everything else

```bash
nidin store <store_id>             # store detail
nidin menu <store_id> --all        # include options/addons and hidden items
nidin menu <store_id> --version    # menu_id + version stamp
nidin order info <order_id>
nidin order rebuild <order_id>     # "same as last time" -> new cart
nidin api-map --json               # the reverse-engineered endpoint map
```

## Escape hatch

For anything unmodelled — auth headers are added for you:

```bash
nidin raw GET  /order/123/applicableCoupons --json
nidin raw PUT  /order/123/groupBuyingStatus --data '{"group_buying_status":20}'
nidin raw POST /some/path --data - <<< '{"k":"v"}'
```

`--debug` dumps the full request/response to stderr when a shape surprises you.

## Payload gotchas (each one cost a failed request)

- `products` is a JSON **string** of `{"items": [item, ...]}` — an object with an
  `items` key, not a bare array. A bare array returns 40011.
- Item shape: `{product_id, type, amount, combine_id, options: [],
  adjusts: [{adjust_id, options: [{option_id, amount}]}]}`.
- A combine's `edge_list` gates valid choices. Row `[id, qty, null, price_type,
  kind]`: kind `1` = the product, `2` = this single option is allowed, `20` = the
  whole adjust group is allowed.
- `POST /carts/item` with `order_type:2, is_empty:1` **already opens the group**
  and returns `sharing_code`. Calling `toGroupBuy` after it returns 41001.
- `POST /carts/subItem` (with `group_buy_code`) is how a **follower** joins; the
  owner gets 40059. Owners add via `POST /carts/{order_id}/productItem`.
- `GET /order/{id}/groupBuyingStatus` returns 1001 — PUT-only. Read group state
  from `/groupBuy/{share_code}/brief` (no auth). `group_buying_status` 20 = closed.
- `GET /groupBuy/{code}/info` needs a login (1302 without one).
  `group_buy_info.role` 1 = opener; for a follower `group_buy_list[0].order_id`
  is their sub-order.
- A follower sends with `POST /order/productCouponFlow`
  `{order_id, order_type:3, name, total_money, contact_method_remark:0}`.
- `GET /memberinfo` and `GET /carts` do not exist (1001). Use
  `GET /user/{uid}/info` and `POST /carts/list`.
- Login is `type:1, identity:"nidinphone"`. The `type:7/existingmember` pair is
  the brand-member upgrade path and returns 30102.
- Some responses carry a UTF-8 BOM; decode with `utf-8-sig`.
- **`PUT /order/{id}/infoCouponFlow` returns 200 for fields it does not recognise
  and silently drops them.** Always read the order back and confirm a write
  landed. Two traps found this way:
  - the address is `address` / `address_short` / `address_city_code` /
    `address_area_code`, NOT `delivery_address_*`;
  - the slot is `delivery_reserv_start_time: "YYYY-MM-DD HH:MM:SS"` — a single
    string. `delivery_reserv_date`, `delivery_reserv_time`,
    `delivery_reserv_tolerance` and `order_method` are read-only echoes the
    server derives; writing them does nothing.
- `POST /carts/{id}/checkAmountLimit` checks an UPPER cap per time slot
  (`is_over_limit`), not the delivery minimum spend. The minimum is only enforced
  at checkout.
- **`name` / `phone` sent to `infoCouponFlow` are dropped too.** They reach the
  order only in `productCouponFlow`, so `order set --name/--phone` now refuses.
- `startTrade` on an unconfirmed order (status 0 尚未確認) returns **40309**
  「此訂單目前無法使用此操作」 (`cantDoOrder`). Confirm with `productCouponFlow`
  first; `order pay` does that for you.
- `productCouponFlow` body: `{order_id (string), order_type, name, phone,
  phone_type, total_money, invoice_type}`. `phone_type` is 2 for a `09…`
  mobile and 1 otherwise. `invoice_type` defaults to 10, as in the web form, and
  is left out when the store is `is_invoice_exception`. Sending
  `shipping_money` to a store without a shipping fee returns **40022**
  「運費資訊格式不正確」, so leave it out. Stores *with* `has_shipping_fee` need
  `shipping_item_id`/`shipping_money` in a shape that is not mapped yet, so
  `order pay` refuses those; check out in the web UI instead.

## Still unverified

- `GET /checkTrade/{...}` (payment polling) has never been called.
- Whether `startTrade` on a status-2 order whose link expired creates a new link
  without `voidTrade` first.
- `order pay` on stores with a shipping fee, coupons, points or plastic bags
  (the body leaves all of these out).

Every other endpoint referenced here has been exercised against the live API.
`productCouponFlow` → `startTrade` → paid was verified end to end on a 26-cup 外送
揪團 on 2026-11-05.

## Rules

- Only ever operate on the user's own account.
- `cart delete` on a group buy **also destroys every follower's order**, and they
  are not notified. It refuses without `--yes` and names who would lose their
  order. Always show the user that list and get an explicit go-ahead first.
  Despite the verb, it is a soft delete: the order moves to `status: 39` and the
  group to `group_buying_status: 20`. There is no undo — snapshot
  `group who --json` before running it.
- Keep polling intervals human (>= 15s). This is someone else's production service.
- Response shapes can change without notice. If a field is missing the CLI returns
  what it got rather than crashing — re-check with `--raw-body` or `raw`.
