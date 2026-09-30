# Part 95: Case Study: Streaming Platform

## บทนำ

Streaming Platform คือระบบที่ต้องรองรับการรับชม Video ของผู้ใช้นับล้านคนพร้อมกัน บทนี้จะออกแบบระบบที่คล้ายกับ Netflix, YouTube หรือ AIS PLAY โดยครอบคลุม Video Encoding Pipeline, CDN Strategy, Recommendation System และ Analytics Pipeline

---

## 1. System Requirements

### 1.1 Functional Requirements

```
Core Features:
✅ Upload Video (Creator/Admin)
✅ Video Transcoding (Multiple Resolutions)
✅ Stream Video (HLS/DASH Adaptive Bitrate)
✅ Search Videos
✅ Recommendation System
✅ User Profiles & History
✅ Subscription Management
✅ Content Access Control (DRM)
✅ Comments & Reactions
✅ Analytics (Creator Dashboard)
✅ Live Streaming
✅ Offline Download
```

### 1.2 Scale Requirements

```
ระดับ Netflix Thailand / AIS PLAY:
┌───────────────────────────────────────────────────────────────┐
│ Metric                    │ Value                             │
├───────────────────────────┼───────────────────────────────────┤
│ DAU                       │ 5M                                │
│ Concurrent Streams        │ 1M                                │
│ Videos in Library         │ 100,000                           │
│ Video Upload/day          │ 1,000 videos                      │
│ Storage                   │ 10 PB (petabytes)                 │
│ CDN Bandwidth             │ 40 Tbps peak                      │
│ Avg video bitrate         │ 4 Mbps (1080p)                    │
│ API Latency               │ < 100ms (p99)                     │
│ Playback Start Time       │ < 2 seconds                       │
│ Availability              │ 99.99%                            │
└───────────────────────────┴───────────────────────────────────┘

Storage Calculation:
1,000 videos/day × 2 hours avg = 2,000 hours/day
2,000 hours × 3600 sec × 4 Mbps = ~3.6 TB raw/day
After transcoding (5 qualities × 2x): ~36 TB/day
After 5 years: ~65 PB
```

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Streaming Platform Architecture                   │
│                                                                        │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                     Ingestion Layer                            │   │
│  │  ┌─────────────────┐    ┌──────────────────────────────────┐  │   │
│  │  │  Upload Service │    │    Transcoding Pipeline           │  │   │
│  │  │  (Multipart)    │──> │  (FFmpeg Workers on Kubernetes)   │  │   │
│  │  └─────────────────┘    └──────────────────────────────────┘  │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                  │                                     │
│                          ┌───────▼──────────┐                         │
│                          │    S3 Storage     │                         │
│                          │  (Raw + Encoded)  │                         │
│                          └───────────────────┘                         │
│                                  │                                     │
│  ┌───────────────────────────────▼──────────────────────────────┐    │
│  │                    Delivery Layer                             │    │
│  │                                                               │    │
│  │  ┌────────────────────────────────────────────────────────┐  │    │
│  │  │              CloudFront CDN                             │  │    │
│  │  │  Edge Locations: Bangkok, Singapore, HK, Tokyo          │  │    │
│  │  │  - Adaptive Bitrate (HLS/DASH)                         │  │    │
│  │  │  - DRM (Widevine/FairPlay/PlayReady)                   │  │    │
│  │  │  - Token-based Authentication                          │  │    │
│  │  └────────────────────────────────────────────────────────┘  │    │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                        │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                    Application Layer                           │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │   │
│  │  │Video Metadata│  │User Service  │  │Recommendation Svc│   │   │
│  │  │Service       │  │              │  │(ML-based)        │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘   │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │   │
│  │  │Search Service│  │Analytics Svc │  │Subscription Svc  │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘   │   │
│  └───────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. Video Encoding Pipeline

### 3.1 Upload Flow

