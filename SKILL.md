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
# -> {"order_id", "share_code", "share_link": ".../gb/menu/<store_id>/<code>"}
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
`nidin group close <order_id> --yes` ends the group for everyone (no undo —
confirm with the user first).

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

`delivery_type`: 1=自取 2=內用 3=外送 4=外帶 21=店家外送.

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
nidin order set <order_id> --name 楊 --phone 09xxxxxxxx
nidin order set <order_id> --delivery-type 1
nidin order set <order_id> --field key=value      # any other order field
nidin order pay <order_id> --yes                  # SUBMITS — returns trade_url
nidin order void <order_id>                       # cancel a pending payment
nidin pay-methods                                 # trade_type codes
```

`order pay` calls `startTrade`, which **submits the order to the shop** and
returns `trade_info.trade_url` — the payment redirect (LINE Pay etc.). It refuses
without `--yes`, naming the store, total and method so a human can confirm.

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
  from `/groupBuy/{share_code}/brief`.
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

## Still unverified

`GET /checkTrade/{...}` (payment polling) has never been called. Every other
endpoint referenced here has been exercised against the live API.

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
