# Patrón de Almacenamiento de Objetos S3 en Backend

Esta referencia técnica describe el estándar arquitectónico para gestionar archivos adjuntos, comprobantes y documentos digitales en `apps/api` de forma desacoplada y compatible con cualquier proveedor S3.

---

## 1. Principios de Diseño y Desacoplamiento

1. **Independencia del Proveedor**: La aplicación utiliza el SDK estándar `@aws-sdk/client-s3`. El código fuente no diferencia si el backend corre contra **MinIO** en desarrollo local, o **Cloudflare R2 / AWS S3** en producción.
2. **Buffer en Memoria (Stateless API)**: Las peticiones `multipart/form-data` se procesan en memoria (`multer.memoryStorage()`) antes de enviarse al bucket. El contenedor de la API no almacena archivos temporales en su sistema de archivos local.
3. **Seguridad contra Path Traversal**: Las carpetas y nombres de archivo recibidos de clientes se sanitizan estrictamente para impedir inyecciones de retroceso de directorio (`..`) o caracteres de control.
4. **Nombres Inmutables y No Colisionables**: Cada archivo se renombra anteponiendo un timestamp y un hash aleatorio criptográfico:
   `folder/timestamp-cleanName-randomHex.extension`

---

## 2. Implementación del Servicio: `S3StorageService`

Ubicado en `apps/api/src/core/infrastructure/storage/s3Storage.service.ts`:

```typescript
import {
  S3Client,
  PutObjectCommand,
  GetObjectCommand,
  DeleteObjectCommand,
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { envs } from '../../../config/environment.js';
import crypto from 'crypto';

export interface UploadFileInput {
  buffer: Buffer;
  originalname: string;
  mimetype: string;
  folder?: string;
}

export interface UploadFileResult {
  key: string;
  url: string;
  originalname: string;
  mimetype: string;
  size: number;
}

function sanitizeFolder(folder?: string): string {
  if (!folder) return 'general';
  const cleaned = folder
    .replace(/\.\./g, '')
    .replace(/[^a-zA-Z0-9_\-\/]/g, '')
    .replace(/^\/+|\/+$/g, '');
  return cleaned || 'general';
}

export class S3StorageService {
  private client: S3Client;
  private bucket: string;

  constructor() {
    this.bucket = envs.S3_BUCKET;
    this.client = new S3Client({
      endpoint: envs.S3_ENDPOINT,
      region: envs.S3_REGION,
      credentials: {
        accessKeyId: envs.S3_ACCESS_KEY,
        secretAccessKey: envs.S3_SECRET_KEY,
      },
      // Imprescindible para MinIO: usa http://host:port/bucket/key en vez de DNS virtual host
      forcePathStyle: true,
    });
  }

  /**
   * Sube un buffer a S3/MinIO con clave única y sanitizada.
   */
  async uploadFile(input: UploadFileInput): Promise<UploadFileResult> {
    const { buffer, originalname, mimetype } = input;
    const cleanFolder = sanitizeFolder(input.folder);

    const extension = originalname.includes('.')
      ? originalname.split('.').pop()?.toLowerCase()
      : 'bin';
    const randomHex = crypto.randomBytes(6).toString('hex');
    const cleanBaseName = originalname
      .replace(/\.[^/.]+$/, '')
      .replace(/[^a-zA-Z0-9_-]/g, '_')
      .slice(0, 40);

    const key = `${cleanFolder}/${Date.now()}-${cleanBaseName}-${randomHex}.${extension}`;

    await this.client.send(
      new PutObjectCommand({
        Bucket: this.bucket,
        Key: key,
        Body: buffer,
        ContentType: mimetype,
      })
    );

    const url = `${envs.S3_ENDPOINT}/${this.bucket}/${key}`;

    return {
      key,
      url,
      originalname,
      mimetype,
      size: buffer.length,
    };
  }

  /**
   * Genera URL prefirmada temporal para visualización/descarga de archivos protegidos.
   */
  async getPresignedUrl(rawKeyOrUrl: string, expiresInSeconds: number = 3600): Promise<string> {
    let key = rawKeyOrUrl.trim();
    if (key.includes('..')) {
      throw new Error('Clave de archivo no válida: detección de retroceso de directorio.');
    }

    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    return await getSignedUrl(this.client, command, { expiresIn: expiresInSeconds });
  }

  /**
   * Elimina un archivo del bucket.
   */
  async deleteFile(rawKeyOrUrl: string): Promise<void> {
    let key = rawKeyOrUrl.trim();
    if (key.includes('..')) {
      throw new Error('Clave de archivo no válida: detección de retroceso de directorio.');
    }

    await this.client.send(
      new DeleteObjectCommand({
        Bucket: this.bucket,
        Key: key,
      })
    );
  }
}

export const storageService = new S3StorageService();
```

---

## 3. Integración en Rutas Express con Multer

```typescript
import { Router } from 'express';
import multer from 'multer';
import { storageService } from '../storage/s3Storage.service.js';
import { authenticate } from '../middlewares/auth.middleware.js';

const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 10 * 1024 * 1024 }, // Máximo 10 MB
  fileFilter: (_req, file, cb) => {
    const allowedMimes = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];
    if (allowedMimes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('Tipo de archivo no permitido. Solo imágenes o PDF.'));
    }
  },
});

export const uploadRouter = Router();

uploadRouter.post(
  '/upload',
  authenticate,
  upload.single('file'),
  async (req, res, next) => {
    try {
      if (!req.file) {
        return res.status(400).json({ error: 'No se envió ningún archivo.' });
      }

      const result = await storageService.uploadFile({
        buffer: req.file.buffer,
        originalname: req.file.originalname,
        mimetype: req.file.mimetype,
        folder: req.body.folder || 'adjuntos',
      });

      return res.status(201).json({ success: true, data: result });
    } catch (error) {
      next(error);
    }
  }
);
```

---

## 4. Diferencias entre Entornos

| Parámetro | Desarrollo Local (MinIO) | Producción (MinIO / Cloudflare R2 / AWS S3) |
| :--- | :--- | :--- |
| `S3_ENDPOINT` | `http://localhost:9000` | `https://s3.tudominio.com` o endpoint R2 |
| `S3_REGION` | `us-east-1` | `us-east-1` o `auto` (Cloudflare) |
| `forcePathStyle` | `true` (requerido por MinIO) | `true` para MinIO / `false` para AWS |
| `S3_BUCKET` | `finca-archivos` | `finca-archivos-prod` |