```
Video Upload Flow:
┌────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Creator ──> Upload Service ──> S3 (raw-uploads bucket)        │
│                  │                                               │
│              Generate presigned URL (avoid proxy overhead)      │
│                  │                                               │
│  Creator ──> Direct S3 Upload (multipart for large files)      │
│                  │                                               │
│              S3 Event → SQS → Transcoding Job Queue            │
│                  │                                               │
│          Transcoding Workers pick up job                        │
│                  │                                               │
│    ┌─────────────▼──────────────┐                               │
│    │      FFmpeg Processing     │                               │
│    │                            │                               │
│    │  Input: raw_video.mp4      │                               │
│    │  Output:                   │                               │
│    │   ├── 4K_2160p.m3u8        │                               │
│    │   ├── FHD_1080p.m3u8       │                               │
│    │   ├── HD_720p.m3u8         │                               │
│    │   ├── SD_480p.m3u8         │                               │
│    │   ├── Low_360p.m3u8        │                               │
│    │   └── master.m3u8          │ (HLS manifest)               │
│    └────────────────────────────┘                               │
│                  │                                               │
│          Upload to S3 (encoded-videos bucket)                   │
│                  │                                               │
│          Update Video metadata DB                               │
│                  │                                               │
│          Notify Creator: "Video ready!"                         │
└────────────────────────────────────────────────────────────────┘
```

### 3.2 Transcoding Service Implementation

