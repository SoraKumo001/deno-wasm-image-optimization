# deno-wasm-image-optimization

## usage

## Parameters

| Name | Type   | Description |
| ---- | ------ | ----------- |
| url  | string | Image URL   |
| q    | number | Quality     |
| w    | number | Width       |

https://xxxx.deno.dev/?url=https://xxx.png&q=80&w=200

```ts
import { optimizeImage } from "npm:wasm-image-optimization/esm";

const isValidUrl = (url: string) => {
  try {
    new URL(url);
    return true;
  } catch (_e) {
    return false;
  }
};

const cache = await caches.open("image-cache");

Deno.serve(async (request) => {
  const cached = await cache.match(request);
  if (cached) {
    return cached;
  }

  const url = new URL(request.url);
  const params = url.searchParams;
  const type = ["webp", "png", "jpeg"].find((v) => v === params.get("type")) as
    | "webp"
    | "png"
    | "jpeg"
    | undefined;
  const accept = request.headers.get("accept");
  const isWebp =
    accept
      ?.split(",")
      .map((format) => format.trim())
      .some((format) => ["image/webp", "*/*", "image/*"].includes(format)) ??
    true;

  const imageUrl = params.get("url");
  if (!imageUrl || !isValidUrl(imageUrl)) {
    return new Response("url is required", { status: 400 });
  }

  url.searchParams.append("webp", isWebp.toString());
  const cacheKey = new Request(url.toString());
  const cachedResponse = await cache.match(cacheKey);
  if (cachedResponse) {
    return cachedResponse;
  }

  const width = params.get("w");
  const quality = params.get("q");

  const [srcImage, contentType] = await fetch(imageUrl)
    .then(async (res) =>
      res.ok
        ? ([await res.arrayBuffer(), res.headers.get("content-type")] as const)
        : []
    )
    .catch(() => []);

  if (!srcImage) {
    return new Response("image not found", { status: 404 });
  }

  if (contentType && ["image/svg+xml", "image/gif"].includes(contentType)) {
    const response = new Response(srcImage, {
      headers: {
        "Content-Type": contentType,
        "Cache-Control": "public, max-age=31536000, immutable",
      },
    });
    cache.put(request, response.clone());
    return response;
  }

  const format =
    type ?? (isWebp ? "webp" : contentType === "image/jpeg" ? "jpeg" : "png");
  const image = await optimizeImage({
    image: srcImage,
    width: width ? parseInt(width) : undefined,
    quality: quality ? parseInt(quality) : undefined,
    format,
  });
  const response = new Response(image, {
    headers: {
      "Content-Type": `image/${format}`,
      "Cache-Control": "public, max-age=31536000, immutable",
      date: new Date().toUTCString(),
    },
  });
  cache.put(request, response.clone());
  return response;
});
```
