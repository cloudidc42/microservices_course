# Part 95: Case Studies - Video Streaming Platform

## แพลตฟอร์ม Video Streaming - กรณีศึกษา

ในบทนี้เราจะออกแบบสถาปัตยกรรม Video Streaming Platform คล้ายกับ Netflix หรือ YouTube โดยใช้ Microservices

---

## 1. สถาปัตยกรรมภาพรวม

```
┌────────────────────────────────────────────────────────────────────┐
│                 Video Streaming Platform Architecture               │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Client ──► CDN (CloudFront) ──► API Gateway                        │
│               │                        │                            │
│               │          ┌─────────────┼──────────────┐             │
│               │          ▼             ▼              ▼             │
│               │   Video Service   User Service   Search Service     │
│               │          │             │              │             │
│               │          ▼             ▼              │             │
│               │   Transcoding     Watch History   Elasticsearch     │
│               │   Pipeline        Service                           │
│               │          │             │                            │
│               │          ▼             ▼                            │
│               │   Storage (S3)   Recommendation                     │
│               │          │       Engine                             │
│               │          ▼                                          │
│               └──► HLS Segments ◄── Live Streaming (WebRTC)        │
│                          │                                          │
│                   Analytics Service                                 │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## 2. Video Upload & Transcoding Pipeline

```typescript
// video-service/src/video-upload.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';
import { S3Client, CreateMultipartUploadCommand, UploadPartCommand, CompleteMultipartUploadCommand } from '@aws-sdk/client-s3';
import { v4 as uuidv4 } from 'uuid';

export interface VideoUploadSession {
  videoId: string;
  uploadId: string;
  uploadKey: string;
  parts: UploadedPart[];
  status: 'UPLOADING' | 'PROCESSING' | 'READY' | 'FAILED';
}

export interface UploadedPart {
  partNumber: number;
  etag: string;
}

export interface VideoMetadata {
  title: string;
  description: string;
  tags: string[];
  categoryId: string;
  visibility: 'PUBLIC' | 'PRIVATE' | 'UNLISTED';
  language: string;
  thumbnailUrl?: string;
}

@Injectable()
export class VideoUploadService {
  private readonly logger = new Logger(VideoUploadService.name);
  private readonly s3Client: S3Client;
  private readonly RAW_BUCKET = process.env.S3_RAW_BUCKET || 'videos-raw';
  private readonly PROCESSED_BUCKET = process.env.S3_PROCESSED_BUCKET || 'videos-processed';

  constructor(
    @InjectQueue('video-processing') private readonly processingQueue: Queue,
    private readonly videoRepository: VideoRepository,
    private readonly metadataService: VideoMetadataService,
  ) {
    this.s3Client = new S3Client({
      region: process.env.AWS_REGION || 'ap-southeast-1',
      credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
      },
    });
  }

  async initiateUpload(
    userId: string,
    metadata: VideoMetadata,
    fileSize: number,
    contentType: string,
  ): Promise<VideoUploadSession> {
    const videoId = uuidv4();
    const uploadKey = `uploads/${userId}/${videoId}/original`;

    // เริ่ม Multipart Upload ใน S3
    const createCommand = new CreateMultipartUploadCommand({
      Bucket: this.RAW_BUCKET,
      Key: uploadKey,
      ContentType: contentType,
      Metadata: {
        userId,
        videoId,
        title: metadata.title,
      },
    });

    const createResult = await this.s3Client.send(createCommand);

    // บันทึก Video Record
    const video = await this.videoRepository.create({
      id: videoId,
      userId,
      title: metadata.title,
      description: metadata.description,
      tags: metadata.tags,
      categoryId: metadata.categoryId,
      visibility: metadata.visibility,
      language: metadata.language,
      status: 'UPLOADING',
      uploadKey,
      fileSize,
      uploadId: createResult.UploadId,
    });

    return {
      videoId,
      uploadId: createResult.UploadId!,
      uploadKey,
      parts: [],
      status: 'UPLOADING',
    };
  }

  async uploadPart(
    videoId: string,
    uploadId: string,
    partNumber: number,
    body: Buffer,
  ): Promise<UploadedPart> {
    const video = await this.videoRepository.findById(videoId);
    
    if (!video) {
      throw new Error(`Video ${videoId} not found`);
    }

    const uploadCommand = new UploadPartCommand({
      Bucket: this.RAW_BUCKET,
      Key: video.uploadKey,
      UploadId: uploadId,
      PartNumber: partNumber,
      Body: body,
    });

    const result = await this.s3Client.send(uploadCommand);

    return {
      partNumber,
      etag: result.ETag!,
    };
  }

  async completeUpload(
    videoId: string,
    uploadId: string,
    parts: UploadedPart[],
  ): Promise<void> {
    const video = await this.videoRepository.findById(videoId);
    
    if (!video) {
      throw new Error(`Video ${videoId} not found`);
    }

    // Complete Multipart Upload
    const completeCommand = new CompleteMultipartUploadCommand({
      Bucket: this.RAW_BUCKET,
      Key: video.uploadKey,
      UploadId: uploadId,
      MultipartUpload: {
        Parts: parts.sort((a, b) => a.partNumber - b.partNumber).map(p => ({
          PartNumber: p.partNumber,
          ETag: p.etag,
        })),
      },
    });

    await this.s3Client.send(completeCommand);

    // อัปเดตสถานะ
    await this.videoRepository.updateStatus(videoId, 'PROCESSING');

    // เพิ่มใน Transcoding Queue
    await this.processingQueue.add('transcode', {
      videoId,
      sourceKey: video.uploadKey,
      sourceBucket: this.RAW_BUCKET,
      targetBucket: this.PROCESSED_BUCKET,
    }, {
      priority: 1,
      attempts: 3,
    });

    this.logger.log(`Video ${videoId} uploaded successfully, queued for transcoding`);
  }
}
```

---

## 3. Video Transcoding Service

```typescript
// transcoding-service/src/transcoding.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Process, Processor } from '@nestjs/bull';
import { Job } from 'bull';
import { S3Client, GetObjectCommand, PutObjectCommand } from '@aws-sdk/client-s3';
import * as ffmpeg from 'fluent-ffmpeg';
import * as path from 'path';
import * as fs from 'fs/promises';

export interface TranscodeProfile {
  name: string;
  resolution: string;
  bitrate: string;
  audioBitrate: string;
  width: number;
  height: number;
}

const TRANSCODE_PROFILES: TranscodeProfile[] = [
  { name: '1080p', resolution: '1920x1080', bitrate: '5000k', audioBitrate: '192k', width: 1920, height: 1080 },
  { name: '720p',  resolution: '1280x720',  bitrate: '2800k', audioBitrate: '128k', width: 1280, height: 720 },
  { name: '480p',  resolution: '854x480',   bitrate: '1400k', audioBitrate: '128k', width: 854,  height: 480 },
  { name: '360p',  resolution: '640x360',   bitrate: '800k',  audioBitrate: '96k',  width: 640,  height: 360 },
  { name: '240p',  resolution: '426x240',   bitrate: '400k',  audioBitrate: '64k',  width: 426,  height: 240 },
];