```go
// transcoding-service/internal/worker/transcoder.go
package worker

import (
    "context"
    "fmt"
    "os/exec"
    "path/filepath"
)

type TranscodingProfile struct {
    Name       string
    Resolution string
    Bitrate    string
    AudioBit   string
    Codec      string
}

var profiles = []TranscodingProfile{
    {Name: "4k",    Resolution: "3840:2160", Bitrate: "8000k", AudioBit: "320k", Codec: "h264"},
    {Name: "1080p", Resolution: "1920:1080", Bitrate: "4000k", AudioBit: "192k", Codec: "h264"},
    {Name: "720p",  Resolution: "1280:720",  Bitrate: "2000k", AudioBit: "128k", Codec: "h264"},
    {Name: "480p",  Resolution: "854:480",   Bitrate: "1000k", AudioBit: "128k", Codec: "h264"},
    {Name: "360p",  Resolution: "640:360",   Bitrate: "500k",  AudioBit: "96k",  Codec: "h264"},
}

type TranscodingWorker struct {
    s3       S3Client
    db       VideoRepository
    queue    JobQueue
    notifier NotificationService
}

func (w *TranscodingWorker) ProcessJob(ctx context.Context, job TranscodingJob) error {
    // Download raw video from S3
    localPath := fmt.Sprintf("/tmp/raw_%s.mp4", job.VideoID)
    if err := w.s3.Download(ctx, job.S3Key, localPath); err != nil {
        return fmt.Errorf("download failed: %w", err)
    }
    defer os.Remove(localPath)

    outputDir := fmt.Sprintf("/tmp/output_%s", job.VideoID)
    os.MkdirAll(outputDir, 0755)
    defer os.RemoveAll(outputDir)

    // Transcode each profile in parallel (use goroutines per profile)
    type transcodeResult struct {
        profile string
        err     error
        m3u8    string
    }
    
    resultChan := make(chan transcodeResult, len(profiles))
    
    for _, profile := range profiles {
        go func(p TranscodingProfile) {
            outputPath := filepath.Join(outputDir, p.Name)
            os.MkdirAll(outputPath, 0755)
            
            // FFmpeg command for HLS output
            args := []string{
                "-i", localPath,
                "-vf", fmt.Sprintf("scale=%s", p.Resolution),
                "-c:v", p.Codec,
                "-b:v", p.Bitrate,
                "-c:a", "aac",
                "-b:a", p.AudioBit,
                "-hls_time", "6",          // 6-second segments
                "-hls_list_size", "0",     // Include all segments
                "-hls_segment_filename", filepath.Join(outputPath, "segment_%03d.ts"),
                "-hls_flags", "independent_segments",
                filepath.Join(outputPath, "index.m3u8"),
            }
            
            cmd := exec.CommandContext(ctx, "ffmpeg", args...)
            output, err := cmd.CombinedOutput()
            if err != nil {
                resultChan <- transcodeResult{
                    profile: p.Name,
                    err:     fmt.Errorf("ffmpeg error: %w, output: %s", err, output),
                }
                return
            }
            
            resultChan <- transcodeResult{
                profile: p.Name,
                m3u8:    filepath.Join(outputPath, "index.m3u8"),
            }
        }(profile)
    }
    
    // Collect results
    var m3u8Files []string
    for i := 0; i < len(profiles); i++ {
        result := <-resultChan
        if result.err != nil {
            return result.err
        }
        m3u8Files = append(m3u8Files, result.m3u8)
    }
    
    // Generate master playlist
    masterM3U8 := generateMasterPlaylist(profiles)
    masterPath := filepath.Join(outputDir, "master.m3u8")
    os.WriteFile(masterPath, []byte(masterM3U8), 0644)
    
    // Generate thumbnail
    thumbnail, err := w.generateThumbnail(ctx, localPath, outputDir)
    if err != nil {
        log.Warn("thumbnail generation failed", "error", err)
    }
    
    // Upload all segments to S3
    baseKey := fmt.Sprintf("videos/%s", job.VideoID)
    if err := w.s3.UploadDirectory(ctx, outputDir, baseKey); err != nil {
        return fmt.Errorf("upload failed: %w", err)
    }
    
    // Update video status
    w.db.UpdateVideo(ctx, job.VideoID, VideoUpdate{
        Status:       VideoStatusReady,
        MasterM3U8:   fmt.Sprintf("%s/master.m3u8", baseKey),
        ThumbnailKey: thumbnail,
        Resolutions:  extractResolutions(profiles),
        Duration:     w.getDuration(localPath),
    })
    
    // Notify creator
    w.notifier.SendVideoReady(ctx, job.CreatorID, job.VideoID)
    
    return nil
}

func generateMasterPlaylist(profiles []TranscodingProfile) string {
    playlist := "#EXTM3U\n#EXT-X-VERSION:3\n\n"
    
    bitrateMap := map[string]int{
        "360p":  500000,
        "480p":  1000000,
        "720p":  2000000,
        "1080p": 4000000,
        "4k":    8000000,
    }
    
    resolutionMap := map[string]string{
        "360p":  "640x360",
        "480p":  "854x480",
        "720p":  "1280x720",
        "1080p": "1920x1080",
        "4k":    "3840x2160",
    }
    
    for _, p := range profiles {
        playlist += fmt.Sprintf(
            "#EXT-X-STREAM-INF:BANDWIDTH=%d,RESOLUTION=%s\n%s/index.m3u8\n\n",
            bitrateMap[p.Name],
            resolutionMap[p.Name],
            p.Name,
        )
    }
    
    return playlist
}
```

### 3.3 DRM Integration

