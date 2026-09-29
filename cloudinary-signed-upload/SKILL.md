---
name: cloudinary-signed-upload
description: "Direct-to-Cloudinary signed file and image upload pattern for Laravel, Inertia.js v3, React 19, and TypeScript. Use this skill when implementing direct-to-CDN file uploads, Cloudinary signing endpoints, client-side canvas compression/resizing, handling Cloudinary public IDs, model accessors, and asset cleanup."
license: MIT
metadata:
  stack: "laravel-inertia-react"
---

# Direct-to-Cloudinary Signed Upload

This skill guides the implementation of a **direct-to-Cloudinary signed upload pattern** for Laravel 12+, Inertia.js v3, React 19, and TypeScript.

Under this architecture, file binaries never hit or pass through the Laravel backend. The client requests signed upload parameters from Laravel, performs client-side validation and canvas compression, uploads directly to Cloudinary's API, and stores only the resulting `public_id` in the database.

---

## 1. Prerequisites & Environment Setup

### Install Cloudinary PHP SDK
```bash
composer require cloudinary/cloudinary_php
```

### Environment Variables (`.env`)
```dotenv
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
```

### Configuration (`config/cloudinary.php`)
```php
<?php

return [
    'cloud_name' => env('CLOUDINARY_CLOUD_NAME'),
    'api_key' => env('CLOUDINARY_API_KEY'),
    'api_secret' => env('CLOUDINARY_API_SECRET'),
];
```

---

## 2. Backend Implementation

### A. Cloudinary Service (`app/Services/CloudinaryService.php`)
Manages signing requests, asset URL resolution, and asset deletion:

```php
<?php

namespace App\Services;

use Cloudinary\Api\ApiUtils;
use Cloudinary\Api\Upload\UploadApi;
use Cloudinary\Configuration\Configuration;
use Cloudinary\Utils;
use InvalidArgumentException;
use Throwable;

class CloudinaryService
{
    /**
     * White-listed upload folders. Add all application folders here.
     *
     * @var list<string>
     */
    public const FOLDERS = ['products', 'receipts', 'avatars', 'marketing', 'attachments'];

    public function cloudName(): string
    {
        return (string) config('cloudinary.cloud_name', '');
    }

    /**
     * Generate signed upload parameters for the direct client-to-Cloudinary upload.
     *
     * @return array{signature: string, timestamp: int, api_key: string, cloud_name: string, folder: string}
     */
    public function signUpload(string $folder): array
    {
        if (! in_array($folder, self::FOLDERS, true)) {
            throw new InvalidArgumentException("Invalid Cloudinary upload folder [{$folder}].");
        }

        $cloudName = $this->cloudName();
        $apiKey = (string) config('cloudinary.api_key', '');
        $apiSecret = (string) config('cloudinary.api_secret', '');

        if ($cloudName === '' || $apiKey === '' || $apiSecret === '') {
            throw new InvalidArgumentException('Cloudinary is not configured.');
        }

        $timestamp = time();
        $paramsToSign = [
            'folder' => $folder,
            'timestamp' => $timestamp,
        ];

        return [
            'signature' => ApiUtils::signParameters(
                $paramsToSign,
                $apiSecret,
                Utils::ALGO_SHA1,
                1,
            ),
            'timestamp' => $timestamp,
            'api_key' => $apiKey,
            'cloud_name' => $cloudName,
            'folder' => $folder,
        ];
    }

    /**
     * Build the public delivery URL for an image.
     */
    public function url(string $publicId): string
    {
        $cloudName = $this->cloudName();

        if ($cloudName === '') {
            return $publicId;
        }

        return 'https://res.cloudinary.com/'.$cloudName.'/image/upload/'.ltrim($publicId, '/');
    }

    /**
     * Best-effort asset deletion.
     */
    public function destroy(string $publicId): void
    {
        if ($publicId === '') {
            return;
        }

        $cloudName = $this->cloudName();
        $apiKey = (string) config('cloudinary.api_key', '');
        $apiSecret = (string) config('cloudinary.api_secret', '');

        if ($cloudName === '' || $apiKey === '' || $apiSecret === '') {
            return;
        }

        try {
            Configuration::instance([
                'cloud' => [
                    'cloud_name' => $cloudName,
                    'api_key' => $apiKey,
                    'api_secret' => $apiSecret,
                ],
            ]);

            (new UploadApi)->destroy($publicId);
        } catch (Throwable) {
            // Best-effort cleanup; model deletion should still proceed.
        }
    }
}
```

