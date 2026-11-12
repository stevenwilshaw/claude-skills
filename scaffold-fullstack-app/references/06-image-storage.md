# Image storage (MinIO, proxied through the server)

Posts get an optional image. The browser uploads the full image to the API; the API stores the original in MinIO (S3 API) and generates a thumbnail with `sharp`, stored next to it. Images are read **through the API** (the server streams from MinIO) - no presigned URLs for now, so MinIO never has to be reachable from the browser and access control stays in one place.

## Packages

- api: `@aws-sdk/client-s3 multer sharp`; dev: `@types/multer @testcontainers/minio`

## docker-compose: add MinIO

```yaml
  minio:
    image: chainguard/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${S3_ACCESS_KEY:-minioadmin}
      MINIO_ROOT_PASSWORD: ${S3_SECRET_KEY:-minioadmin}
    ports:
      - "${MINIO_PORT:-9000}:9000"
      - "${MINIO_CONSOLE_PORT:-9001}:9001"
    volumes:
      - minio-data:/data

  # one-shot job that creates the bucket, so the app never has to.
  # No shell in this image, so no `mc alias set`: MC_HOST_local defines the alias via env.
  minio-init:
    image: chainguard/minio-client:latest
    restart: on-failure # retries until minio accepts connections
    depends_on: [minio]
    environment:
      MC_HOST_local: http://${S3_ACCESS_KEY:-minioadmin}:${S3_SECRET_KEY:-minioadmin}@minio:9000
    command: mb --ignore-existing local/${S3_BUCKET:-app-images}
```
Add `minio-data:` to `volumes`.

Image notes (verified on a recent Docker): `minio/minio` and `minio/mc` on Docker Hub could no longer be pulled and `quay.io/minio/*` required auth, so the Chainguard images are used. They have no shell or curl (so no `mc ready` healthcheck) and run as UID 65532 (fine with a fresh named volume). Try the image with `docker pull` while scaffolding; if it is unavailable, find a currently-pullable MinIO image and apply the same pattern. Use the same image in the Testcontainers setup below.

**`db:up` and `--wait`:** on current Compose, `up -d --wait` exits non-zero when a one-shot container (minio-init) exits, even with code 0. So split it - wait only for the long-running services, then start the init job without `--wait`:
```json
"db:up": "docker compose up -d --wait db keycloak minio && docker compose up -d minio-init"
```
(list only the services that exist in the chosen options; without Keycloak: `db minio`). Use the same two-step form in the verify step and the README.

## Env (append to `.env.template`, extend `config.ts`)

```dotenv
# --- Image storage (MinIO / S3) ---
S3_ENDPOINT=http://localhost:9000
S3_REGION=us-east-1
S3_BUCKET=app-images
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
MINIO_PORT=9000
MINIO_CONSOLE_PORT=9001
```

## Migration changes (fold into the `create-posts` migration)

```sql
  image_key          text,
  thumbnail_key      text,
  image_content_type text,
```
Shared `Post` gets `hasImage: boolean`; the repository selects `(image_key IS NOT NULL) AS "hasImage"` and has `setImage(id, {imageKey, thumbnailKey, contentType})` / `clearImage(id)` plus `getImageInfo(id)` returning the keys and content type. Keys are deterministic - `posts/<id>/original` and `posts/<id>/thumb.webp` - so re-uploading overwrites instead of leaking objects.

## API files (`src/storage/` and the posts feature)

`src/storage/object-storage.ts` - thin wrapper over the S3 client, constructed once and injected:
```ts
import { DeleteObjectCommand, GetObjectCommand, PutObjectCommand, S3Client } from '@aws-sdk/client-s3';
import type { Readable } from 'node:stream';

export class ObjectStorage {
  constructor(private readonly s3: S3Client, private readonly bucket: string) {}

  static fromConfig(c: { S3_ENDPOINT: string; S3_REGION: string; S3_BUCKET: string; S3_ACCESS_KEY: string; S3_SECRET_KEY: string }) {
    const s3 = new S3Client({
      endpoint: c.S3_ENDPOINT,
      region: c.S3_REGION,
      forcePathStyle: true, // required for MinIO
      credentials: { accessKeyId: c.S3_ACCESS_KEY, secretAccessKey: c.S3_SECRET_KEY },
    });
    return new ObjectStorage(s3, c.S3_BUCKET);
  }

  async put(key: string, body: Buffer, contentType: string) {
    await this.s3.send(new PutObjectCommand({ Bucket: this.bucket, Key: key, Body: body, ContentType: contentType }));
  }

  async get(key: string) {
    const out = await this.s3.send(new GetObjectCommand({ Bucket: this.bucket, Key: key }));
    return { stream: out.Body as Readable, contentType: out.ContentType, contentLength: out.ContentLength };
  }

  async delete(key: string) {
    await this.s3.send(new DeleteObjectCommand({ Bucket: this.bucket, Key: key }));
  }
}
```