```go
// drm-service/internal/service/drm.go
package service

import (
    "context"
    "time"
    "crypto/rand"
)

// DRM: Digital Rights Management
// Support: Widevine (Android/Chrome), FairPlay (iOS/Safari), PlayReady (Windows)

type DRMService struct {
    widevineLicenseURL string
    fairplayLicenseURL string
    keyDB              KeyRepository
}

type ContentKey struct {
    KeyID  string
    Key    []byte
    VideoID string
    ExpiresAt time.Time
}

func (s *DRMService) GenerateContentKey(ctx context.Context, videoID string) (*ContentKey, error) {
    keyID := generateUUID()
    key := make([]byte, 16) // 128-bit AES key
    if _, err := rand.Read(key); err != nil {
        return nil, err
    }
    
    contentKey := &ContentKey{
        KeyID:    keyID,
        Key:      key,
        VideoID:  videoID,
        ExpiresAt: time.Now().Add(365 * 24 * time.Hour),
    }
    
    // Store encrypted key in HSM-backed storage
    if err := s.keyDB.Store(ctx, contentKey); err != nil {
        return nil, err
    }
    
    return contentKey, nil
}

// Generate signed playback token
func (s *DRMService) GeneratePlaybackToken(ctx context.Context, req PlaybackTokenRequest) (string, error) {
    // Check subscription
    if err := s.checkAccess(ctx, req.UserID, req.ContentID); err != nil {
        return "", ErrAccessDenied
    }
    
    claims := PlaybackTokenClaims{
        UserID:    req.UserID,
        ContentID: req.ContentID,
        ExpiresAt: time.Now().Add(4 * time.Hour).Unix(), // Token valid for 4 hours
        MaxResolution: getMaxResolution(req.SubscriptionTier),
        AllowDownload: req.SubscriptionTier == "premium",
        SessionID: generateSessionID(),
    }
    
    return s.signToken(claims)
}

// CDN Token Auth (prevent URL sharing)
func (s *DRMService) GenerateCDNToken(ctx context.Context, videoID, userID string) string {
    // CloudFront Signed URL
    expiry := time.Now().Add(4 * time.Hour)
    
    policy := fmt.Sprintf(`{
        "Statement": [{
            "Resource": "https://cdn.example.com/videos/%s/*",
            "Condition": {
                "DateLessThan": {"AWS:EpochTime": %d},
                "IpAddress": {"AWS:SourceIp": "0.0.0.0/0"}
            }
        }]
    }`, videoID, expiry.Unix())
    
    return signCloudFrontPolicy(policy, s.privateKey)
}
```

---

## 4. CDN Strategy

### 4.1 CDN Architecture

```
CDN Strategy สำหรับ Thai Streaming Platform:

Geographic Distribution:
┌────────────────────────────────────────────────────────────────┐
│                                                                  │
│  User in Bangkok ──> Edge: Bangkok PoP (AWS ap-southeast-1)    │
│  User in Chiang Mai ──> Edge: Bangkok PoP                      │
│  User in Phuket ──> Edge: Bangkok PoP                          │
│  User in Singapore ──> Edge: Singapore PoP                     │
│  User in Japan ──> Edge: Tokyo PoP                             │
│                                                                  │
│  Cache Hit Rate Target: 95%+ (popular content)                 │
│                                                                  │
│  Origin Shield: Singapore (reduces origin load)                │
│                                                                  │
│    User ──> Edge ──> Regional Cache ──> Origin Shield ──> S3  │
└────────────────────────────────────────────────────────────────┘

CDN Caching Strategy:
Content Type          TTL        Behavior
─────────────────     ─────      ──────────
.ts segments          7 days     Cache aggressive
.m3u8 (master)        10 min     Medium cache (for quality changes)
.m3u8 (media)         5 sec      Short cache (live/DVR)
Thumbnails            7 days     Cache aggressive
Subtitles             24 hours   Cache medium
```

### 4.2 Adaptive Bitrate Streaming