### B. Signing Controller (`app/Http/Controllers/CloudinarySignController.php`)
Enforces authorization per folder before issuing signatures:

```php
<?php

namespace App\Http\Controllers;

use App\Services\CloudinaryService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Validation\Rule;

class CloudinarySignController extends Controller
{
    public function __invoke(Request $request, CloudinaryService $cloudinary): JsonResponse
    {
        $validated = $request->validate([
            'folder' => ['required', 'string', Rule::in(CloudinaryService::FOLDERS)],
        ]);

        $folder = $validated['folder'];

        // Enforce folder-level permissions
        if (in_array($folder, ['products', 'marketing'], true)) {
            $user = $request->user();

            if ($user === null || ! $user->isAdmin()) {
                abort(403, 'Unauthorized to upload to this folder.');
            }
        }

        return response()->json($cloudinary->signUpload($folder));
    }
}
```

### C. Route Registration (`routes/web.php`)
```php
use App\Http\Controllers\CloudinarySignController;

Route::post('cloudinary/sign', CloudinarySignController::class)
    ->middleware(['web'])
    ->name('cloudinary.sign');
```

---

## 3. Client-Side TypeScript Implementation

Create `resources/js/lib/cloudinary.ts` to manage validation, client-side compression/downscaling, signing handshakes, and Cloudinary uploads.