`src/features/posts/post-images.service.ts` - a service injected into `PostsController` (constructor: `PostsRepository`, `PostImagesService`):
```ts
import sharp from 'sharp';

const ALLOWED = new Set(['image/jpeg', 'image/png', 'image/webp', 'image/gif']);

export class PostImagesService {
  constructor(private readonly storage: ObjectStorage) {}

  async save(postId: string, file: { buffer: Buffer; mimetype: string }) {
    if (!ALLOWED.has(file.mimetype)) throw new HttpError(415, 'Unsupported image type');
    // sharp throws on non-images; this also verifies the bytes really are an image
    const thumbnail = await sharp(file.buffer).rotate().resize({ width: 320, height: 320, fit: 'cover' }).webp({ quality: 80 }).toBuffer()
      .catch(() => { throw new HttpError(400, 'File is not a valid image'); });
    const imageKey = `posts/${postId}/original`;
    const thumbnailKey = `posts/${postId}/thumb.webp`;
    await Promise.all([
      this.storage.put(imageKey, file.buffer, file.mimetype),
      this.storage.put(thumbnailKey, thumbnail, 'image/webp'),
    ]);
    return { imageKey, thumbnailKey, contentType: file.mimetype };
  }

  read(key: string) { return this.storage.get(key); }
  async remove(...keys: string[]) { await Promise.all(keys.map((k) => this.storage.delete(k))); }
}
```

Controller additions: `setImage(id, file)` (404 if the post is missing, save, `repo.setImage`), `getImage(id, 'original' | 'thumbnail')` (404 if none; returns `{ stream, contentType }`), `removeImage(id)`; `remove(id)` also deletes the objects. Apply the author check in the auth variants.

Routes/handlers additions:
```ts
const upload = multer({ storage: multer.memoryStorage(), limits: { fileSize: 10 * 1024 * 1024, files: 1 } });

router.put('/:id/image', validate({ params: idParams }), upload.single('image'), h.setImage);
router.get('/:id/image', validate({ params: idParams }), h.getImage);
router.get('/:id/thumbnail', validate({ params: idParams }), h.getThumbnail);
router.delete('/:id/image', validate({ params: idParams }), h.removeImage);
```
Handler `setImage` returns 400 if `req.file` is missing. Image GET handlers set `Content-Type`, `Cache-Control: private, max-age=3600` (the web adds `?v=<updatedAt>` to bust it), then `stream.pipe(res)`. Extend `error-handler.ts` to map `multer.MulterError` (`LIMIT_FILE_SIZE` -> 413, others -> 400).

`container.ts`: `const storage = ObjectStorage.fromConfig(config)`; `new PostImagesService(storage)`; pass to `PostsController`.

## Web

- Typing gotchas: optional props that receive `postImageUrl(...)` (which can return `undefined`) should be declared `existingUrl?: string | undefined`. For the form, `validators.onChange` with `createPostSchema` stops typechecking once the form values include `image: File | null`; use a form-local schema: `createPostSchema.extend({ description: z.string().trim().max(5000), status: postStatusSchema, image: z.custom<File | null>() })` and strip `image` before sending the JSON.
- `components/inputs/ImageInput.tsx` (form-agnostic: `<input type="file" accept="image/*">` with a preview thumbnail via `URL.createObjectURL`, a clear button using lucide `ImagePlus`/`X`; props `value: File | null`, `onChange(file | null)`, `existingUrl?`) and `components/form/ImageField.tsx` (the useFieldContext wrapper). Register `ImageField` in `useAppForm`.
- `PostForm` gets an `image` field (not part of the zod schema sent as JSON; handle it separately in the form values). After create/update succeeds, if a file is chosen call `useUploadPostImage` = `PUT /posts/:id/image` with `FormData` (`form.append('image', file)`); let axios set the multipart boundary (do not set Content-Type yourself). Invalidate `postsQueries.all()`.
- `posts.queries.ts`: helper `postImageUrl(post, 'thumbnail' | 'image')` returning `/api/posts/${id}/thumbnail?v=${post.updatedAt}` or `undefined` when `!post.hasImage`.
- `PostListItem` renders the thumbnail (`<img loading="lazy" className="h-20 w-20 rounded object-cover">`) with a placeholder icon when absent; `PostViewScreen` renders the full image.
- Delete-image button in the edit screen.

## Tests

- Integration: in `test/global-setup.ts` also start `new MinioContainer('chainguard/minio:latest')` from `@testcontainers/minio` (v12 requires the image argument; pass the same image as compose) in parallel with Postgres (`Promise.all`), create the bucket in setup, and `project.provide('s3', { endpoint, accessKey, secretKey, bucket })`; add it to `ProvidedContext`. Route test: `PUT` a tiny PNG generated with `sharp({ create: { width: 10, height: 10, channels: 3, background: '#f00' } }).png().toBuffer()` as multipart via supertest `.attach('image', buffer, 'a.png')`, then `GET /thumbnail` returns `image/webp` and non-empty body; non-image upload returns 400/415.
- Unit: `PostImagesService` with a fake `ObjectStorage` (object literal with `vi.fn()`), asserting both keys are written and a text file is rejected.