```javascript
// Frontend: Video.js with HLS.js implementation
// client/src/components/VideoPlayer.jsx

import React, { useEffect, useRef } from 'react';
import Hls from 'hls.js';

const VideoPlayer = ({ videoId, userTier }) => {
    const videoRef = useRef(null);
    const hlsRef = useRef(null);
    
    useEffect(() => {
        const initPlayer = async () => {
            // Get signed playback URL
            const response = await fetch(`/api/v1/videos/${videoId}/playback-token`, {
                headers: { Authorization: `Bearer ${getToken()}` }
            });
            const { playbackUrl, cdnToken } = await response.json();
            
            if (Hls.isSupported()) {
                const hls = new Hls({
                    // Adaptive Bitrate Config
                    startLevel: -1,           // Auto select starting level
                    abrEwmaDefaultEstimate: 5000000, // Start assuming 5 Mbps
                    abrEwmaFastLive: 3.0,
                    abrEwmaSlowLive: 9.0,
                    
                    // Buffer settings
                    maxBufferLength: 60,      // Buffer up to 60 seconds
                    maxMaxBufferLength: 600,
                    maxBufferSize: 60 * 1000 * 1000, // 60 MB
                    
                    // Recovery settings
                    enableWorker: true,
                    lowLatencyMode: false,
                    
                    // XHR setup (for CDN tokens)
                    xhrSetup: (xhr, url) => {
                        xhr.setRequestHeader('CloudFront-Signature', cdnToken);
                    },
                });
                
                hls.loadSource(playbackUrl);
                hls.attachMedia(videoRef.current);
                
                // Analytics: Track quality switches
                hls.on(Hls.Events.LEVEL_SWITCHED, (event, data) => {
                    const level = hls.levels[data.level];
                    trackQualityChange({
                        videoId,
                        resolution: `${level.height}p`,
                        bitrate: level.bitrate,
                        timestamp: Date.now(),
                    });
                });
                
                // Stall detection
                hls.on(Hls.Events.ERROR, (event, data) => {
                    if (data.fatal) {
                        switch (data.type) {
                            case Hls.ErrorTypes.NETWORK_ERROR:
                                hls.startLoad(); // Retry
                                break;
                            case Hls.ErrorTypes.MEDIA_ERROR:
                                hls.recoverMediaError();
                                break;
                        }
                    }
                });
                
                hlsRef.current = hls;
            } else if (videoRef.current.canPlayType('application/vnd.apple.mpegurl')) {
                // Safari native HLS
                videoRef.current.src = playbackUrl;
            }
        };
        
        initPlayer();
        return () => hlsRef.current?.destroy();
    }, [videoId]);
    
    return (
        <div className="video-container">
            <video
                ref={videoRef}
                controls
                autoPlay
                playsInline
                style={{ width: '100%' }}
            />
        </div>
    );
};
```

---

## 5. Recommendation System

### 5.1 Recommendation Architecture

```
Recommendation System สำหรับ Streaming Platform:

Types of Recommendations:
1. Collaborative Filtering: "คนที่ดูเหมือนกับคุณชอบ..."
2. Content-Based: "เพราะคุณดู X ที่มีแนวเดียวกัน..."
3. Trending: "กำลังนิยมในไทยตอนนี้"
4. Personalized (Hybrid): ผสม 1+2+3

Data Sources:
- Watch history (ดูนาน = ชอบมาก)
- Explicit ratings
- Search queries
- Time of day patterns
- Genre preferences
- Language preferences
- Skip behavior (skip intro = อยากดูหนังเลย)
```

### 5.2 Recommendation Model

