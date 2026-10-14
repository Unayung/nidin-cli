# nidin

Command-line control of the 你訂 (Nidin) group-ordering platform — open a 揪團,
get the share link, watch the tally, and see who has not paid, without touching
the web UI.

Zero dependencies. Python 3.8+. Single file.

> **Unofficial.** Not affiliated with, endorsed by, or supported by 你訂 / Nidin.
> This is an independent client built by reading the public web bundle at
> `cdn-order.nidin.shop`. The API it talks to is undocumented and private, so it
> can break without warning.
>
> Use it on **your own account only**, at human request rates, and check 你訂's
> terms of service before you do — automating a private API is commonly against
> them. `order pay` submits a real order to a real shop and spends real money.
> You are responsible for how you use this.

## Install

```bash
chmod +x nidin
ln -s "$PWD/nidin" /usr/local/bin/nidin   # optional
```

## Quick start

```bash
nidin login --phone 09xxxxxxxx      # prompts for password
nidin doctor                        # sanity check

nidin stores --lat 25.0330 --lng 121.5654 --keyword 德正
nidin group create <store_id>       # 開團 (外送 by default) -> share link to paste in chat
nidin group create <store_id> --delivery-type 1   # 自取 instead
#   store cannot deliver? it says why, and falls back to 自取
nidin menu <store_id> --keyword 綠茶
nidin product <store_id> <product_id> --combine 冰
nidin cart add <order_id> <product_id> --combine 冰 --pick 甜度=半糖 --pick 溫度=少冰 --pick 尺寸=L
nidin cart items <order_id>         # lines + item_set_uid, for cart rm
nidin group watch <order_id>        # live tally
nidin group collect <order_id> <sub_order_id> --paid

# 跟團 (someone else's link)
nidin group peek <share link>                       # store + open/closed + menu, no login
nidin group join <share link> <product_id> --pick 甜度=三分糖
nidin group send <order_id> --yes                   # hand it to the opener
```

## Where the token lives

`~/.config/nidin/state.json`, mode 0600. Use `--keychain` at login to put it in
the macOS Keychain instead. `NIDIN_TOKEN` / `NIDIN_UID` env vars override both.

The token renews itself — the API flags `token_need_replace` on responses and the
CLI calls `/user/{uid}/replaceToken` transparently. One login is enough.

The password is never stored.

## Agent runtimes

`SKILL.md` is written for agents. Everything is scriptable:

```bash
nidin group status 12345 --json | jq '.data.unpaid_total'
```

`--json` works before or after the subcommand. Exit codes: 0 ok, 1 error,
2 usage, 3 auth, 4 network.

## How this was built

The API is undocumented. Endpoints, the `MC-API-Token` / `MC-API-User` auth
headers, the `toGroupBuy` call, the share-link format, and the `trade_type` codes
were all read out of the public web bundle at `cdn-order.nidin.shop`.
`nidin api-map` prints the map.

Because nothing is contractual, commands degrade gracefully: a missing field
returns the raw payload rather than raising. `nidin raw` reaches anything the CLI
does not model.

## Paying

```bash
nidin order set <order_id> --method line_pay
nidin order pay <order_id> --yes     # submits the order, prints the LINE Pay URL
```

`order pay` is the commit point: it submits to the shop and returns
`trade_info.trade_url`. It refuses without `--yes` and tells you the store, total
and method first. `nidin order void <order_id>` cancels a pending payment.

## Caveats

- Only use this on your own account.
- Check 你訂's terms; automating an undocumented API is usually against them.
- Keep polling human (>= 15s).
- Upstream can change shape without warning.