@Processor('video-processing')
@Injectable()
export class VideoTranscodingService {
  private readonly logger = new Logger(VideoTranscodingService.name);
  private readonly s3Client: S3Client;
  private readonly TEMP_DIR = '/tmp/video-processing';

  constructor(
    private readonly videoRepository: VideoRepository,
    private readonly eventEmitter: EventEmitter2,
  ) {
    this.s3Client = new S3Client({ region: process.env.AWS_REGION });
    this.ensureTempDir();
  }

  private async ensureTempDir(): Promise<void> {
    await fs.mkdir(this.TEMP_DIR, { recursive: true });
  }

  @Process('transcode')
  async transcodeVideo(job: Job<{
    videoId: string;
    sourceKey: string;
    sourceBucket: string;
    targetBucket: string;
  }>): Promise<void> {
    const { videoId, sourceKey, sourceBucket, targetBucket } = job.data;
    const workDir = path.join(this.TEMP_DIR, videoId);

    this.logger.log(`Starting transcoding for video ${videoId}`);

    try {
      await fs.mkdir(workDir, { recursive: true });

      // 1. Download original video from S3
      const originalPath = path.join(workDir, 'original');
      await this.downloadFromS3(sourceBucket, sourceKey, originalPath);

      // 2. Get video metadata
      const metadata = await this.getVideoMetadata(originalPath);
      await this.videoRepository.updateMetadata(videoId, metadata);

      // 3. สร้าง Thumbnail
      const thumbnailPath = path.join(workDir, 'thumbnail.jpg');
      await this.generateThumbnail(originalPath, thumbnailPath);
      const thumbnailKey = `thumbnails/${videoId}/thumbnail.jpg`;
      await this.uploadToS3(targetBucket, thumbnailKey, thumbnailPath, 'image/jpeg');

      // 4. Transcode ในทุก Quality Levels
      const completedProfiles: string[] = [];
      
      for (const profile of TRANSCODE_PROFILES) {
        // ข้ามถ้า Source resolution ต่ำกว่า Target
        if (metadata.height < profile.height) continue;

        await job.progress(
          (TRANSCODE_PROFILES.indexOf(profile) / TRANSCODE_PROFILES.length) * 80
        );

        const profileDir = path.join(workDir, profile.name);
        await fs.mkdir(profileDir, { recursive: true });

        // Transcode เป็น HLS
        await this.transcodeToHLS(originalPath, profileDir, profile);

        // Upload HLS Segments to S3
        await this.uploadHLSSegments(targetBucket, videoId, profile.name, profileDir);
        
        completedProfiles.push(profile.name);
        this.logger.log(`Completed ${profile.name} transcoding for video ${videoId}`);
      }

      // 5. สร้าง Master Playlist
      const masterPlaylistPath = path.join(workDir, 'master.m3u8');
      await this.generateMasterPlaylist(masterPlaylistPath, completedProfiles, videoId);
      const masterKey = `videos/${videoId}/master.m3u8`;
      await this.uploadToS3(targetBucket, masterKey, masterPlaylistPath, 'application/vnd.apple.mpegurl');

      // 6. อัปเดตสถานะเป็น READY
      await this.videoRepository.updateStatus(videoId, 'READY', {
        thumbnailUrl: `https://cdn.example.com/${thumbnailKey}`,
        masterPlaylistUrl: `https://cdn.example.com/${masterKey}`,
        availableQualities: completedProfiles,
        duration: metadata.duration,
      });

      await job.progress(100);

      // ส่ง Event ว่าพร้อมแล้ว
      this.eventEmitter.emit('video.transcoding_completed', { videoId });

      this.logger.log(`Video ${videoId} transcoding completed successfully`);
    } catch (error) {
      this.logger.error(`Transcoding failed for video ${videoId}: ${error.message}`, error.stack);
      await this.videoRepository.updateStatus(videoId, 'FAILED', {
        errorMessage: error.message,
      });
      throw error;
    } finally {
      // Cleanup temp files
      await fs.rm(workDir, { recursive: true, force: true });
    }
  }

  private async transcodeToHLS(
    inputPath: string,
    outputDir: string,
    profile: TranscodeProfile,
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      ffmpeg(inputPath)
        .videoCodec('libx264')
        .audioCodec('aac')
        .size(profile.resolution)
        .videoBitrate(profile.bitrate)
        .audioBitrate(profile.audioBitrate)
        .outputOptions([
          '-hls_time 6',          // แต่ละ Segment = 6 วินาที
          '-hls_list_size 0',     // เก็บทุก Segment ใน Playlist
          '-hls_segment_type mpegts',
          '-hls_flags independent_segments',
          `-hls_segment_filename ${outputDir}/segment%05d.ts`,
          '-f hls',
          '-preset fast',
          '-crf 23',
          '-movflags +faststart',
        ])
        .output(`${outputDir}/playlist.m3u8`)
        .on('end', () => resolve())
        .on('error', (err) => reject(err))
        .run();
    });
  }

  private async generateMasterPlaylist(
    outputPath: string,
    profiles: string[],
    videoId: string,
  ): Promise<void> {
    const bandwidthMap: Record<string, string> = {
      '1080p': 'BANDWIDTH=5000000,RESOLUTION=1920x1080',
      '720p':  'BANDWIDTH=2800000,RESOLUTION=1280x720',
      '480p':  'BANDWIDTH=1400000,RESOLUTION=854x480',
      '360p':  'BANDWIDTH=800000,RESOLUTION=640x360',
      '240p':  'BANDWIDTH=400000,RESOLUTION=426x240',
    };

    let content = '#EXTM3U\n#EXT-X-VERSION:3\n\n';
    
    for (const profile of profiles) {
      if (bandwidthMap[profile]) {
        content += `#EXT-X-STREAM-INF:${bandwidthMap[profile]}\n`;
        content += `https://cdn.example.com/videos/${videoId}/${profile}/playlist.m3u8\n\n`;
      }
    }

    await fs.writeFile(outputPath, content);
  }

  private async generateThumbnail(inputPath: string, outputPath: string): Promise<void> {
    return new Promise((resolve, reject) => {
      ffmpeg(inputPath)
        .screenshots({
          timestamps: ['10%'],
          filename: path.basename(outputPath),
          folder: path.dirname(outputPath),
          size: '1280x720',
        })
        .on('end', () => resolve())
        .on('error', (err) => reject(err));
    });
  }

  private async getVideoMetadata(filePath: string): Promise<{
    duration: number;
    width: number;
    height: number;
    bitrate: number;
    codec: string;
  }> {
    return new Promise((resolve, reject) => {
      ffmpeg.ffprobe(filePath, (err, data) => {
        if (err) return reject(err);
        
        const videoStream = data.streams.find(s => s.codec_type === 'video');
        resolve({
          duration: Math.round(data.format.duration || 0),
          width: videoStream?.width || 0,
          height: videoStream?.height || 0,
          bitrate: parseInt(data.format.bit_rate?.toString() || '0'),
          codec: videoStream?.codec_name || '',
        });
      });
    });
  }

  private async downloadFromS3(
    bucket: string,
    key: string,
    localPath: string,
  ): Promise<void> {
    const command = new GetObjectCommand({ Bucket: bucket, Key: key });
    const response = await this.s3Client.send(command);
    const writeStream = require('fs').createWriteStream(localPath);
    
    return new Promise((resolve, reject) => {
      (response.Body as any).pipe(writeStream)
        .on('finish', resolve)
        .on('error', reject);
    });
  }

  private async uploadToS3(
    bucket: string,
    key: string,
    localPath: string,
    contentType: string,
  ): Promise<void> {
    const fileContent = await fs.readFile(localPath);
    
    await this.s3Client.send(new PutObjectCommand({
      Bucket: bucket,
      Key: key,
      Body: fileContent,
      ContentType: contentType,
    }));
  }

  private async uploadHLSSegments(
    bucket: string,
    videoId: string,
    profileName: string,
    sourceDir: string,
  ): Promise<void> {
    const files = await fs.readdir(sourceDir);
    
    const uploadPromises = files.map(async (file) => {
      const localPath = path.join(sourceDir, file);
      const key = `videos/${videoId}/${profileName}/${file}`;
      const contentType = file.endsWith('.m3u8')
        ? 'application/vnd.apple.mpegurl'
        : 'video/mp2t';
      
      await this.uploadToS3(bucket, key, localPath, contentType);
    });

    await Promise.all(uploadPromises);
  }
}
```

---

## 4. CDN Integration & HLS Streaming

```typescript
// cdn-service/src/cdn.service.ts
import { Injectable, Logger } from '@nestjs/common';
import {
  CloudFrontClient,
  CreateInvalidationCommand,
} from '@aws-sdk/client-cloudfront';
import { GetSignedUrlConfig, Storage } from '@google-cloud/storage';