```python
# recommendation-service/app/models/collaborative_filter.py
import numpy as np
from scipy.sparse import csr_matrix
from sklearn.decomposition import TruncatedSVD
from typing import List, Tuple
import redis
import json

class CollaborativeFilteringRecommender:
    """
    Matrix Factorization using SVD (Singular Value Decomposition)
    User-Item Matrix → Latent Factors
    """
    
    def __init__(self, n_factors: int = 50):
        self.n_factors = n_factors
        self.svd = TruncatedSVD(n_components=n_factors)
        self.user_factors = None
        self.item_factors = None
        self.user_index = {}  # userID → matrix row
        self.item_index = {}  # videoID → matrix col
    
    def fit(self, interactions_df):
        """
        interactions_df: DataFrame with columns [user_id, video_id, rating]
        rating = watch_percentage * 5 (0-5 scale)
        """
        # Create user/item indices
        users = interactions_df['user_id'].unique()
        items = interactions_df['video_id'].unique()
        
        self.user_index = {u: i for i, u in enumerate(users)}
        self.item_index = {v: i for i, v in enumerate(items)}
        self.reverse_item_index = {i: v for v, i in self.item_index.items()}
        
        # Build sparse matrix
        rows = interactions_df['user_id'].map(self.user_index)
        cols = interactions_df['video_id'].map(self.item_index)
        data = interactions_df['rating']
        
        user_item_matrix = csr_matrix(
            (data, (rows, cols)),
            shape=(len(users), len(items))
        )
        
        # Fit SVD
        self.user_factors = self.svd.fit_transform(user_item_matrix)
        self.item_factors = self.svd.components_.T
        
        print(f"Model trained: {len(users)} users, {len(items)} items, {n_factors} factors")
    
    def recommend(self, user_id: str, n: int = 20, exclude_watched: List[str] = None) -> List[Tuple[str, float]]:
        """Get top N recommendations for a user"""
        if user_id not in self.user_index:
            return self._get_popular_items(n)
        
        user_idx = self.user_index[user_id]
        user_vector = self.user_factors[user_idx]
        
        # Dot product: user vector × all item vectors
        scores = np.dot(user_vector, self.item_factors.T)
        
        # Create (video_id, score) pairs
        video_scores = [
            (self.reverse_item_index[i], float(score))
            for i, score in enumerate(scores)
        ]
        
        # Filter out watched content
        if exclude_watched:
            watched_set = set(exclude_watched)
            video_scores = [(vid, score) for vid, score in video_scores 
                          if vid not in watched_set]
        
        # Sort by score
        video_scores.sort(key=lambda x: x[1], reverse=True)
        
        return video_scores[:n]
    
    def _get_popular_items(self, n: int) -> List[Tuple[str, float]]:
        """Fallback for new users (cold start)"""
        # Return trending/popular videos from Redis
        return []


class HybridRecommender:
    """Combines multiple recommendation strategies"""
    
    def __init__(self, redis_client: redis.Redis):
        self.collab_filter = CollaborativeFilteringRecommender()
        self.content_filter = ContentBasedRecommender()
        self.trending = TrendingService(redis_client)
        self.redis = redis_client
    
    def get_recommendations(self, user_id: str, context: RecommendationContext) -> List[Video]:
        cache_key = f"recs:{user_id}:{context.page}"
        
        # Check cache (5 minute TTL)
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)
        
        # Get watch history
        watch_history = self.get_watch_history(user_id)
        
        # Determine strategy based on user maturity
        if len(watch_history) < 5:
            # Cold start: use trending + content popular
            recommendations = self.trending.get_top(context.genre, 20)
        else:
            # Hybrid: 60% collaborative + 30% content-based + 10% trending
            collab_recs = self.collab_filter.recommend(user_id, 12, watch_history)
            content_recs = self.content_filter.recommend(watch_history[-5:], 6)
            trending_recs = self.trending.get_top(context.genre, 2)
            
            # Merge and deduplicate
            recommendations = self.merge_recommendations(
                collab_recs, content_recs, trending_recs
            )
        
        # Re-rank based on context (time of day, device type)
        final_recs = self.contextual_rerank(recommendations, context)
        
        # Cache results
        self.redis.setex(cache_key, 300, json.dumps(final_recs))
        
        return final_recs
    
    def contextual_rerank(self, recs, context: RecommendationContext):
        """Re-rank based on contextual signals"""
        scored = []
        for rec in recs:
            score = rec['base_score']
            
            # Boost based on time of day
            if context.hour >= 21 and rec['genre'] in ['drama', 'romance']:
                score *= 1.2  # Evening: boost drama/romance
            elif context.hour <= 8 and rec['genre'] in ['news', 'documentary']:
                score *= 1.3  # Morning: boost news
            
            # Device context
            if context.device == 'mobile' and rec['duration_minutes'] < 30:
                score *= 1.1  # Mobile: prefer shorter content
            
            # Language preference
            if context.preferred_language == rec['language']:
                score *= 1.5  # Strong boost for preferred language
            
            scored.append({**rec, 'final_score': score})
        
        return sorted(scored, key=lambda x: x['final_score'], reverse=True)
```

---

## 6. Analytics Pipeline

### 6.1 Real-time Analytics

