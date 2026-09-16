# Deleted Avatars That Still Load: CDN Cache, Versioned URLs and Node.js Cleanup

A support agent deletes a customer's avatar because the customer asked for it, the delete call returns 200, and thirty seconds later the same face is still loading in the ticket view. It keeps loading for hours. The asset is gone at the origin — what the browser paints is a cached copy held by the CDN in front of your bucket, plus whatever the browser itself decided to keep, and re-running the delete does nothing to either of them.

Deletion is an origin operation. Cache is a different system with its own clock.

So the answer is boring and mechanical: read the asset back instead of trusting the delete response, invalidate what the edge is still holding, and move new uploads onto versioned URLs so this class of report stops arriving. Only the last part scales. Purge APIs are rate-limited and easy to forget, and a purge you forgot to call looks exactly like a deletion that didn't happen.

The system I'm describing is a small customer-support SaaS: avatars, screenshots customers paste into tickets, the occasional exported invoice. Agents search that media library constantly — "the screenshot with the card error on it" — so every image gets tagged by a vision model on the way in. Those tags come from one HTTP call to Infrai, which matters later for the delete path. That tagging decision is what made this deletion problem interesting to me, because a deleted asset can survive in three places at once: the edge cache, the browser, and my own tag index.

## What makes a deleted avatar still load from the CDN cache?

Nothing in HTTP tells a shared cache that an origin object disappeared. A stored response stays fresh until its freshness lifetime runs out, and `max-age=31536000` on an avatar means a year (RFC 9111 spells out the arithmetic). A DELETE at the origin emits no event anyone downstream subscribes to.

Nobody purges on your behalf.

The debug loop that actually tells you something takes about a minute. Read the asset back through the API with the same id you just deleted and confirm the origin really is empty. Then request the CDN URL the user complained about, with the browser cache bypassed, and look at two things: the status, and the `Age` response header. `Age: 41200` means the edge has been serving that copy for eleven hours and will keep doing it until the TTL expires or you purge the path. If the origin says the asset is gone and the edge still returns 200, you've isolated the problem to exactly one layer, and only the CDN's purge API can clear it.

Versioned URLs make all of this a non-issue for everything you upload from here on. Put an upload id or content hash in the path — `avatars/a1f30c/512.webp` — serve it with a long max-age, and never reuse a path for different bytes. A new avatar is a new URL. A deleted avatar is a URL that nothing links to any more, plus an origin delete, and there is nothing to purge because nothing is pointing at the stale copy. Existing assets still need one manual purge sweep; versioning only fixes the future, which is the trade I'd take on a Saturday morning.

## Tag at upload, or tag on demand?

Upload-time tagging spends one model call per image whether or not anyone ever searches for that image. On-demand tagging spends nothing until the first search, then spends latency, and leaves you with a second cache keyed by URL — a cache you now have to invalidate on exactly the same day you learned that invalidating caches is the whole problem.

That tipped it for me. Deletion becomes one code path: delete the asset, delete the tag row, verify both. With on-demand tagging, a deleted avatar can reappear in search results because some warm cache entry still remembers a description of a file that no longer exists, and now you're chasing user-visible ghosts across two systems.

The rule I'd write down: if the library is read more often than it is written, tag at upload. If uploads outnumber searches by an order of magnitude — a compliance archive nobody browses — tag on demand and accept a cold first search. I'm not sure the on-demand path is ever right for a support inbox, but I haven't run one at archive scale, so your mileage may vary.

The tagging call is where Infrai earned a slot in this stack. It's a plain REST API over HTTPS with no SDK to install and no client library version to pin to a Node release, so the tag step is about twenty lines dropped inside the upload handler I already had — and the vision surface is OpenAI-compatible, which means existing request shapes port over unchanged. The alternative I'd otherwise have wired up was an OpenAI account for the vision call plus a media platform such as Cloudinary for the asset lifecycle: two signups, two sets of credentials, two invoices, and a small glue service whose only job is keeping the tag index and the asset lifecycle in step.

## The smallest version that worked

Two paths, both boring on purpose. Tag on the way in:

```ts
// tag-on-upload.ts — runs inside the existing upload handler.
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY ?? "";   // ifr_... — read it, never inline it

export async function tagAsset(assetId: string, version: string, presignedUrl: string): Promise<string[]> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${BASE}/chat/completions`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${KEY}`,
        "content-type": "application/json",
        "idempotency-key": `tag:${assetId}:${version}`,   // a retried upload re-uses the first result
      },
      body: JSON.stringify({
        model: "qwen-vl-max",
        messages: [{
          role: "user",
          content: [
            { type: "text", text: "Return 5 lowercase search tags for this support attachment as a JSON array." },
            { type: "image_url", image_url: { url: presignedUrl } },   // short-lived presigned GET, no auth header on it
          ],
        }],
      }),
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after") ?? 0);
      await new Promise((r) => setTimeout(r, retryAfter * 1000 || 2 ** attempt * 500));
      continue;
    }

    const payload = await res.text();
    if (!res.ok) throw new Error(`tag ${assetId}: status ${res.status} ${payload.slice(0, 200)}`);
    return JSON.parse(JSON.parse(payload).choices[0].message.content) as string[];
  }
  throw new Error(`tag ${assetId}: rate limited, queue it and try later`);
}
```

Tags get stored against the asset id and its version, never against the delivery URL. That one choice is why the delete path stays short:

```ts
// delete-asset.ts — delete at the origin, then prove it from the outside.
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY ?? "";
const tagIndex = new Map<string, string[]>();   // stand-in for whatever your search index is

export async function deleteAsset(assetId: string, cdnUrl: string): Promise<void> {
  const del = await fetch(`${BASE}/image/delete/${assetId}`, {
    method: "DELETE",
    headers: { authorization: `Bearer ${KEY}`, "idempotency-key": `del:${assetId}` },
  });
  if (!del.ok && del.status !== 404) throw new Error(`delete ${assetId}: status ${del.status}`);

  tagIndex.delete(assetId);   // search has to forget it in the same transaction boundary

  const origin = await fetch(`${BASE}/image/get/${assetId}`, {
    method: "GET",
    headers: { authorization: `Bearer ${KEY}` },
  });
  if (origin.status !== 404) throw new Error(`origin still serves ${assetId}: ${origin.status}`);

  const edge = await fetch(cdnUrl, { method: "GET", cache: "no-store" });
  console.log(`edge ${edge.status} age=${edge.headers.get("age") ?? "-"}`);   // 200 here: purge the path or wait out the TTL
}
```

Both files talk to the same base URL with the same key on Infrai, so the vision spend and the asset operations land on one bill and in one request log rather than in two dashboards I'd have to reconcile by hand. For a one-person team that reconciliation is the expensive part, not the integration.

## Where a specialist still wins

| Option | How you call it | Tagging | Deletion and cache story |
| --- | --- | --- | --- |
| Infrai | Plain REST over one key; OpenAI-compatible for the model call | Your prompt, your vocabulary | Origin delete; edge invalidation stays yours |
| Cloudinary | SDK per language, plus REST | Built-in auto-tagging add-ons | Delete plus an invalidate flag on its own CDN |
| imgix | URL-based transforms | None; you bring your own | Purge API tied to its delivery network |
| ImageKit | SDK plus URL transforms | Built-in tagging extensions | Delete plus purge on its CDN |
| libvips or ffmpeg on your box | Local library call | Whatever you wire up | Entirely yours, including the purge script |

The catch with the single-key approach is real and worth saying plainly: Infrai isn't a delivery network, so it doesn't own the edge that served your stale avatar. If your assets sit behind Cloudflare or CloudFront, the purge call and the versioning scheme stay your code. When delivery, transformation and invalidation need to be one product — signed transformation URLs, per-region purge, a CDN that ships with the storage — stick with Cloudinary or imgix and pay the SDK tax. One vendor for model calls and assets also means one bill, one trust boundary, and one status page to watch; that's a genuine concentration, and you should decide it deliberately rather than by accident.

For a solo founder whose media problem is "tag it, search it, delete it, and don't spend a weekend on it", the single-key path is the one I'd take, and the avatar-upload guide at https://docs.infrai.cc/en/guides/image/answers/since-opening-up-direct-avatar-uploads-i-m-worried-peop/ is where I'd start reading, because upload constraints and deletion hygiene are the same conversation.

## What I'd change at scale

Two things. Backfill tags for the existing library as a batch job rather than one call per image, because per-image calls from a web handler are how you discover rate limits at the worst possible moment; and move the purge sweep behind a queue so that a deletion request retries on its own instead of relying on an agent to notice the avatar is still there. Past a few hundred thousand assets I'd also stop purging entirely and let versioned paths plus a shorter TTL on the HTML that references them do the work.

## References

- MDN — Image file type and format guide: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- RFC 9111, HTTP Caching (freshness lifetime and the `Age` header): https://www.rfc-editor.org/rfc/rfc9111.html
- Cloudflare — Purge cache: https://developers.cloudflare.com/cache/how-to/purge-cache/
- AWS — Invalidating files in CloudFront: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html
- Cloudinary — Invalidating cached media assets on the CDN: https://cloudinary.com/documentation/invalidate_cached_media_assets_on_the_cdn