export interface StreamingUrl {
  masterPlaylistUrl: string;
  qualities: QualityUrl[];
  thumbnailUrl: string;
  subtitlesUrl?: string;
  expiresAt: Date;
}

export interface QualityUrl {
  quality: string;
  playlistUrl: string;
  bandwidth: number;
}

@Injectable()
export class CDNService {
  private readonly logger = new Logger(CDNService.name);
  private readonly cfClient: CloudFrontClient;
  private readonly CF_DISTRIBUTION_ID = process.env.CLOUDFRONT_DISTRIBUTION_ID!;
  private readonly CF_DOMAIN = process.env.CLOUDFRONT_DOMAIN!;
  private readonly CF_KEY_PAIR_ID = process.env.CLOUDFRONT_KEY_PAIR_ID!;

  constructor() {
    this.cfClient = new CloudFrontClient({
      region: 'us-east-1',
    });
  }

  async getStreamingUrls(
    videoId: string,
    userId: string,
    expiryMinutes: number = 120,
  ): Promise<StreamingUrl> {
    const expiresAt = new Date(Date.now() + expiryMinutes * 60 * 1000);
    
    // สร้าง Signed URL สำหรับ Protected Content
    const baseUrl = `${this.CF_DOMAIN}/videos/${videoId}`;
    
    const masterPlaylistUrl = await this.createSignedUrl(
      `${baseUrl}/master.m3u8`,
      expiresAt,
    );

    const thumbnailUrl = await this.createSignedUrl(
      `${this.CF_DOMAIN}/thumbnails/${videoId}/thumbnail.jpg`,
      expiresAt,
    );

    const qualities: QualityUrl[] = [];
    const qualityProfiles = [
      { quality: '1080p', bandwidth: 5000000 },
      { quality: '720p', bandwidth: 2800000 },
      { quality: '480p', bandwidth: 1400000 },
      { quality: '360p', bandwidth: 800000 },
    ];

    for (const profile of qualityProfiles) {
      const playlistUrl = await this.createSignedUrl(
        `${baseUrl}/${profile.quality}/playlist.m3u8`,
        expiresAt,
      );
      qualities.push({ ...profile, playlistUrl });
    }

    return {
      masterPlaylistUrl,
      qualities,
      thumbnailUrl,
      expiresAt,
    };
  }

  async invalidateCache(paths: string[]): Promise<void> {
    const command = new CreateInvalidationCommand({
      DistributionId: this.CF_DISTRIBUTION_ID,
      InvalidationBatch: {
        CallerReference: `invalidation-${Date.now()}`,
        Paths: {
          Quantity: paths.length,
          Items: paths.map(p => (p.startsWith('/') ? p : `/${p}`)),
        },
      },
    });

    await this.cfClient.send(command);
    this.logger.log(`Cache invalidated for ${paths.length} paths`);
  }