```python
# analytics-service/app/streaming/viewer_analytics.py
from kafka import KafkaConsumer
from elasticsearch import AsyncElasticsearch
import asyncio
import json
from collections import defaultdict
from datetime import datetime

class ViewerAnalyticsPipeline:
    """Real-time processing of viewer events"""
    
    def __init__(self, kafka_servers: list, es: AsyncElasticsearch):
        self.kafka_servers = kafka_servers
        self.es = es
        # Sliding window aggregation (5-minute windows)
        self.window = defaultdict(lambda: {
            'total_views': 0,
            'unique_viewers': set(),
            'total_watch_seconds': 0,
            'buffering_events': 0,
        })
    
    async def process_events(self):
        consumer = KafkaConsumer(
            'video.view.events',
            bootstrap_servers=self.kafka_servers,
            value_deserializer=lambda x: json.loads(x.decode()),
            group_id='analytics-processor',
            auto_offset_reset='latest',
        )
        
        async for msg in consumer:
            event = msg.value
            await self.process_event(event)
    
    async def process_event(self, event: dict):
        event_type = event.get('type')
        video_id = event.get('video_id')
        user_id = event.get('user_id')
        timestamp = event.get('timestamp')
        
        window_key = f"{video_id}:{datetime.fromtimestamp(timestamp).strftime('%Y%m%d%H%M')[:11]}0"
        
        if event_type == 'VIEW_START':
            self.window[window_key]['total_views'] += 1
            self.window[window_key]['unique_viewers'].add(user_id)
            
        elif event_type == 'VIEW_PROGRESS':
            watch_seconds = event.get('watch_seconds', 0)
            self.window[window_key]['total_watch_seconds'] += watch_seconds
            
        elif event_type == 'BUFFERING':
            self.window[window_key]['buffering_events'] += 1
            
        elif event_type == 'QUALITY_CHANGE':
            await self.track_quality_change(event)
        
        # Flush to Elasticsearch every 100 events
        if sum(v['total_views'] for v in self.window.values()) % 100 == 0:
            await self.flush_to_es()
    
    async def flush_to_es(self):
        """Write aggregated metrics to Elasticsearch"""
        bulk_data = []
        
        for window_key, metrics in self.window.items():
            video_id, window_time = window_key.split(':', 1)
            
            doc = {
                'video_id': video_id,
                'window_time': window_time,
                'total_views': metrics['total_views'],
                'unique_viewers': len(metrics['unique_viewers']),
                'total_watch_seconds': metrics['total_watch_seconds'],
                'buffering_events': metrics['buffering_events'],
                'buffering_rate': metrics['buffering_events'] / max(metrics['total_views'], 1),
            }
            
            bulk_data.extend([
                {'index': {'_index': 'video-analytics', '_id': window_key}},
                doc
            ])
        
        if bulk_data:
            await self.es.bulk(body=bulk_data)
        
        self.window.clear()


class CreatorDashboardService:
    """Creator analytics dashboard"""
    
    def __init__(self, es: AsyncElasticsearch, clickhouse_client):
        self.es = es
        self.clickhouse = clickhouse_client
    
    async def get_video_analytics(self, creator_id: str, video_id: str, period: str = '7d') -> dict:
        # Query Elasticsearch for real-time data
        response = await self.es.search(
            index='video-analytics',
            body={
                'query': {
                    'bool': {
                        'must': [
                            {'term': {'video_id': video_id}},
                            {'range': {'window_time': {'gte': f'now-{period}', 'lte': 'now'}}}
                        ]
                    }
                },
                'aggs': {
                    'hourly': {
                        'date_histogram': {
                            'field': 'window_time',
                            'calendar_interval': 'hour'
                        },
                        'aggs': {
                            'total_views': {'sum': {'field': 'total_views'}},
                            'unique_viewers': {'sum': {'field': 'unique_viewers'}},
                            'watch_time': {'sum': {'field': 'total_watch_seconds'}},
                            'buffering_rate': {'avg': {'field': 'buffering_rate'}},
                        }
                    }
                }
            }
        )
        
        # Query ClickHouse for audience retention curve
        retention = await self.clickhouse.query(f"""
            SELECT 
                position_seconds,
                count() as viewer_count,
                count() / max(count()) OVER () as retention_rate
            FROM video_watch_events
            WHERE video_id = '{video_id}'
              AND timestamp >= now() - INTERVAL {period}
            GROUP BY position_seconds
            ORDER BY position_seconds
        """)
        
        return {
            'total_views': sum(b['total_views']['value'] for b in response['aggregations']['hourly']['buckets']),
            'unique_viewers': sum(b['unique_viewers']['value'] for b in response['aggregations']['hourly']['buckets']),
            'total_watch_hours': sum(b['watch_time']['value'] for b in response['aggregations']['hourly']['buckets']) / 3600,
            'avg_buffering_rate': sum(b['buffering_rate']['value'] for b in response['aggregations']['hourly']['buckets']) / len(response['aggregations']['hourly']['buckets']),
            'hourly_breakdown': response['aggregations']['hourly']['buckets'],
            'audience_retention': retention,
        }
```