```typescript
export type CloudinaryFolder = 'products' | 'receipts' | 'avatars' | 'marketing' | 'attachments';

export type CloudinaryUploadResult = {
    public_id: string;
    secure_url: string;
};

type SignResponse = {
    signature: string;
    timestamp: number;
    api_key: string;
    cloud_name: string;
    folder: string;
};

const MAX_RAW_IMAGE_BYTES = 40 * 1024 * 1024; // 40 MB max raw selection
const MAX_UPLOAD_BYTES = 8 * 1024 * 1024;     // 8 MB target upload size
const MAX_IMAGE_DIMENSION = 2048;             // Max width/height after resize

const ALLOWED_TYPES = new Set([
    'image/jpeg',
    'image/jpg',
    'image/png',
    'image/webp',
    'image/gif',
    'image/heic',
    'image/heif',
]);

const EXTENSION_MIME: Record<string, string> = {
    jpg: 'image/jpeg',
    jpeg: 'image/jpeg',
    png: 'image/png',
    webp: 'image/webp',
    gif: 'image/gif',
    heic: 'image/heic',
    heif: 'image/heif',
};

export class CloudinaryUploadError extends Error {
    constructor(message: string) {
        super(message);
        this.name = 'CloudinaryUploadError';
    }
}

function xsrfToken(): string {
    const match = document.cookie.match(/(?:^|;\s*)XSRF-TOKEN=([^;]+)/);
    return match ? decodeURIComponent(match[1]) : '';
}

function fileExtension(file: File): string {
    const name = file.name.trim();
    const dot = name.lastIndexOf('.');
    return dot < 0 || dot === name.length - 1 ? '' : name.slice(dot + 1).toLowerCase();
}

/**
 * Normalizes MIME type, falling back to extension for mobile pickers (e.g. iOS / Android).
 */
export function resolveImageMimeType(file: File): string {
    const declared = (file.type || '').toLowerCase().trim();
    if (ALLOWED_TYPES.has(declared)) {
        return declared === 'image/jpg' ? 'image/jpeg' : declared;
    }
    const fromExtension = EXTENSION_MIME[fileExtension(file)];
    return fromExtension || declared;
}

function formatMegabytes(bytes: number): string {
    return `${(bytes / (1024 * 1024)).toFixed(bytes >= 10 * 1024 * 1024 ? 0 : 1)} MB`;
}

export function assertValidImageFile(file: File): void {
    const mime = resolveImageMimeType(file);

    if (!ALLOWED_TYPES.has(mime)) {
        throw new CloudinaryUploadError(
            `"${file.name || 'This file'}" is not a supported image. Use JPEG, PNG, WebP, GIF, or HEIC.`,
        );
    }

    if (file.size <= 0) {
        throw new CloudinaryUploadError(`"${file.name || 'This file'}" is empty.`);
    }

    if (file.size > MAX_RAW_IMAGE_BYTES) {
        throw new CloudinaryUploadError(
            `"${file.name || 'This photo'}" is ${formatMegabytes(file.size)} and exceeds the ${formatMegabytes(MAX_RAW_IMAGE_BYTES)} limit.`,
        );
    }
}

async function canvasToJpegBlob(canvas: HTMLCanvasElement, quality: number): Promise<Blob> {
    const blob = await new Promise<Blob | null>((resolve) => {
        canvas.toBlob((result) => resolve(result), 'image/jpeg', quality);
    });

    if (!blob) {
        throw new CloudinaryUploadError('Could not process this photo. Try another image or JPEG screenshot.');
    }

    return blob;
}

/**
 * Resizes large images using canvas and applies progressive JPEG compression if over MAX_UPLOAD_BYTES.
 */
async function prepareImageForUpload(file: File): Promise<File> {
    const mime = resolveImageMimeType(file);

    if (mime === 'image/heic' || mime === 'image/heif' || mime === 'image/gif') {
        if (file.size > MAX_UPLOAD_BYTES) {
            throw new CloudinaryUploadError(
                `"${file.name}" is ${formatMegabytes(file.size)}. Please export as JPEG under ${formatMegabytes(MAX_UPLOAD_BYTES)}.`,
            );
        }
        return file;
    }

    if (file.size <= MAX_UPLOAD_BYTES) {
        return file;
    }

    let bitmap: ImageBitmap;
    try {
        bitmap = await createImageBitmap(file);
    } catch {
        throw new CloudinaryUploadError(
            `"${file.name}" is too large and cannot be resized by this browser.`,
        );
    }

    try {
        const scale = Math.min(1, MAX_IMAGE_DIMENSION / Math.max(bitmap.width, bitmap.height));
        const width = Math.max(1, Math.round(bitmap.width * scale));
        const height = Math.max(1, Math.round(bitmap.height * scale));

        const canvas = document.createElement('canvas');
        canvas.width = width;
        canvas.height = height;

        const context = canvas.getContext('2d');
        if (!context) {
            throw new CloudinaryUploadError('Could not prepare image canvas context.');
        }

        context.drawImage(bitmap, 0, 0, width, height);

        let quality = 0.85;
        let blob = await canvasToJpegBlob(canvas, quality);

        while (blob.size > MAX_UPLOAD_BYTES && quality > 0.45) {
            quality -= 0.1;
            blob = await canvasToJpegBlob(canvas, quality);
        }

        if (blob.size > MAX_UPLOAD_BYTES) {
            throw new CloudinaryUploadError(
                `"${file.name}" is still too large after compression (${formatMegabytes(blob.size)}).`,
            );
        }

        const baseName = file.name.replace(/\.[^.]+$/, '') || 'photo';
        return new File([blob], `${baseName}.jpg`, {
            type: 'image/jpeg',
            lastModified: Date.now(),
        });
    } finally {
        bitmap.close();
    }
}

async function requestSignature(folder: CloudinaryFolder): Promise<SignResponse> {
    const response = await fetch('/cloudinary/sign', {
        method: 'POST',
        credentials: 'same-origin',
        headers: {
            Accept: 'application/json',
            'Content-Type': 'application/json',
            'X-Requested-With': 'XMLHttpRequest',
            'X-XSRF-TOKEN': xsrfToken(),
        },
        body: JSON.stringify({ folder }),
    });

    if (!response.ok) {
        if (response.status === 401 || response.status === 403) {
            throw new CloudinaryUploadError('You are not authorized to upload images to this folder.');
        }
        if (response.status === 419) {
            throw new CloudinaryUploadError('Your session has expired. Refresh and try again.');
        }
        throw new CloudinaryUploadError('Failed to retrieve upload signature from the server.');
    }

    return (await response.json()) as SignResponse;
}

/**
 * Uploads a single image directly to Cloudinary.
 */
export async function uploadImageToCloudinary(
    file: File,
    folder: CloudinaryFolder,
): Promise<CloudinaryUploadResult> {
    assertValidImageFile(file);

    const prepared = await prepareImageForUpload(file);
    const signed = await requestSignature(folder);

    const formData = new FormData();
    formData.append('file', prepared);
    formData.append('api_key', signed.api_key);
    formData.append('timestamp', String(signed.timestamp));
    formData.append('signature', signed.signature);
    formData.append('folder', signed.folder);

    let uploadResponse: Response;
    try {
        uploadResponse = await fetch(
            `https://api.cloudinary.com/v1_1/${signed.cloud_name}/image/upload`,
            {
                method: 'POST',
                body: formData,
            },
        );
    } catch {
        throw new CloudinaryUploadError('Upload failed due to network drop. Check connection and retry.');
    }

    if (!uploadResponse.ok) {
        if (uploadResponse.status === 413) {
            throw new CloudinaryUploadError('Photo exceeds Cloudinary direct upload size limit.');
        }
        throw new CloudinaryUploadError('Image upload to Cloudinary failed.');
    }

    const payload = (await uploadResponse.json()) as {
        public_id?: string;
        secure_url?: string;
    };

    if (!payload.public_id || !payload.secure_url) {
        throw new CloudinaryUploadError('Cloudinary returned an incomplete response.');
    }

    return {
        public_id: payload.public_id,
        secure_url: payload.secure_url,
    };
}