  private async createSignedUrl(url: string, expiresAt: Date): Promise<string> {
    // สร้าง CloudFront Signed URL
    const policy = JSON.stringify({
      Statement: [
        {
          Resource: url,
          Condition: {
            DateLessThan: { 'AWS:EpochTime': Math.floor(expiresAt.getTime() / 1000) },
          },
        },
      ],
    });

    // ในการใช้งานจริงต้อง Sign ด้วย Private Key
    // ตัวอย่างนี้แสดงแนวทาง
    const signature = this.signPolicy(policy);
    const encodedPolicy = Buffer.from(policy).toString('base64')
      .replace(/\+/g, '-').replace(/=/g, '_').replace(/\//g, '~');

    return `${url}?Policy=${encodedPolicy}&Signature=${signature}&Key-Pair-Id=${this.CF_KEY_PAIR_ID}`;
  }

  private signPolicy(policy: string): string {
    // ลงนาม Policy ด้วย RSA Private Key
    const crypto = require('crypto');
    const privateKey = process.env.CLOUDFRONT_PRIVATE_KEY!.replace(/\\n/g, '\n');
    
    const sign = crypto.createSign('RSA-SHA1');
    sign.update(policy);
    
    return sign.sign(privateKey, 'base64')
      .replace(/\+/g, '-').replace(/=/g, '_').replace(/\//g, '~');
  }
}
```

---

## 5. Recommendation Engine

```typescript
// recommendation/src/recommendation.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';

export interface RecommendationResult {
  videoId: string;
  score: number;
  reason: RecommendationReason;
}

export enum RecommendationReason {
  BASED_ON_HISTORY = 'BASED_ON_HISTORY',
  SIMILAR_CONTENT = 'SIMILAR_CONTENT',
  TRENDING = 'TRENDING',
  POPULAR_IN_CATEGORY = 'POPULAR_IN_CATEGORY',
  CONTINUE_WATCHING = 'CONTINUE_WATCHING',
}

@Injectable()
export class RecommendationService {
  private readonly logger = new Logger(RecommendationService.name);
  private readonly CACHE_TTL = 1800; // 30 นาที

  constructor(
    private readonly watchHistoryService: WatchHistoryService,
    private readonly videoService: VideoService,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  async getRecommendations(
    userId: string,
    limit: number = 20,
  ): Promise<RecommendationResult[]> {
    const cacheKey = `recommendations:${userId}`;
    const cached = await this.redis.get(cacheKey);
    
    if (cached) {
      return JSON.parse(cached);
    }

    const recommendations = await this.generateRecommendations(userId, limit);
    
    await this.redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(recommendations));
    
    return recommendations;
  }

  private async generateRecommendations(
    userId: string,
    limit: number,
  ): Promise<RecommendationResult[]> {
    const results: RecommendationResult[] = [];
    const usedVideoIds = new Set<string>();

    // 1. Continue Watching (ดูค้างไว้)
    const continueWatching = await this.watchHistoryService.getInProgress(userId, 5);
    for (const item of continueWatching) {
      results.push({
        videoId: item.videoId,
        score: 1.0,
        reason: RecommendationReason.CONTINUE_WATCHING,
      });
      usedVideoIds.add(item.videoId);
    }

    // 2. Based on Watch History
    const watchHistory = await this.watchHistoryService.getRecent(userId, 20);
    const tagScores = await this.calculateTagScores(watchHistory);
    const historyBased = await this.getVideosByTags(tagScores, usedVideoIds, 10);
    
    for (const video of historyBased) {
      results.push({
        videoId: video.id,
        score: video.score,
        reason: RecommendationReason.BASED_ON_HISTORY,
      });
      usedVideoIds.add(video.id);
    }

    // 3. Trending Videos
    const trending = await this.getTrendingVideos(usedVideoIds, 5);
    for (const video of trending) {
      results.push({
        videoId: video.id,
        score: 0.7,
        reason: RecommendationReason.TRENDING,
      });
      usedVideoIds.add(video.id);
    }

    // 4. Popular in Category
    const categories = watchHistory.map(h => h.categoryId).filter(Boolean);
    if (categories.length > 0) {
      const topCategory = this.getMostFrequent(categories);
      const popularInCategory = await this.getPopularInCategory(
        topCategory,
        usedVideoIds,
        5,
      );
      
      for (const video of popularInCategory) {
        results.push({
          videoId: video.id,
          score: 0.6,
          reason: RecommendationReason.POPULAR_IN_CATEGORY,
        });
        usedVideoIds.add(video.id);
      }
    }

    // เรียงลำดับตาม Score
    return results
      .sort((a, b) => b.score - a.score)
      .slice(0, limit);
  }

  async getTrendingVideos(
    excludeIds: Set<string>,
    limit: number,
  ): Promise<{ id: string; score: number }[]> {
    // ดึง Trending Videos จาก Redis (อัปเดตโดย Analytics Service)
    const trendingData = await this.redis.zrevrange('trending:videos', 0, limit * 2 - 1, 'WITHSCORES');
    
    const results: { id: string; score: number }[] = [];
    
    for (let i = 0; i < trendingData.length; i += 2) {
      const videoId = trendingData[i];
      const score = parseFloat(trendingData[i + 1]);
      
      if (!excludeIds.has(videoId)) {
        results.push({ id: videoId, score: score / 1000 });
        if (results.length >= limit) break;
      }
    }

    return results;
  }

  private async calculateTagScores(
    watchHistory: WatchHistoryItem[],
  ): Promise<Map<string, number>> {
    const tagScores = new Map<string, number>();
    
    for (let i = 0; i < watchHistory.length; i++) {
      const item = watchHistory[i];
      const recencyWeight = 1 - (i / watchHistory.length) * 0.5;
      const watchWeight = Math.min(item.watchPercentage / 100, 1);
      const itemScore = recencyWeight * watchWeight;
      
      for (const tag of item.tags || []) {
        tagScores.set(tag, (tagScores.get(tag) || 0) + itemScore);
      }
    }
    
    return tagScores;
  }

  private async getVideosByTags(
    tagScores: Map<string, number>,
    excludeIds: Set<string>,
    limit: number,
  ): Promise<{ id: string; score: number }[]> {
    const topTags = Array.from(tagScores.entries())
      .sort((a, b) => b[1] - a[1])
      .slice(0, 5)
      .map(([tag]) => tag);

    // ค้นหาวิดีโอที่มี Tags เหล่านี้
    const videos = await this.videoService.findByTags(topTags, limit * 2);
    
    return videos
      .filter(v => !excludeIds.has(v.id))
      .map(v => ({
        id: v.id,
        score: v.tags.reduce((sum, tag) => sum + (tagScores.get(tag) || 0), 0),
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, limit);
  }

  private async getPopularInCategory(
    categoryId: string,
    excludeIds: Set<string>,
    limit: number,
  ): Promise<{ id: string }[]> {
    const cacheKey = `popular:category:${categoryId}`;
    const cached = await this.redis.get(cacheKey);
    
    if (cached) {
      const videos = JSON.parse(cached);
      return videos.filter((v: any) => !excludeIds.has(v.id)).slice(0, limit);
    }

    const videos = await this.videoService.findPopularByCategory(categoryId, 20);
    await this.redis.setex(cacheKey, 3600, JSON.stringify(videos));
    
    return videos.filter(v => !excludeIds.has(v.id)).slice(0, limit);
  }

  private getMostFrequent<T>(arr: T[]): T {
    const counts = arr.reduce((acc, val) => {
      acc.set(val as any, (acc.get(val as any) || 0) + 1);
      return acc;
    }, new Map<any, number>());
    
    return Array.from(counts.entries()).sort((a, b) => b[1] - a[1])[0][0];
  }
}
```

---

## 6. Watch History Service

```typescript
// watch-history/src/watch-history.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';

export interface WatchHistoryItem {
  videoId: string;
  userId: string;
  watchPercentage: number;
  watchedDuration: number;
  lastWatchedAt: Date;
  tags?: string[];
  categoryId?: string;
}

@Injectable()
export class WatchHistoryService {
  private readonly logger = new Logger(WatchHistoryService.name);
  private readonly SESSION_TTL = 3600; // 1 ชั่วโมง

  constructor(
    @InjectRepository(WatchHistory)
    private readonly watchHistoryRepository: Repository<WatchHistory>,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  async recordWatch(
    userId: string,
    videoId: string,
    currentPosition: number,
    totalDuration: number,
  ): Promise<void> {
    const watchPercentage = Math.round((currentPosition / totalDuration) * 100);
    
    // บันทึกใน Redis Session (Real-time)
    const sessionKey = `watch:session:${userId}:${videoId}`;
    await this.redis.setex(sessionKey, this.SESSION_TTL, JSON.stringify({
      currentPosition,
      watchPercentage,
      updatedAt: new Date().toISOString(),
    }));

    // อัปเดตหรือสร้างใน Database
    const existing = await this.watchHistoryRepository.findOne({
      where: { userId, videoId },
    });

    if (existing) {
      // อัปเดตถ้าดูมากกว่าเดิม
      if (currentPosition > existing.watchedDuration) {
        await this.watchHistoryRepository.update(
          { userId, videoId },
          {
            watchedDuration: currentPosition,
            watchPercentage,
            lastWatchedAt: new Date(),
            watchCount: existing.watchCount + 1,
          },
        );
      }
    } else {
      await this.watchHistoryRepository.save({
        userId,
        videoId,
        watchedDuration: currentPosition,
        watchPercentage,
        lastWatchedAt: new Date(),
        watchCount: 1,
      });
    }

    // อัปเดต Analytics
    await this.updateVideoAnalytics(videoId, watchPercentage);
  }

  async getWatchPosition(userId: string, videoId: string): Promise<number> {
    // ตรวจ Redis ก่อน
    const sessionKey = `watch:session:${userId}:${videoId}`;
    const session = await this.redis.get(sessionKey);
    
    if (session) {
      return JSON.parse(session).currentPosition;
    }

    // ดึงจาก Database
    const history = await this.watchHistoryRepository.findOne({
      where: { userId, videoId },
    });

    return history?.watchedDuration || 0;
  }

  async getRecent(userId: string, limit: number = 20): Promise<WatchHistoryItem[]> {
    const history = await this.watchHistoryRepository.find({
      where: { userId },
      order: { lastWatchedAt: 'DESC' },
      take: limit,
      relations: ['video'],
    });

    return history.map(h => ({
      videoId: h.videoId,
      userId: h.userId,
      watchPercentage: h.watchPercentage,
      watchedDuration: h.watchedDuration,
      lastWatchedAt: h.lastWatchedAt,
      tags: h.video?.tags,
      categoryId: h.video?.categoryId,
    }));
  }

  async getInProgress(userId: string, limit: number = 5): Promise<WatchHistoryItem[]> {
    const history = await this.watchHistoryRepository.find({
      where: { userId },
      order: { lastWatchedAt: 'DESC' },
      take: 50,
    });

    return history
      .filter(h => h.watchPercentage > 5 && h.watchPercentage < 95)
      .slice(0, limit)
      .map(h => ({
        videoId: h.videoId,
        userId: h.userId,
        watchPercentage: h.watchPercentage,
        watchedDuration: h.watchedDuration,
        lastWatchedAt: h.lastWatchedAt,
      }));
  }

  private async updateVideoAnalytics(videoId: string, watchPercentage: number): Promise<void> {
    const analyticsKey = `analytics:video:${videoId}`;
    const todayKey = `analytics:video:${videoId}:${new Date().toISOString().split('T')[0]}`;
    
    await this.redis.incr(`${analyticsKey}:views`);
    await this.redis.incr(`${todayKey}:views`);
    
    // อัปเดต Trending Score
    const trendingKey = 'trending:videos';
    await this.redis.zincrby(trendingKey, 1, videoId);
    
    // ตั้งค่า Expiry สำหรับ Trending (7 วัน)
    await this.redis.expire(trendingKey, 86400 * 7);
    
    if (watchPercentage >= 80) {
      await this.redis.incr(`${analyticsKey}:completions`);
    }
  }
}
```

---

## 7. Search Service with Elasticsearch

```typescript
// search-service/src/search.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { ElasticsearchService } from '@nestjs/elasticsearch';

export interface VideoSearchQuery {
  query: string;
  filters?: {
    categoryId?: string;
    duration?: 'SHORT' | 'MEDIUM' | 'LONG'; // <4m, 4-20m, >20m
    uploadDate?: 'TODAY' | 'THIS_WEEK' | 'THIS_MONTH' | 'THIS_YEAR';
    sortBy?: 'RELEVANCE' | 'DATE' | 'VIEW_COUNT' | 'RATING';
  };
  page?: number;
  limit?: number;
}

export interface VideoSearchResult {
  videos: VideoSearchItem[];
  total: number;
  page: number;
  totalPages: number;
  suggestions?: string[];
}

export interface VideoSearchItem {
  id: string;
  title: string;
  description: string;
  thumbnailUrl: string;
  channelName: string;
  viewCount: number;
  duration: number;
  uploadedAt: Date;
  tags: string[];
  score: number;
}

@Injectable()
export class VideoSearchService {
  private readonly logger = new Logger(VideoSearchService.name);
  private readonly INDEX_NAME = 'videos';

  constructor(private readonly esService: ElasticsearchService) {}

  async indexVideo(video: any): Promise<void> {
    await this.esService.index({
      index: this.INDEX_NAME,
      id: video.id,
      document: {
        id: video.id,
        title: video.title,
        description: video.description,
        tags: video.tags,
        categoryId: video.categoryId,
        channelName: video.channelName,
        channelId: video.channelId,
        viewCount: video.viewCount,
        duration: video.duration,
        thumbnailUrl: video.thumbnailUrl,
        uploadedAt: video.createdAt,
        language: video.language,
        isPublic: video.visibility === 'PUBLIC',
        // Thai text fields
        titleTh: video.titleTh,
        descriptionTh: video.descriptionTh,
      },
    });
  }

  async search(query: VideoSearchQuery): Promise<VideoSearchResult> {
    const page = query.page || 1;
    const limit = query.limit || 20;
    const from = (page - 1) * limit;

    const esQuery: any = {
      bool: {
        must: [
          {
            multi_match: {
              query: query.query,
              fields: [
                'title^3',
                'title.ngram^2',
                'titleTh^3',
                'description^1',
                'tags^2',
                'channelName^1.5',
              ],
              type: 'best_fields',
              fuzziness: 'AUTO',
            },
          },
          { term: { isPublic: true } },
        ],
        filter: [],
      },
    };

    // Apply Filters
    if (query.filters?.categoryId) {
      esQuery.bool.filter.push({ term: { categoryId: query.filters.categoryId } });
    }

    if (query.filters?.duration) {
      const durationRanges = {
        SHORT: { lte: 240 },
        MEDIUM: { gte: 240, lte: 1200 },
        LONG: { gte: 1200 },
      };
      esQuery.bool.filter.push({
        range: { duration: durationRanges[query.filters.duration] },
      });
    }

    if (query.filters?.uploadDate) {
      const dateRanges = {
        TODAY: 'now/d',
        THIS_WEEK: 'now-7d/d',
        THIS_MONTH: 'now-30d/d',
        THIS_YEAR: 'now-365d/d',
      };
      esQuery.bool.filter.push({
        range: {
          uploadedAt: { gte: dateRanges[query.filters.uploadDate] },
        },
      });
    }

    // Sort
    let sort: any[] = [{ _score: 'desc' }];
    
    if (query.filters?.sortBy === 'DATE') {
      sort = [{ uploadedAt: 'desc' }];
    } else if (query.filters?.sortBy === 'VIEW_COUNT') {
      sort = [{ viewCount: 'desc' }];
    }

    const response = await this.esService.search<any>({
      index: this.INDEX_NAME,
      from,
      size: limit,
      query: esQuery,
      sort,
      highlight: {
        fields: {
          title: { number_of_fragments: 1 },
          description: { number_of_fragments: 2, fragment_size: 150 },
        },
      },
      suggest: {
        didYouMean: {
          text: query.query,
          phrase: {
            field: 'title',
            size: 3,
          },
        },
      },
    });

    const videos = response.hits.hits.map((hit: any) => ({
      id: hit._id,
      title: hit.highlight?.title?.[0] || hit._source.title,
      description: hit.highlight?.description?.[0] || hit._source.description,
      thumbnailUrl: hit._source.thumbnailUrl,
      channelName: hit._source.channelName,
      viewCount: hit._source.viewCount,
      duration: hit._source.duration,
      uploadedAt: hit._source.uploadedAt,
      tags: hit._source.tags,
      score: hit._score,
    }));

    const suggestions = (response as any).suggest?.didYouMean?.[0]?.options?.map(
      (opt: any) => opt.text
    ) || [];

    return {
      videos,
      total: typeof response.hits.total === 'number'
        ? response.hits.total
        : response.hits.total?.value || 0,
      page,
      totalPages: Math.ceil(
        (typeof response.hits.total === 'number' ? response.hits.total : response.hits.total?.value || 0) / limit
      ),
      suggestions,
    };
  }

  async createIndex(): Promise<void> {
    const exists = await this.esService.indices.exists({ index: this.INDEX_NAME });
    
    if (!exists) {
      await this.esService.indices.create({
        index: this.INDEX_NAME,
        mappings: {
          properties: {
            title: {
              type: 'text',
              analyzer: 'standard',
              fields: {
                ngram: { type: 'text', analyzer: 'ngram_analyzer' },
                keyword: { type: 'keyword' },
              },
            },
            titleTh: { type: 'text', analyzer: 'thai' },
            description: { type: 'text', analyzer: 'standard' },
            descriptionTh: { type: 'text', analyzer: 'thai' },
            tags: { type: 'keyword' },
            categoryId: { type: 'keyword' },
            channelName: { type: 'text', fields: { keyword: { type: 'keyword' } } },
            viewCount: { type: 'long' },
            duration: { type: 'integer' },
            uploadedAt: { type: 'date' },
            isPublic: { type: 'boolean' },
          },
        },
        settings: {
          analysis: {
            analyzer: {
              ngram_analyzer: {
                type: 'custom',
                tokenizer: 'ngram_tokenizer',
                filter: ['lowercase'],
              },
            },
            tokenizer: {
              ngram_tokenizer: {
                type: 'ngram',
                min_gram: 2,
                max_gram: 3,
              },
            },
          },
        },
      });

      this.logger.log(`Created Elasticsearch index: ${this.INDEX_NAME}`);
    }
  }
}
```

---

## 8. Live Streaming with WebRTC

```typescript
// live-streaming/src/live-stream.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { WebSocketGateway, WebSocketServer, SubscribeMessage } from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

export interface LiveStream {
  streamId: string;
  userId: string;
  title: string;
  description: string;
  viewerCount: number;
  startedAt: Date;
  status: 'LIVE' | 'ENDED';
  chatEnabled: boolean;
  rtmpUrl?: string;
  playbackUrl?: string;
}

export interface LiveStreamOffer {
  streamId: string;
  sdp: string;
  type: 'offer' | 'answer';
}

export interface IceCandidate {
  streamId: string;
  candidate: RTCIceCandidateInit;
}

@WebSocketGateway({ namespace: 'live' })
@Injectable()
export class LiveStreamGateway {
  @WebSocketServer()
  private server: Server;
  
  private readonly logger = new Logger(LiveStreamGateway.name);
  
  // เก็บ Peer Connections
  private readonly streams = new Map<string, Set<string>>(); // streamId -> viewerSockets
  private readonly broadcasters = new Map<string, string>(); // streamId -> broadcasterSocket

  constructor(
    private readonly liveStreamService: LiveStreamService,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  @SubscribeMessage('start_broadcast')
  async handleStartBroadcast(
    client: Socket,
    data: { title: string; description: string },
  ): Promise<void> {
    const userId = client.data.userId;
    const stream = await this.liveStreamService.createStream({
      userId,
      title: data.title,
      description: data.description,
    });

    this.broadcasters.set(stream.streamId, client.id);
    this.streams.set(stream.streamId, new Set());
    
    client.join(`stream:${stream.streamId}`);
    
    // เผยแพร่ว่ามี Live Stream ใหม่
    this.server.emit('new_live_stream', {
      streamId: stream.streamId,
      title: stream.title,
      channelName: stream.channelName,
    });

    client.emit('broadcast_started', {
      streamId: stream.streamId,
      rtmpUrl: `rtmp://live.example.com/live/${stream.streamKey}`,
    });

    this.logger.log(`User ${userId} started stream ${stream.streamId}`);
  }

  @SubscribeMessage('join_stream')
  async handleJoinStream(
    client: Socket,
    data: { streamId: string },
  ): Promise<void> {
    const { streamId } = data;
    
    const stream = await this.liveStreamService.findById(streamId);
    if (!stream || stream.status !== 'LIVE') {
      client.emit('error', { message: 'Stream not available' });
      return;
    }

    client.join(`stream:${streamId}`);
    
    const viewers = this.streams.get(streamId);
    if (viewers) {
      viewers.add(client.id);
    }

    // อัปเดต Viewer Count
    await this.liveStreamService.incrementViewerCount(streamId);
    
    const viewerCount = viewers?.size || 0;
    this.server.to(`stream:${streamId}`).emit('viewer_count_update', { viewerCount });

    client.emit('joined_stream', {
      streamId,
      viewerCount,
      playbackUrl: stream.playbackUrl,
    });

    this.logger.log(`Viewer ${client.id} joined stream ${streamId}`);
  }

  @SubscribeMessage('webrtc_offer')
  async handleWebRTCOffer(client: Socket, data: LiveStreamOffer): Promise<void> {
    const broadcasterSocketId = this.broadcasters.get(data.streamId);
    
    if (!broadcasterSocketId) {
      client.emit('error', { message: 'Broadcaster not found' });
      return;
    }

    // ส่ง SDP Offer ไปยัง Broadcaster
    this.server.to(broadcasterSocketId).emit('viewer_offer', {
      viewerSocketId: client.id,
      sdp: data.sdp,
    });
  }

  @SubscribeMessage('webrtc_answer')
  async handleWebRTCAnswer(
    client: Socket,
    data: { viewerSocketId: string; streamId: string; sdp: string },
  ): Promise<void> {
    // ส่ง SDP Answer กลับไปยัง Viewer
    this.server.to(data.viewerSocketId).emit('broadcaster_answer', {
      sdp: data.sdp,
    });
  }

  @SubscribeMessage('ice_candidate')
  async handleIceCandidate(client: Socket, data: IceCandidate & { targetSocketId?: string }): Promise<void> {
    if (data.targetSocketId) {
      // ส่งไปยัง Specific Socket
      this.server.to(data.targetSocketId).emit('ice_candidate', {
        candidate: data.candidate,
        fromSocketId: client.id,
      });
    } else {
      // ส่งไปยัง Broadcaster
      const broadcasterSocketId = this.broadcasters.get(data.streamId);
      if (broadcasterSocketId) {
        this.server.to(broadcasterSocketId).emit('ice_candidate', {
          candidate: data.candidate,
          fromSocketId: client.id,
        });
      }
    }
  }

  @SubscribeMessage('chat_message')
  async handleChatMessage(
    client: Socket,
    data: { streamId: string; message: string },
  ): Promise<void> {
    const userId = client.data.userId;
    const user = await this.userService.findById(userId);
    
    // ตรวจสอบ Chat สแปม
    const spamKey = `chat:spam:${userId}:${data.streamId}`;
    const messageCount = await this.redis.incr(spamKey);
    await this.redis.expire(spamKey, 10);
    
    if (messageCount > 5) {
      client.emit('chat_error', { message: 'Too many messages, please slow down' });
      return;
    }

    const chatMessage = {
      userId,
      username: user.username,
      avatarUrl: user.avatarUrl,
      message: data.message,
      timestamp: new Date().toISOString(),
      isSubscriber: user.isChannelSubscriber,
    };

    // ส่ง Message ไปยังทุกคนใน Stream
    this.server.to(`stream:${data.streamId}`).emit('chat_message', chatMessage);
    
    // บันทึก Chat History
    await this.redis.lpush(
      `chat:history:${data.streamId}`,
      JSON.stringify(chatMessage),
    );
    await this.redis.ltrim(`chat:history:${data.streamId}`, 0, 199);
  }

  @SubscribeMessage('end_broadcast')
  async handleEndBroadcast(client: Socket, data: { streamId: string }): Promise<void> {
    const { streamId } = data;
    
    await this.liveStreamService.endStream(streamId);
    
    // แจ้งทุกคนใน Stream
    this.server.to(`stream:${streamId}`).emit('stream_ended', { streamId });
    
    this.broadcasters.delete(streamId);
    this.streams.delete(streamId);

    this.logger.log(`Stream ${streamId} ended`);
  }
}
```

---

## 9. Analytics Service

```typescript
// analytics/src/analytics.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';
import { Cron } from '@nestjs/schedule';

export interface VideoAnalyticsEvent {
  eventType: 'VIEW' | 'LIKE' | 'DISLIKE' | 'SHARE' | 'COMMENT' | 'SUBSCRIBE';
  videoId: string;
  userId?: string;
  sessionId: string;
  timestamp: Date;
  metadata?: Record<string, any>;
}

@Injectable()
export class AnalyticsService {
  private readonly logger = new Logger(AnalyticsService.name);
  private readonly eventBuffer: VideoAnalyticsEvent[] = [];
  private readonly BATCH_SIZE = 100;
  private readonly FLUSH_INTERVAL = 5000; // 5 วินาที

  constructor(
    @InjectRepository(VideoAnalytics)
    private readonly analyticsRepository: Repository<VideoAnalytics>,
    @InjectRedis() private readonly redis: Redis,
  ) {
    // Flush Buffer เป็นระยะ
    setInterval(() => this.flushBuffer(), this.FLUSH_INTERVAL);
  }

  async trackEvent(event: VideoAnalyticsEvent): Promise<void> {
    // เพิ่มใน Buffer
    this.eventBuffer.push(event);
    
    // อัปเดต Real-time Counters ใน Redis
    await this.updateRealtimeCounters(event);
    
    // Flush ถ้า Buffer เต็ม
    if (this.eventBuffer.length >= this.BATCH_SIZE) {
      await this.flushBuffer();
    }
  }

  private async updateRealtimeCounters(event: VideoAnalyticsEvent): Promise<void> {
    const date = new Date().toISOString().split('T')[0];
    const hour = new Date().getHours();
    
    const keys = {
      total: `analytics:${event.videoId}:${event.eventType.toLowerCase()}:total`,
      daily: `analytics:${event.videoId}:${event.eventType.toLowerCase()}:${date}`,
      hourly: `analytics:${event.videoId}:${event.eventType.toLowerCase()}:${date}:${hour}`,
    };

    await this.redis.incr(keys.total);
    await this.redis.incr(keys.daily);
    await this.redis.incr(keys.hourly);
    
    // Expire Daily/Hourly (เก็บ 30 วัน)
    await this.redis.expire(keys.daily, 86400 * 30);
    await this.redis.expire(keys.hourly, 86400 * 7);

    // อัปเดต Trending Score สำหรับ Views
    if (event.eventType === 'VIEW') {
      const viewWeight = 1;
      const likeWeight = 5;
      const shareWeight = 10;
      
      await this.redis.zincrby('trending:videos', viewWeight, event.videoId);
      await this.redis.expire('trending:videos', 86400 * 7);
    }
  }

  private async flushBuffer(): Promise<void> {
    if (this.eventBuffer.length === 0) return;
    
    const events = this.eventBuffer.splice(0, this.BATCH_SIZE);
    
    try {
      await this.analyticsRepository.createQueryBuilder()
        .insert()
        .into(VideoAnalytics)
        .values(events)
        .execute();
    } catch (error) {
      this.logger.error(`Failed to flush analytics buffer: ${error.message}`);
      // คืนกลับ Buffer
      this.eventBuffer.unshift(...events);
    }
  }

  // อัปเดต Trending ทุกชั่วโมง
  @Cron('0 * * * *')
  async updateTrendingVideos(): Promise<void> {
    // คำนวณ Trending Score ใหม่โดยรวม Views ในช่วง 24 ชั่วโมงที่ผ่านมา
    const recentViews = await this.analyticsRepository
      .createQueryBuilder('a')
      .where('a.eventType = :type', { type: 'VIEW' })
      .andWhere('a.timestamp >= :since', { since: new Date(Date.now() - 86400000) })
      .groupBy('a.videoId')
      .select(['a.videoId', 'COUNT(*) as viewCount'])
      .getRawMany();

    // อัปเดต Redis Sorted Set
    if (recentViews.length > 0) {
      await this.redis.del('trending:videos:new');
      
      for (const item of recentViews) {
        await this.redis.zadd('trending:videos:new', item.viewCount, item.videoId);
      }
      
      // Swap
      await this.redis.rename('trending:videos:new', 'trending:videos');
      await this.redis.expire('trending:videos', 86400 * 7);
    }

    this.logger.log(`Updated trending videos with ${recentViews.length} entries`);
  }

  async getVideoStats(videoId: string): Promise<{
    totalViews: number;
    totalLikes: number;
    dailyViews: { date: string; count: number }[];
    averageWatchPercentage: number;
  }> {
    const [totalViews, totalLikes] = await Promise.all([
      this.redis.get(`analytics:${videoId}:view:total`),
      this.redis.get(`analytics:${videoId}:like:total`),
    ]);

    // ดึง Daily Views ย้อนหลัง 30 วัน
    const dailyViews = [];
    for (let i = 0; i < 30; i++) {
      const date = new Date(Date.now() - i * 86400000).toISOString().split('T')[0];
      const count = await this.redis.get(`analytics:${videoId}:view:${date}`);
      dailyViews.push({ date, count: parseInt(count || '0') });
    }

    return {
      totalViews: parseInt(totalViews || '0'),
      totalLikes: parseInt(totalLikes || '0'),
      dailyViews: dailyViews.reverse(),
      averageWatchPercentage: 0,
    };
  }
}
```

---

## 10. Kubernetes Deployment

```yaml
# k8s/transcoding-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: transcoding-service
  namespace: video-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: transcoding-service
  template:
    metadata:
      labels:
        app: transcoding-service
    spec:
      containers:
        - name: transcoding-service
          image: your-registry/transcoding-service:1.0.0
          resources:
            requests:
              cpu: 2
              memory: 4Gi
            limits:
              cpu: 4
              memory: 8Gi
          env:
            - name: AWS_REGION
              value: ap-southeast-1
            - name: S3_RAW_BUCKET
              value: my-videos-raw
            - name: S3_PROCESSED_BUCKET
              value: my-videos-processed
          volumeMounts:
            - name: temp-storage
              mountPath: /tmp/video-processing
      volumes:
        - name: temp-storage
          emptyDir:
            sizeLimit: 50Gi
      nodeSelector:
        # ใช้ Node ที่มี CPU/GPU สูง
        node-type: compute-intensive
---
# HPA สำหรับ Transcoding
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: transcoding-hpa
  namespace: video-platform
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: transcoding-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: External
      external:
        metric:
          name: bull_queue_waiting
          selector:
            matchLabels:
              queue: video-processing
        target:
          type: AverageValue
          averageValue: "5"
```

```yaml
# k8s/streaming-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: streaming-service
  namespace: video-platform
spec:
  replicas: 5
  selector:
    matchLabels:
      app: streaming-service
  template:
    metadata:
      labels:
        app: streaming-service
      annotations:
        prometheus.io/scrape: "true"
    spec:
      containers:
        - name: streaming-service
          image: your-registry/streaming-service:1.0.0
          ports:
            - containerPort: 3000  # HTTP
            - containerPort: 3001  # WebSocket
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
---
# Service สำหรับ WebSocket ต้องใช้ Session Affinity
apiVersion: v1
kind: Service
metadata:
  name: streaming-service
  namespace: video-platform
spec:
  selector:
    app: streaming-service
  ports:
    - name: http
      port: 80
      targetPort: 3000
    - name: websocket
      port: 3001
      targetPort: 3001
  sessionAffinity: ClientIP  # สำคัญมากสำหรับ WebSocket
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 ชั่วโมง
```

---

## 11. Content Delivery Optimization

```typescript
// cdn-optimization/src/adaptive-bitrate.ts

/**
 * Adaptive Bitrate Streaming (ABR) Client-side Logic
 * ใช้ใน Browser/Mobile App
 */

interface BitrateLevel {
  quality: string;
  bandwidth: number;
  playlistUrl: string;
}

export class AdaptiveBitratePlayer {
  private currentQuality: string = 'auto';
  private bitrateHistory: number[] = [];
  private readonly HISTORY_SIZE = 5;
  private readonly BUFFER_THRESHOLD_HIGH = 15; // วินาที
  private readonly BUFFER_THRESHOLD_LOW = 5;   // วินาที

  constructor(
    private readonly levels: BitrateLevel[],
    private readonly videoElement: HTMLVideoElement,
  ) {}

  selectQuality(currentBandwidth: number, bufferLength: number): string {
    if (this.currentQuality !== 'auto') {
      return this.currentQuality;
    }

    // ถ้า Buffer ต่ำ ลด Quality ลง
    if (bufferLength < this.BUFFER_THRESHOLD_LOW) {
      return this.getNextLowerQuality(this.getCurrentLevel());
    }

    // ถ้า Buffer สูง และ Bandwidth เพียงพอ เพิ่ม Quality
    if (bufferLength > this.BUFFER_THRESHOLD_HIGH) {
      const appropriateLevel = this.getBestQualityForBandwidth(currentBandwidth * 0.8);
      return appropriateLevel?.quality || this.levels[0].quality;
    }

    // รักษา Quality ปัจจุบัน
    return this.getCurrentLevel()?.quality || this.levels[0].quality;
  }

  updateBandwidthHistory(bandwidth: number): void {
    this.bitrateHistory.push(bandwidth);
    if (this.bitrateHistory.length > this.HISTORY_SIZE) {
      this.bitrateHistory.shift();
    }
  }

  getEstimatedBandwidth(): number {
    if (this.bitrateHistory.length === 0) return 0;
    
    // ใช้ Harmonic Mean สำหรับการประมาณที่แม่นยำกว่า
    const sum = this.bitrateHistory.reduce((acc, bw) => acc + 1 / bw, 0);
    return this.bitrateHistory.length / sum;
  }

  private getBestQualityForBandwidth(bandwidth: number): BitrateLevel | undefined {
    return this.levels
      .filter(l => l.bandwidth <= bandwidth)
      .sort((a, b) => b.bandwidth - a.bandwidth)[0];
  }

  private getCurrentLevel(): BitrateLevel | undefined {
    return this.levels.find(l => l.quality === this.currentQuality);
  }

  private getNextLowerQuality(currentLevel?: BitrateLevel): string {
    if (!currentLevel) return this.levels[this.levels.length - 1].quality;
    
    const currentIndex = this.levels.indexOf(currentLevel);
    const nextLevel = this.levels[currentIndex + 1];
    
    return nextLevel?.quality || currentLevel.quality;
  }
}
```

---

## สรุปบทที่ 95

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **Upload Pipeline** | Multipart Upload to S3 + Transcoding Queue |
| **Transcoding** | FFmpeg + HLS Segments (240p/360p/480p/720p/1080p) |
| **CDN** | CloudFront Signed URLs + Cache Invalidation |
| **Recommendation** | Watch History + Tag Scoring + Trending |
| **Search** | Elasticsearch + Multi-field + Thai Analyzer |
| **Live Streaming** | WebRTC + Socket.io + RTMP |
| **Analytics** | Real-time Counters (Redis) + Batch Persist (PostgreSQL) |
| **Watch History** | Redis Session + PostgreSQL History |
| **ABR Streaming** | Adaptive Bitrate based on Bandwidth + Buffer |
| **Kubernetes** | HPA based on Queue Length + Session Affinity |