---

## 7. Live Streaming

### 7.1 Live Streaming Architecture

```
Live Streaming Flow:
┌────────────────────────────────────────────────────────────────┐
│                                                                  │
│  OBS/vMix ──RTMP──> Media Server (nginx-rtmp) ──> HLS Segments │
│                          │                                       │
│                    Transcode realtime                           │
│                    (4 quality levels)                            │
│                          │                                       │
│                    Push to CDN Origin ──> CDN Edge ──> Viewers  │
│                          │                                       │
│                    Low Latency: < 10 seconds                    │
│                    Standard: < 30 seconds                       │
│                                                                  │
│  Chat + Reactions:                                               │
│  Viewers ──> WebSocket ──> Chat Service ──> Redis Pub/Sub       │
│                                     └──────────────> All viewers│
└────────────────────────────────────────────────────────────────┘
```

### 7.2 Nginx RTMP Configuration

```nginx
# nginx-rtmp.conf
worker_processes auto;
events {
    worker_connections 1024;
}

rtmp {
    server {
        listen 1935;
        chunk_size 4096;
        
        application live {
            live on;
            record off;
            
            # Auth via HTTP callback
            on_publish http://auth-service/rtmp/authenticate;
            on_publish_done http://auth-service/rtmp/end;
            
            # Transcoding: Push to multiple HLS outputs
            exec_push /usr/local/bin/ffmpeg 
                -i rtmp://localhost/live/$name
                # 1080p
                -c:v libx264 -b:v 4000k -vf scale=1920:1080 
                -c:a aac -b:a 192k
                -hls_time 2 -hls_list_size 5
                /tmp/hls/$name/1080p/index.m3u8
                # 720p
                -c:v libx264 -b:v 2000k -vf scale=1280:720
                -c:a aac -b:a 128k
                -hls_time 2 -hls_list_size 5
                /tmp/hls/$name/720p/index.m3u8
                # 480p
                -c:v libx264 -b:v 800k -vf scale=854:480
                -c:a aac -b:a 96k
                -hls_time 2 -hls_list_size 5
                /tmp/hls/$name/480p/index.m3u8;
        }
    }
}

http {
    server {
        listen 8080;
        
        location /hls/ {
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /tmp;
            add_header Cache-Control no-cache;
            add_header Access-Control-Allow-Origin *;
            
            # Serve .m3u8 with short TTL, .ts with longer TTL
            if ($request_filename ~* \.m3u8$) {
                add_header Cache-Control "no-cache, no-store";
            }
        }
    }
}
```

---

## สรุป

การออกแบบ Streaming Platform ต้องเน้นที่:

1. **Video Pipeline** - Upload → Transcode → Store → Deliver via CDN
2. **Adaptive Bitrate** - HLS/DASH ปรับคุณภาพตาม Network Speed
3. **CDN Strategy** - Cache aggressive, Global edge locations
4. **DRM** - Protect content จาก Piracy (Widevine/FairPlay)
5. **Recommendation** - Hybrid ML model: Collaborative + Content-Based
6. **Analytics** - Real-time Kafka pipeline → Creator Dashboard
7. **Live Streaming** - RTMP ingest → HLS delivery with low latency

> "The best streaming experience is one the user doesn't notice — smooth playback, instant start, and content they actually want to watch"

---

*ถัดไป: Part 96 - Microservices Roadmap 2024-2025*
