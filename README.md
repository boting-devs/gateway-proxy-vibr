# gateway-proxy

> This is a very hacky project, so it might stop working if Discord changes their API core. This is unlikely, but keep that in mind while using the proxy.

This is a proxy for Discord gateway connections - clients can connect to this proxy instead of the Discord Gateway and interact with it just like they would with the Discord Gateway.

The proxy connects to Discord instead of the client - allowing for zero-downtime client restarts while the proxy keeps its connections to the gateway open. The proxy won't invalidate your sessions or disconnect you (exceptions below).

## How?

It connects all shards to Discord upfront and mimics to be the actual API gateway.

When a client sends an `IDENTIFY` payload, it takes the shard ID specified and relays all events for that shard to the client.

It also sends you self-crafted, but valid `READY` and `GUILD_CREATE`/`GUILD_DELETE` payloads at startup to keep your guild state up to date, just like Discord does, even though it doesn't reconnect when you do internally.

Because the `IDENTIFY` is not actually controlled by the client side, activity data must be specified in the config file and will have no effect when sent in the client's `IDENTIFY` payload.

It uses a minimal algorithm to replace the sequence numbers in incoming payloads with fake sequence numbers that are valid for the clients, but does not need to parse the JSON for that.

## Configuration

Create a file `config.json` and fill in these fields as you wish:

```json
{
  "log_level": "info",
  "token": "",
  "intents": 32511,
  "port": 7878,
  "activity": {
    "type": 0,
    "name": "on shard {{shard}} with kubernetes"
  },
  "status": "idle",
  "backpressure": 100,
  "cache": {
    "channels": false,
    "presences": false,
    "emojis": false,
    "current_member": false,
    "members": false,
    "roles": false,
    "stage_instances": false,
    "stickers": false,
    "users": false,
    "voice_states": false
  }
}
```

If you have a fixed shard count, set `shards` to the amount of shards. If you're using twilight's HTTP-proxy, set `twilight_http_proxy` to the `ip:port` of the HTTP proxy.

Take special care when setting cache flags, only enable what you actually need. The proxy will tend to send more than Discord would, so double check what your bot depends on.

## Running

Compiling this from source isn't the most fun, you'll need a nightly Rust compiler with the rust-src component installed. Then run `cargo build --release --target=MY_RUSTC_TARGET`, where `MY_RUSTC_TARGET` is probably `x86_64-unknown-linux-gnu`.

Instead, I recommend running the Docker images that are prebuilt by CI.

`docker.io/gelbpunkt/gateway-proxy:latest` requires a Haswell-family CPU or newer (due to AVX2) and will perform best, `docker.io/gelbpunkt/gateway-proxy:sandybridge` requires a Sandy Bridge-family CPU or newer and will perform slightly worse.

To run the image, mount the config file at `/config.json`, for example:

```bash
docker run --rm -it -v /path/to/my/config.json:/config.json docker.io/gelbpunkt/gateway-proxy:latest
```

## Connecting

Connecting is fairly simple, just hardcode the gateway URL in your client to `ws://localhost:7878`. Make sure not to ratelimit your connections on your end.

**Important:** The proxy detects `zlib-stream` query parameters and `compress` fields in your `IDENTIFY` payloads and will encode packets if they are enabled, just like Discord. This comes with CPU overhead and is likely not desired in localhost networking. Make sure to disable this if so.

## Metrics

The proxy exposes Prometheus metrics at the `/metrics` endpoint. They contain event counters, cache size and shard latency histograms specific to each shard.

## Caveats

`RESUME`s will always result in a session invalidation because the proxy doesn't really track session IDs. Just reidentify, it's free.

## Known Issues / TODOs

- Maybe adhere to this: `When initially connecting, if you don't have the GUILD_PRESENCES Gateway Intent, or if the guild is over 75k members, it will only send members who are in voice, plus the member for you (the connecting user). Otherwise, if a guild has over large_threshold members (value in the Gateway Identify), it will only send members who are online, have a role, have a nickname, or are in a voice channel, and if it has under large_threshold members, it will send all members.` (though I think this is already wrong documentation from Discord, a 30k member guild above threshold already only sends the bot member)