/**
 * Sequentially uploads multiple images to Cloudinary.
 */
export async function uploadImagesToCloudinary(
    files: File[],
    folder: CloudinaryFolder,
): Promise<CloudinaryUploadResult[]> {
    const results: CloudinaryUploadResult[] = [];
    for (const file of files) {
        results.push(await uploadImageToCloudinary(file, folder));
    }
    return results;
}
```

---

## 4. Model & Database Integration

### Migration
Store only the `public_id` string:
```php
Schema::create('products', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('image_path')->nullable(); // Cloudinary public_id
    $table->json('gallery_paths')->nullable(); // Optional: multiple public_ids
    $table->timestamps();
});
```

### Model Accessor (`app/Models/Product.php`)
```php
<?php

namespace App\Models;

use App\Services\CloudinaryService;
use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class Product extends Model
{
    protected $fillable = ['name', 'image_path', 'gallery_paths'];

    protected function casts(): array
    {
        return [
            'gallery_paths' => 'array',
        ];
    }

    /**
     * Accessor for full CDN image URL.
     *
     * @return Attribute<string|null, never>
     */
    protected function imageUrl(): Attribute
    {
        return Attribute::get(
            fn (): ?string => $this->image_path
                ? app(CloudinaryService::class)->url($this->image_path)
                : null,
        );
    }
}
```

### Form Request Validation
Form requests validate the incoming string token(s) rather than binary files:
```php
public function rules(): array
{
    return [
        'name' => ['required', 'string', 'max:255'],
        'image_path' => ['required', 'string', 'max:512'],
        'gallery_paths' => ['nullable', 'array', 'max:10'],
        'gallery_paths.*' => ['required', 'string', 'max:512'],
    ];
}
```

---

## 5. React Component Pattern (Inertia.js v3)

Here is a typical single-image upload field integrated with Inertia's `useForm`:

```tsx
import React, { useState, useRef, type ChangeEvent } from 'react';
import { useForm } from '@inertiajs/react';
import { uploadImageToCloudinary, CloudinaryUploadError } from '@/lib/cloudinary';

