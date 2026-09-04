# rustcord-fork

Fork of **Rustcord 3.4.3** (Kirollos & OuTSMoKE) — a Discord ↔ Rust server integration plugin for Oxide/uMod that relays chat, player events, deaths, and server logs into Discord channels.

This fork exists to carry a single fix: **native clan chat was leaking into the public Discord chat channel**, with no configuration option able to stop it.

---

## Problem

On a server running Rust's native clan system, messages sent in the in-game **clan chat** were being relayed into the public Discord chat channel alongside normal global chat. Clan chat is private by design — team strategy, raid planning, base locations — so republishing it to a channel the whole Discord can read is a meaningful information leak.

The behaviour could not be turned off through config. Toggling the plugin's existing clan-related setting (`plugin_clanchat`) made no difference, because that setting governs a completely different code path (see below). From the plugin's perspective the leaked messages simply *were* global chat, so any channel configured to receive normal chat received them too.

## Root cause

`OnPlayerChat` decided where to route a message using a two-way ternary that only special-cased one channel:

```csharp
// before
if (_settings.Channels[i].perms.Contains(
        channel == ConVar.Chat.ChatChannel.Team ? "msg_teamchat" : "msg_chat"))
```

`ConVar.Chat.ChatChannel` is not a two-value enum — it includes `Global`, `Team`, `Server`, `Cards`, `Local`, and `Clan`. Only `Team` was handled explicitly, so **every other channel fell into the `else` branch and was treated as `msg_chat`**. Once Facepunch shipped the native clan system, in-game clan chat began arriving at this hook as `ChatChannel.Clan`, hit the `else`, and was relayed to every Discord channel holding the `msg_chat` permission.

What made this confusing to diagnose: the plugin *does* already have clan chat handling, in the `OnClanChat` hook gated behind the `plugin_clanchat` permission. But that hook belongs to the third-party **Clans plugin** and is an entirely separate path from Rust's native clan channel. Native clan chat never reached it, which is why that setting appeared to do nothing.

## The fix

The ternary is replaced with an explicit switch that gives native clan chat its own permission and lang key:

```csharp
// after
switch (channel)
{
    case ConVar.Chat.ChatChannel.Team:
        chatperm = "msg_teamchat";
        chatlang = "RUST_OnPlayerTeamChat";
        break;
    case ConVar.Chat.ChatChannel.Clan:
        chatperm = "msg_clanchat";
        chatlang = "RUST_OnPlayerClanChat";
        break;
    default:
        chatperm = "msg_chat";
        chatlang = "RUST_OnPlayerChat";
        break;
}
```

Three changes in `Rustcord.cs`:

| Location | Change |
| --- | --- |
| `OnPlayerChat` (~line 1979) | Ternary replaced with the switch above |
| `LoadDefaultMessages` (~line 1463) | New lang key `RUST_OnPlayerClanChat`, formatted `[CLAN] {playername}: {message}` |
| `LoadDefaultConfig` (~line 979) | `msg_clanchat` added to the second channel's default perms, mirroring `msg_teamchat` |

Routing for `Global`, `Local`, `Cards`, and `Server` is deliberately unchanged — they still fall through to `msg_chat` exactly as before, so this fix is limited to clan chat and introduces no other behavioural change.

## Upgrading

Drop the updated `Rustcord.cs` into `oxide/plugins/` and Oxide will hot-reload it. Oxide merges the new lang key into existing language files automatically.

**Existing installs need no config change to stop the leak.** Your current config has `msg_clanchat` on no channel, so after reload clan chat is relayed nowhere.

To *opt in* to clan chat relay — for a private staff or admin channel — add the permission to that channel's `perms` list:

```json
"perms": [
  "msg_teamchat",
  "msg_clanchat"
]
```

## Verification status

The fix is **not yet confirmed against a live server.** The reasoning is sound from the code, but the assumption it rests on — that clan chat on your Rust build arrives as `ChatChannel.Clan` — can only be proven at runtime. To confirm, temporarily add this near the top of `OnPlayerChat`:

```csharp
Puts($"[chatdebug] channel={channel} ({(int)channel}) player={player.displayName}");
```

Send one message in clan chat and one in global. Clan chat logging `channel=Clan (5)` confirms the fix. If it logs `Global (0)` instead, the leak has a different source and the diagnosis needs revisiting.

Note that `ConVar.Chat.ChatChannel.Clan` only exists on Rust builds that include the native clan system. On an older build the plugin will fail to compile on that line — which would itself indicate the leak is coming from somewhere other than native clan chat.

## Known issues (pre-existing, not addressed by this fork)

- **`OnPlayerChat` null guard is dead code.** `if (channel == null) return;` compares an enum against `null` via the lifted nullable operator, so it is always false and never fires. Whatever exception it was added to prevent is still unhandled.
- **`GetPlayerCache` writes to the wrong cache bucket.** The `CacheType.OnPlayerJoin` branch inserts into `cache[CacheType.OnPlayerDisconnected]`.
- **`!com` grants arbitrary console command execution from Discord.** Per-command role restrictions (`Commandroles`) default to empty, which means *no role restriction*. Any user who can post in a channel holding the `cmd_com` permission can run arbitrary server console commands. Restrict this explicitly in your config.
