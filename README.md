# antislave.com Unofficial API

> Community reference for the undocumented backend behind [antislave.com](https://www.antislave.com).

![status](https://img.shields.io/badge/status-unofficial-orange) ![backend](https://img.shields.io/badge/backend-Supabase%20%2F%20PostgREST-3ECF8E) ![updated](https://img.shields.io/badge/updated-2026--10--14-blue)

> [!WARNING]
> **This is not an official API.** ANTISLAVE publishes no API or terms for programmatic use. Everything here was reconstructed from the site's public frontend bundles and checked against live responses. Endpoints can change or disappear without notice. Use it respectfully: keep request rates low, cache results, and don't use it to scrape private data or abuse the service.

---

## Table of contents

- [Overview](#overview)
- [Quick start](#quick-start)
- [Connection details](#connection-details)
- [Authentication](#authentication)
- [Confirmed endpoints](#confirmed-endpoints)
- [Tables & views](#tables--views)
- [RPC functions](#rpc-functions)
- [Storage](#storage)
- [Realtime](#realtime)
- [Finding RPC parameters](#finding-rpc-parameters)
- [Examples](#examples)
- [Contributing](#contributing)
- [Disclaimer](#disclaimer)

---

## Overview

antislave.com is a React single-page app backed by **Supabase (PostgREST)**. That means:

- **Tables and views** are read through auto-generated REST: `GET /rest/v1/<name>?select=*`
- **Functions** are called through `POST /rest/v1/rpc/<name>` with a JSON body of named params
- **Realtime** (chat, presence, viewer counts) runs over **Pusher** websockets, not Supabase Realtime
- **Media** is served from Supabase Storage
- Access uses the public Supabase **anon key** shipped in the frontend; Postgres **row-level security (RLS)** decides what each request can see

## Quick start

```bash
export ANTISLAVE_ANON_KEY="<anon key from the site's JS bundle>"

curl "https://api.antislave.com/rest/v1/live_now?select=*" \
  -H "apikey: $ANTISLAVE_ANON_KEY" \
  -H "Authorization: Bearer $ANTISLAVE_ANON_KEY"
```

## Connection details

| Item | Value |
| --- | --- |
| REST base | `https://api.antislave.com` |
| Table / view read | `GET /rest/v1/<name>?select=*` |
| Function call | `POST /rest/v1/rpc/<name>` (JSON body) |
| Storage | `https://api.antislave.com/storage/v1/object/public/<bucket>/<path>` |
| Realtime | Pusher websocket (`wss://ws-<cluster>.pusher.com`) |
| Edge functions | `translate` |

## Authentication

Every request needs both headers:

```http
apikey: <ANON_KEY>
Authorization: Bearer <ANON_KEY>
```

- The anon key is a standard public Supabase JWT (`role: anon`, expires 2031). It starts with `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiYW5vbi...`. Pull the current value from the site's `/assets/*.js` bundle rather than hard-coding it.
- It's public by design. Data exposure is enforced server-side by RLS.
- **Read-only public data** works with the anon key alone.
- **Write / account actions** (DMs, posts, clips, lounges, settings) need a **logged-in user JWT** in `Authorization: Bearer <USER_JWT>` (keep `apikey` as the anon key). Without it RLS rejects the call.

## Confirmed endpoints

Verified against the live backend (HTTP 200):

| Endpoint | Returns |
| --- | --- |
| `GET /rest/v1/live_now?select=*` | Currently-live channels: username, title, category, avatar/frame URLs, source (`antislave` / `kick`), `started_at` |
| `GET /rest/v1/recently_live?select=*` | Recently-live list with Kick slugs and `last_live_at` |

> [!NOTE]
> `POST /rest/v1/rpc/films_home` with an empty body returns `PGRST202` (function not found for those arguments). It expects named parameters. Most RPCs behave this way; see [Finding RPC parameters](#finding-rpc-parameters).

## Tables & views

Read with `GET /rest/v1/<name>?select=*`. About 50 are referenced by the frontend. Not all are readable with the anon key; RLS may return empty results or an error.

| Area | Names |
| --- | --- |
| Live / streams | `live_now`, `recently_live`, `live_channel`, `live_categories`, `movie_nights_now`, `channel_vods`, `channel_rerun`, `rerun_media`, `watch_parties` |
| Discovery | `games`, `game_genres`, `game_favourites`, `category_art`, `category_follows`, `category_follower_counts`, `my_category_follows` |
| Social | `posts`, `profiles`, `verified_users`, `supporter_users`, `follows`, `follower_counts`, `my_following`, `name_colors`, `gradient_name_users` |
| Films | `indie_films`, `film_favourites`, `tv_episodes` |
| Lounges | `lounge_directory`, `lounge_media`, `lounge_reactions`, `lounge_voice_lines` |
| Account / billing | `premium_plans_public`, `premium_subscriptions`, `addon_prices`, `email_consent`, `push_subscriptions`, `platform_flags`, `tts_hotbars` |
| Chat / mod | `active_chatters`, `channel_pin`, `channel_hidden`, `channel_chat_clears` |
| Misc | `slot_jackpots`, `feedback_items` |

PostgREST query syntax works as usual:

```
/rest/v1/live_now?select=username,title,category&order=started_at.desc&limit=10
/rest/v1/profiles?select=*&username=eq.someuser
```

## RPC functions

About 180 functions, called with `POST /rest/v1/rpc/<name>`. Functions named `my_*` or `set_*`, and most write actions, require a user JWT.

<details>
<summary><b>Streams / playback</b></summary>

`stream_playback`, `stream_edges`, `mint_stream_token`, `drop_stream_token`, `my_stream_key`, `my_stream_setup`, `set_stream_setup`, `rotate_stream_key`, `my_live`, `my_live_listing`, `answer_live_listing`, `set_my_live_visibility`
</details>

<details>
<summary><b>VODs / reruns</b></summary>

`channel_vods`, `latest_vod`, `my_vod_settings`, `set_my_vod_recording`, `delete_my_vod`, `request_vod_download`, `vod_download_state`, `rerun_state`, `set_my_rerun`, `set_rerun_items`, `my_rerun_access`
</details>

<details>
<summary><b>Clips / shorts</b></summary>

`browse_clips`, `channel_clips`, `clip_by_slug`, `create_clip`, `refine_clip`, `delete_clip`, `my_clips`, `my_clip_storage`, `shorts_feed`, `kickClip`
</details>

<details>
<summary><b>Chat / moderation</b></summary>

`chat_history`, `user_messages`, `hide_message`, `pin_message`, `unpin_message`, `clear_channel_chat`, `channel_chat_rules_of`, `set_my_chat_rules`, `channel_roles_of`, `set_channel_role`, `my_mod_channels`, `my_ban_status`, `submit_unban_request`
</details>

<details>
<summary><b>Direct messages</b></summary>

`dm_inbox`, `dm_open`, `dm_page`, `dm_send`, `dm_mark_read`, `dm_set_block`, `my_dm_prefs`, `set_dm_prefs`
</details>

<details>
<summary><b>Films</b></summary>

`films_home`, `films_page`, `film_detail`, `films_cards`, `films_like`, `set_film_favourite`, `indie_film_submit_upload`, `indie_film_edit`, `indie_film_review`, `indie_film_withdraw`, `my_indie_room`
</details>

<details>
<summary><b>Posts / feed</b></summary>

`post_feed`, `post_thread`, `create_post`, `delete_post`, `set_post_like`, `post_view_counts`, `record_post_events`, `search_site`, `post_site_message`
</details>

<details>
<summary><b>Lounges</b></summary>

`create_lounge`, `join_lounge`, `leave_lounge`, `lounge_by_ref`, `lounge_roster`, `lounge_messages_page`, `send_lounge_message`, `edit_lounge_message`, `delete_lounge_message`, `lounge_reactions_for`, `set_lounge_reaction`, `lounge_stickers_list`, `send_lounge_sticker`, `kick_lounge_member`, `unban_lounge_member`, `lounge_ban_list`
</details>

<details>
<summary><b>Lounge voice / TTS</b></summary>

`my_lounge_voice`, `set_lounge_voice`, `set_lounge_voice_listen`, `mark_lounge_voice_seen`, `channel_voice_setup`, `my_voice_lines`, `set_my_voice_hotbar`, `voice_cooldown_ms`, `tts_listen_beat`, `tts_listen_leave`
</details>

<details>
<summary><b>Account / profile</b></summary>

`user_card`, `has_password`, `set_avatar`, `set_banner`, `my_profile_art`, `set_my_gradient_name`, `set_socials`, `username_available`, `username_rarity`, `set_primary_platform`, `start_platform_link`, `unlink_kick`, `my_connections`
</details>

<details>
<summary><b>Premium / badges</b></summary>

`claim_premium`, `claim_daily_bonus`, `set_my_sub_cancel`, `set_my_supporter_badge`, `channel_sub_badges`, `channel_sub_events`, `set_marketing_consent`
</details>

<details>
<summary><b>Follows / notifications</b></summary>

`set_follow`, `set_follow_notify`, `my_muted_follows`, `notification_feed`, `unread_notifications`, `mark_notifications_read`, `save_push_subscription`, `drop_push_subscription`, `request_test_push`
</details>

<details>
<summary><b>Schedule / events</b></summary>

`channel_schedule`, `create_schedule`, `update_schedule`, `delete_schedule`, `skip_occurrence`, `start_watch_party`, `watch_party_state`, `watch_party_beat`, `watch_party_control`, `holdem_event_now`, `join_holdem_event`, `leave_holdem_event`, `slot_winners`
</details>

<details>
<summary><b>Feedback / changelog</b></summary>

`submit_feedback`, `submit_feedback_note`, `feedback_list`, `search_feedback`, `vote_feedback`, `site_changelog`, `site_changelog_latest`
</details>

## Storage

Public objects are served at:

```
https://api.antislave.com/storage/v1/object/public/<bucket>/<path>
```

| Bucket | Contents |
| --- | --- |
| `profile-media` | Avatars, banners |
| `stream-frames` | Live stream thumbnails / frames |
| `post-media` | Images attached to posts |
| `lounge-media` | Lounge uploads |
| `indie-films` | Indie film media |

## Realtime

Live updates (chat, presence, viewer counts) use **Pusher** websockets (`wss://ws-<cluster>.pusher.com`). The Pusher app key and cluster are in the frontend bundle. Related heartbeat RPCs:

- `presence_beat`
- `watch_party_beat`
- `tts_listen_beat`

## Finding RPC parameters

Most RPCs need named JSON params, and calling with the wrong ones returns `PGRST202`. Two ways to find them:

1. **Network tab:** open the relevant page on antislave.com, open DevTools → Network, filter by `rpc/`, and inspect the request payload.
2. **Bundles:** search the `/assets/*.js` files for `.rpc("<name>"` to see the object passed as arguments.

If you work out a function's params, please open a PR adding it to this README.

## Examples

### JavaScript (supabase-js)

```js
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  "https://api.antislave.com",
  process.env.ANTISLAVE_ANON_KEY
);

// Who's live right now
const { data, error } = await supabase
  .from("live_now")
  .select("*")
  .order("started_at", { ascending: false });

if (error) throw error;
console.log(data);
```

### JavaScript (fetch)

```js
const BASE = "https://api.antislave.com/rest/v1";
const KEY = process.env.ANTISLAVE_ANON_KEY;

const res = await fetch(`${BASE}/recently_live?select=*`, {
  headers: { apikey: KEY, Authorization: `Bearer ${KEY}` },
});
console.log(await res.json());
```

### Python

```python
import os
import requests

BASE = "https://api.antislave.com/rest/v1"
KEY = os.environ["ANTISLAVE_ANON_KEY"]
HEADERS = {"apikey": KEY, "Authorization": f"Bearer {KEY}"}

live = requests.get(f"{BASE}/live_now", params={"select": "*"}, headers=HEADERS)
live.raise_for_status()

for ch in live.json():
    print(ch.get("username"), "-", ch.get("title"))
```

### Calling an RPC

```bash
curl -X POST "https://api.antislave.com/rest/v1/rpc/<function_name>" \
  -H "apikey: $ANTISLAVE_ANON_KEY" \
  -H "Authorization: Bearer $ANTISLAVE_ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "param_name": "value" }'
```

## Contributing

PRs are welcome, especially:

- Confirmed RPC parameter shapes and example responses
- Response schemas for tables and views
- Notes on which endpoints work with the anon key vs. a user JWT
- Corrections when something changes on the live site

Please don't commit user JWTs, private data, or anything obtained by bypassing access controls.

## Disclaimer

This project is not affiliated with, endorsed by, or supported by ANTISLAVE. It documents publicly reachable behavior of the site's frontend for educational and interoperability purposes. The backend is private and undocumented; it may change at any time. You're responsible for how you use it, including respecting the site's terms of service and its users' privacy.

---

*Source: ANTISLAVE frontend bundles (`https://www.antislave.com/assets/*.js`) and live responses from `https://api.antislave.com/rest/v1/`. Originally captured 2026-10-13 by [@riotcoke](#).*