export default function ProductCreateForm() {
    const { data, setData, post, processing, errors } = useForm({
        name: '',
        image_path: '',
    });

    const [previewUrl, setPreviewUrl] = useState<string | null>(null);
    const [uploading, setUploading] = useState<boolean>(false);
    const [uploadError, setUploadError] = useState<string | null>(null);
    const fileInputRef = useRef<HTMLInputElement>(null);

    async function handleFileSelect(e: ChangeEvent<HTMLInputElement>) {
        const file = e.target.files?.[0];
        if (!file) return;

        setUploadError(null);
        setUploading(true);

        // Immediate local preview
        const localUrl = URL.createObjectURL(file);
        setPreviewUrl(localUrl);

        try {
            const result = await uploadImageToCloudinary(file, 'products');
            setData('image_path', result.public_id);
        } catch (error) {
            setUploadError(
                error instanceof CloudinaryUploadError
                    ? error.message
                    : 'Failed to upload photo. Please try again.',
            );
            setPreviewUrl(null);
            setData('image_path', '');
        } finally {
            setUploading(false);
            if (fileInputRef.current) {
                fileInputRef.current.value = '';
            }
        }
    }

    function handleSubmit(e: React.FormEvent) {
        e.preventDefault();
        if (uploading || !data.image_path) return;
        post('/products');
    }

    return (
        <form onSubmit={handleSubmit} className="space-y-4">
            <div>
                <label className="block text-sm font-medium">Product Name</label>
                <input
                    type="text"
                    value={data.name}
                    onChange={(e) => setData('name', e.target.value)}
                    className="border rounded p-2 w-full"
                />
                {errors.name && <p className="text-red-500 text-xs mt-1">{errors.name}</p>}
            </div>

            <div>
                <label className="block text-sm font-medium">Product Image</label>
                <input
                    ref={fileInputRef}
                    type="file"
                    accept="image/jpeg,image/png,image/webp,image/gif,image/heic"
                    onChange={handleFileSelect}
                    disabled={uploading}
                    className="mt-1"
                />

                {uploading && (
                    <div className="flex items-center space-x-2 mt-2 text-sm text-blue-600">
                        <span className="animate-spin inline-block w-4 h-4 border-2 border-blue-600 border-t-transparent rounded-full" />
                        <span>Uploading directly to Cloudinary...</span>
                    </div>
                )}

                {uploadError && <p className="text-red-500 text-xs mt-2">{uploadError}</p>}

                {previewUrl && (
                    <div className="mt-3 relative w-32 h-32 border rounded overflow-hidden">
                        <img src={previewUrl} alt="Preview" className="w-full h-full object-cover" />
                    </div>
                )}

                {errors.image_path && (
                    <p className="text-red-500 text-xs mt-1">{errors.image_path}</p>
                )}
            </div>

            <button
                type="submit"
                disabled={processing || uploading || !data.image_path}
                className="bg-black text-white px-4 py-2 rounded disabled:opacity-50"
            >
                {processing ? 'Saving...' : 'Create Product'}
            </button>
        </form>
    );
}
```

---

## 6. Key Best Practices Checklist

- [ ] **Always validate folder permission**: Never sign an arbitrary folder from client input.
- [ ] **Do not expose API Secret**: Only `api_key` and SHA-1 signed hashes go to the browser; `CLOUDINARY_API_SECRET` must remain on the server.
- [ ] **Store only `public_id`**: Storing entire URLs in the database breaks domain or transformation changes. Store `public_id` and construct URLs dynamically via accessors.
- [ ] **Disable form submission while uploading**: Prevent users from clicking "Submit" while `uploading === true`.
- [ ] **Handle mobile image formats**: Normalize MIME types with file extensions because mobile browsers often omit `file.type`.
