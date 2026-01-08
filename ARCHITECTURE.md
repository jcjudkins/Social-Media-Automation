# Social Media Automation Platform - Architecture Design

**Version:** 1.0
**Last Updated:** 2026-01-08
**Author:** Jason Judkins

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Database Architecture](#2-database-architecture)
3. [API Integration Layer](#3-api-integration-layer)
4. [Task Scheduling System](#4-task-scheduling-system)
5. [REST API Design](#5-rest-api-design)
6. [Media Management](#6-media-management)
7. [Authentication & Security](#7-authentication--security)
8. [Deployment Architecture](#8-deployment-architecture)
9. [Testing Strategy](#9-testing-strategy)
10. [Performance Considerations](#10-performance-considerations)
11. [Implementation Roadmap](#11-implementation-roadmap)

---

## 1. System Overview

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend Layer                            │
│  (React/Vue Dashboard - Future Phase)                           │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Django REST API Layer                          │
│  - Authentication & Authorization                                │
│  - Business Logic                                                │
│  - Validation & Serialization                                    │
└───────────┬───────────────────────────────┬─────────────────────┘
            │                               │
            ▼                               ▼
┌───────────────────────┐      ┌──────────────────────────────────┐
│   PostgreSQL DB       │      │    Celery + Redis                │
│   - User Data         │      │    - Task Queue                  │
│   - Posts             │      │    - Scheduled Jobs              │
│   - Accounts          │      │    - Rate Limiting               │
│   - Analytics         │      │    - Retry Logic                 │
└───────────────────────┘      └──────────────┬───────────────────┘
                                              │
                                              ▼
                               ┌──────────────────────────────────┐
                               │  Social Media Integrations       │
                               │  - Twitter API v2                │
                               │  - Bluesky ATP                   │
                               │  - LinkedIn API                  │
                               │  - Mastodon API                  │
                               │  - Instagram Graph API           │
                               └──────────────────────────────────┘
```

### 1.2 Core Components

1. **Django Application**: Core business logic and API
2. **Celery Workers**: Asynchronous task processing
3. **Redis**: Message broker and caching
4. **PostgreSQL**: Primary data store
5. **S3 (AWS)**: Media file storage
6. **Docker**: Containerization
7. **GitHub Actions**: CI/CD pipeline

### 1.3 Design Principles

- **Separation of Concerns**: Each platform integration is isolated
- **Fail-Safe**: Graceful degradation when platforms are unavailable
- **Scalability**: Horizontal scaling via containerization
- **Testability**: Comprehensive test coverage with mocked integrations
- **Security First**: OAuth2, encrypted credentials, API key management

---

## 2. Database Architecture

### 2.1 Core Models

#### User & Authentication
```python
# Extended Django User model
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    timezone = models.CharField(max_length=50, default='UTC')
    preferred_posting_times = models.JSONField(default=list)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

#### Social Media Accounts
```python
class SocialMediaPlatform(models.TextChoices):
    TWITTER = 'twitter', 'Twitter/X'
    BLUESKY = 'bluesky', 'Bluesky'
    LINKEDIN = 'linkedin', 'LinkedIn'
    MASTODON = 'mastodon', 'Mastodon'
    INSTAGRAM = 'instagram', 'Instagram'

class ConnectedAccount(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='connected_accounts')
    platform = models.CharField(max_length=20, choices=SocialMediaPlatform.choices)
    platform_user_id = models.CharField(max_length=255)
    platform_username = models.CharField(max_length=255)

    # Encrypted OAuth tokens
    access_token = models.TextField()  # Use django-cryptography
    refresh_token = models.TextField(null=True, blank=True)
    token_expires_at = models.DateTimeField(null=True, blank=True)

    # Platform-specific settings
    platform_settings = models.JSONField(default=dict)  # e.g., character limits, features

    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        unique_together = [['user', 'platform', 'platform_user_id']]
        indexes = [
            models.Index(fields=['user', 'platform']),
            models.Index(fields=['is_active']),
        ]
```

#### Posts & Scheduling
```python
class PostStatus(models.TextChoices):
    DRAFT = 'draft', 'Draft'
    SCHEDULED = 'scheduled', 'Scheduled'
    QUEUED = 'queued', 'Queued'
    POSTING = 'posting', 'Posting'
    POSTED = 'posted', 'Posted'
    FAILED = 'failed', 'Failed'
    CANCELLED = 'cancelled', 'Cancelled'

class Post(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')

    # Content
    content = models.TextField(max_length=5000)

    # Scheduling
    status = models.CharField(max_length=20, choices=PostStatus.choices, default=PostStatus.DRAFT)
    scheduled_time = models.DateTimeField(null=True, blank=True)
    posted_at = models.DateTimeField(null=True, blank=True)

    # Metadata
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Task tracking
    celery_task_id = models.CharField(max_length=255, null=True, blank=True)

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['user', 'status']),
            models.Index(fields=['scheduled_time']),
            models.Index(fields=['status', 'scheduled_time']),
        ]

class PostTarget(models.Model):
    """Many-to-many through table for Post -> ConnectedAccount"""
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='targets')
    account = models.ForeignKey(ConnectedAccount, on_delete=models.CASCADE)

    # Platform-specific content variations
    custom_content = models.TextField(null=True, blank=True)  # Override main content
    platform_specific_settings = models.JSONField(default=dict)  # e.g., hashtags, mentions

    # Status tracking per platform
    status = models.CharField(max_length=20, choices=PostStatus.choices, default=PostStatus.DRAFT)
    platform_post_id = models.CharField(max_length=255, null=True, blank=True)  # ID from platform
    platform_post_url = models.URLField(null=True, blank=True)

    error_message = models.TextField(null=True, blank=True)
    retry_count = models.IntegerField(default=0)
    last_attempt_at = models.DateTimeField(null=True, blank=True)

    posted_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        unique_together = [['post', 'account']]
        indexes = [
            models.Index(fields=['status']),
            models.Index(fields=['post', 'status']),
        ]
```

#### Media Files
```python
class MediaFile(models.Model):
    class MediaType(models.TextChoices):
        IMAGE = 'image', 'Image'
        VIDEO = 'video', 'Video'
        GIF = 'gif', 'GIF'

    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='media_files')
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='media_files')

    media_type = models.CharField(max_length=10, choices=MediaType.choices)
    original_filename = models.CharField(max_length=255)

    # Storage
    file = models.FileField(upload_to='media/%Y/%m/%d/')  # S3 in production
    file_size = models.BigIntegerField()  # bytes
    mime_type = models.CharField(max_length=100)

    # Dimensions (for images/videos)
    width = models.IntegerField(null=True, blank=True)
    height = models.IntegerField(null=True, blank=True)
    duration = models.FloatField(null=True, blank=True)  # seconds for video

    # Platform-specific media IDs (uploaded to each platform)
    platform_media_ids = models.JSONField(default=dict)  # {platform: media_id}

    alt_text = models.CharField(max_length=1000, blank=True)

    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']
```

#### Analytics
```python
class PostAnalytics(models.Model):
    """Aggregated analytics for posts across platforms"""
    post_target = models.OneToOneField(PostTarget, on_delete=models.CASCADE, related_name='analytics')

    # Engagement metrics
    likes = models.IntegerField(default=0)
    retweets = models.IntegerField(default=0)  # or equivalent (shares, reposts)
    replies = models.IntegerField(default=0)
    impressions = models.IntegerField(default=0)
    clicks = models.IntegerField(default=0)

    # Platform-specific metrics
    platform_metrics = models.JSONField(default=dict)

    last_updated = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name_plural = 'Post analytics'

class AnalyticsSnapshot(models.Model):
    """Time-series data for analytics tracking"""
    post_target = models.ForeignKey(PostTarget, on_delete=models.CASCADE, related_name='snapshots')

    snapshot_data = models.JSONField()  # Store all metrics at this point in time
    captured_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-captured_at']
        indexes = [
            models.Index(fields=['post_target', 'captured_at']),
        ]
```

#### Queue Management
```python
class PostQueue(models.Model):
    """Organize posts into queues for automated scheduling"""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='queues')
    name = models.CharField(max_length=100)
    description = models.TextField(blank=True)

    # Target accounts for this queue
    target_accounts = models.ManyToManyField(ConnectedAccount)

    # Scheduling rules
    posting_schedule = models.JSONField(default=dict)  # e.g., {"monday": ["09:00", "15:00"], ...}
    timezone = models.CharField(max_length=50, default='UTC')

    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

class QueuedPost(models.Model):
    """Posts in a queue waiting to be scheduled"""
    queue = models.ForeignKey(PostQueue, on_delete=models.CASCADE, related_name='queued_posts')
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    position = models.IntegerField()

    class Meta:
        ordering = ['position']
        unique_together = [['queue', 'position']]
```

### 2.2 Database Indexes Strategy

- **User lookups**: Index on user FK for all models
- **Scheduled posts**: Composite index on (status, scheduled_time)
- **Platform queries**: Index on platform type
- **Date ranges**: Indexes on created_at, scheduled_time, posted_at

### 2.3 Data Migration Plan

**Phase 1 (Development)**: SQLite
**Phase 2 (Production)**: PostgreSQL

Migration checklist:
- Update `DATABASES` setting
- Install `psycopg2-binary`
- Export data: `python manage.py dumpdata > data.json`
- Import to PostgreSQL: `python manage.py loaddata data.json`
- Update connection pooling (use `django-db-pool` or `pgbouncer`)

---

## 3. API Integration Layer

### 3.1 Platform Abstraction Pattern

**Strategy Pattern** for different platform implementations:

```python
# social_media/platforms/base.py
from abc import ABC, abstractmethod
from typing import Dict, Any, List, Optional

class SocialMediaPlatformBase(ABC):
    """Abstract base class for all social media integrations"""

    def __init__(self, connected_account: ConnectedAccount):
        self.account = connected_account
        self.client = self._initialize_client()

    @abstractmethod
    def _initialize_client(self):
        """Initialize platform-specific client"""
        pass

    @abstractmethod
    def post_text(self, content: str, **kwargs) -> Dict[str, Any]:
        """Post text content"""
        pass

    @abstractmethod
    def post_with_media(self, content: str, media_files: List[str], **kwargs) -> Dict[str, Any]:
        """Post content with media"""
        pass

    @abstractmethod
    def delete_post(self, platform_post_id: str) -> bool:
        """Delete a post"""
        pass

    @abstractmethod
    def get_post_analytics(self, platform_post_id: str) -> Dict[str, Any]:
        """Fetch analytics for a post"""
        pass

    @abstractmethod
    def validate_content(self, content: str, media_count: int = 0) -> Dict[str, Any]:
        """Validate content meets platform requirements"""
        pass

    @abstractmethod
    def refresh_access_token(self) -> bool:
        """Refresh OAuth token if needed"""
        pass

    def get_character_limit(self) -> int:
        """Return platform character limit"""
        return self.account.platform_settings.get('character_limit', 280)

    def handle_rate_limit(self, response):
        """Handle rate limiting"""
        # Implement exponential backoff
        pass
```

### 3.2 Platform Implementations

#### Twitter/X Integration
```python
# social_media/platforms/twitter.py
import tweepy
from .base import SocialMediaPlatformBase

class TwitterPlatform(SocialMediaPlatformBase):
    def _initialize_client(self):
        auth = tweepy.OAuthHandler(
            settings.TWITTER_API_KEY,
            settings.TWITTER_API_SECRET
        )
        auth.set_access_token(
            self.account.access_token,
            self.account.refresh_token
        )
        return tweepy.API(auth)

    def post_text(self, content: str, **kwargs) -> Dict[str, Any]:
        try:
            tweet = self.client.update_status(content)
            return {
                'success': True,
                'platform_post_id': str(tweet.id),
                'platform_post_url': f'https://twitter.com/{tweet.user.screen_name}/status/{tweet.id}',
                'posted_at': tweet.created_at
            }
        except tweepy.TweepyException as e:
            return {
                'success': False,
                'error': str(e)
            }

    def validate_content(self, content: str, media_count: int = 0) -> Dict[str, Any]:
        errors = []
        if len(content) > 280:
            errors.append(f"Content exceeds 280 characters ({len(content)})")
        if media_count > 4:
            errors.append(f"Too many media files (max 4, got {media_count})")

        return {
            'valid': len(errors) == 0,
            'errors': errors
        }
```

#### Bluesky Integration
```python
# social_media/platforms/bluesky.py
from atproto import Client
from .base import SocialMediaPlatformBase

class BlueskyPlatform(SocialMediaPlatformBase):
    def _initialize_client(self):
        client = Client()
        client.login(
            self.account.platform_username,
            self.account.access_token
        )
        return client

    def post_text(self, content: str, **kwargs) -> Dict[str, Any]:
        try:
            response = self.client.send_post(text=content)
            return {
                'success': True,
                'platform_post_id': response.uri,
                'platform_post_url': f'https://bsky.app/profile/{self.account.platform_username}/post/{response.uri.split("/")[-1]}',
                'posted_at': response.indexed_at
            }
        except Exception as e:
            return {
                'success': False,
                'error': str(e)
            }
```

#### LinkedIn Integration
```python
# social_media/platforms/linkedin.py
from linkedin_api import Linkedin
from .base import SocialMediaPlatformBase

class LinkedInPlatform(SocialMediaPlatformBase):
    def _initialize_client(self):
        # linkedin-api uses email/password or cookies for authentication
        # For OAuth flow, you'll need to use requests-oauthlib separately
        # This is a simplified example using stored credentials
        client = Linkedin(
            self.account.platform_username,
            self.account.access_token
        )
        return client

    def post_text(self, content: str, **kwargs) -> Dict[str, Any]:
        try:
            # Share text post
            response = self.client.post({
                'content': {
                    'contentEntities': [],
                    'title': content[:100],  # First 100 chars as title
                    'description': content
                },
                'distribution': {
                    'linkedInDistributionTarget': {}
                }
            })
            return {
                'success': True,
                'platform_post_id': response.get('id', ''),
                'platform_post_url': response.get('url', ''),
                'posted_at': timezone.now()
            }
        except Exception as e:
            return {
                'success': False,
                'error': str(e)
            }

    def validate_content(self, content: str, media_count: int = 0) -> Dict[str, Any]:
        errors = []
        if len(content) > 3000:
            errors.append(f"Content exceeds 3000 characters ({len(content)})")
        if media_count > 9:
            errors.append(f"Too many media files (max 9, got {media_count})")

        return {
            'valid': len(errors) == 0,
            'errors': errors
        }
```

### 3.3 Platform Factory

```python
# social_media/platforms/__init__.py
from .twitter import TwitterPlatform
from .bluesky import BlueskyPlatform
from .linkedin import LinkedInPlatform
from .mastodon import MastodonPlatform
from .instagram import InstagramPlatform

class PlatformFactory:
    _platforms = {
        'twitter': TwitterPlatform,
        'bluesky': BlueskyPlatform,
        'linkedin': LinkedInPlatform,
        'mastodon': MastodonPlatform,
        'instagram': InstagramPlatform,
    }

    @classmethod
    def get_platform(cls, connected_account: ConnectedAccount):
        platform_class = cls._platforms.get(connected_account.platform)
        if not platform_class:
            raise ValueError(f"Unsupported platform: {connected_account.platform}")
        return platform_class(connected_account)
```

### 3.4 OAuth Flow Architecture

```python
# social_media/oauth/manager.py
class OAuthManager:
    """Centralized OAuth management"""

    @staticmethod
    def get_authorization_url(platform: str, callback_url: str) -> str:
        """Generate OAuth authorization URL"""
        pass

    @staticmethod
    def handle_callback(platform: str, code: str, state: str) -> ConnectedAccount:
        """Handle OAuth callback and save tokens"""
        pass

    @staticmethod
    def refresh_token(connected_account: ConnectedAccount) -> bool:
        """Refresh expired access token"""
        pass
```

### 3.5 Error Handling Strategy

```python
# social_media/platforms/exceptions.py
class PlatformException(Exception):
    """Base exception for platform errors"""
    pass

class RateLimitException(PlatformException):
    """Rate limit exceeded"""
    def __init__(self, retry_after: int):
        self.retry_after = retry_after

class AuthenticationException(PlatformException):
    """Authentication failed - token expired or invalid"""
    pass

class ContentValidationException(PlatformException):
    """Content doesn't meet platform requirements"""
    pass

class PlatformUnavailableException(PlatformException):
    """Platform API is down or unreachable"""
    pass
```

---

## 4. Task Scheduling System

### 4.1 Celery Configuration

```python
# config/celery.py
from celery import Celery
from celery.schedules import crontab
import os

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')

app = Celery('social_media_automation')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()

# Celery Beat schedule
app.conf.beat_schedule = {
    'process-scheduled-posts': {
        'task': 'social_media.tasks.process_scheduled_posts',
        'schedule': 60.0,  # Every minute
    },
    'refresh-analytics': {
        'task': 'social_media.tasks.refresh_all_analytics',
        'schedule': crontab(hour='*/6'),  # Every 6 hours
    },
    'refresh-expired-tokens': {
        'task': 'social_media.tasks.refresh_expired_tokens',
        'schedule': crontab(hour=2, minute=0),  # Daily at 2 AM
    },
}

# Task configuration
app.conf.task_routes = {
    'social_media.tasks.post_to_platform': {'queue': 'posting'},
    'social_media.tasks.refresh_analytics': {'queue': 'analytics'},
    'social_media.tasks.process_media': {'queue': 'media'},
}

app.conf.task_acks_late = True
app.conf.worker_prefetch_multiplier = 1
app.conf.task_time_limit = 300  # 5 minutes
app.conf.task_soft_time_limit = 240  # 4 minutes
```

### 4.2 Core Tasks

```python
# social_media/tasks.py
from celery import shared_task, chain, group
from celery.utils.log import get_task_logger
from django.utils import timezone
from datetime import timedelta

logger = get_task_logger(__name__)

@shared_task(bind=True, max_retries=3)
def post_to_platform(self, post_target_id: int):
    """Post content to a specific platform"""
    try:
        post_target = PostTarget.objects.select_related('post', 'account').get(id=post_target_id)

        # Get platform client
        platform = PlatformFactory.get_platform(post_target.account)

        # Validate content
        content = post_target.custom_content or post_target.post.content
        media_files = post_target.post.media_files.all()

        validation = platform.validate_content(content, len(media_files))
        if not validation['valid']:
            post_target.status = PostStatus.FAILED
            post_target.error_message = ', '.join(validation['errors'])
            post_target.save()
            return

        # Post content
        post_target.status = PostStatus.POSTING
        post_target.last_attempt_at = timezone.now()
        post_target.save()

        if media_files:
            result = platform.post_with_media(content, media_files)
        else:
            result = platform.post_text(content)

        if result['success']:
            post_target.status = PostStatus.POSTED
            post_target.platform_post_id = result['platform_post_id']
            post_target.platform_post_url = result.get('platform_post_url')
            post_target.posted_at = result.get('posted_at', timezone.now())
            post_target.error_message = None
        else:
            raise Exception(result.get('error', 'Unknown error'))

        post_target.save()

        # Update main post status if all targets posted
        post = post_target.post
        if all(t.status == PostStatus.POSTED for t in post.targets.all()):
            post.status = PostStatus.POSTED
            post.posted_at = timezone.now()
            post.save()

        logger.info(f"Successfully posted to {post_target.account.platform}: {post_target_id}")

    except Exception as exc:
        logger.error(f"Error posting to platform: {exc}")
        post_target.status = PostStatus.FAILED
        post_target.error_message = str(exc)
        post_target.retry_count += 1
        post_target.save()

        # Retry with exponential backoff
        raise self.retry(exc=exc, countdown=60 * (2 ** post_target.retry_count))

@shared_task
def process_scheduled_posts():
    """Find and queue posts scheduled for now"""
    now = timezone.now()
    scheduled_posts = Post.objects.filter(
        status=PostStatus.SCHEDULED,
        scheduled_time__lte=now
    ).select_related('user')

    for post in scheduled_posts:
        post.status = PostStatus.QUEUED
        post.save()

        # Create tasks for each target platform
        tasks = []
        for target in post.targets.all():
            if target.account.is_active:
                tasks.append(post_to_platform.s(target.id))

        # Execute all platform posts in parallel
        if tasks:
            group(tasks).apply_async()
            logger.info(f"Queued post {post.id} for {len(tasks)} platforms")

@shared_task
def refresh_post_analytics(post_target_id: int):
    """Refresh analytics for a specific post"""
    try:
        post_target = PostTarget.objects.get(id=post_target_id)

        if post_target.status != PostStatus.POSTED:
            return

        platform = PlatformFactory.get_platform(post_target.account)
        analytics_data = platform.get_post_analytics(post_target.platform_post_id)

        # Update or create analytics
        analytics, created = PostAnalytics.objects.update_or_create(
            post_target=post_target,
            defaults=analytics_data
        )

        # Create snapshot
        AnalyticsSnapshot.objects.create(
            post_target=post_target,
            snapshot_data=analytics_data
        )

        logger.info(f"Refreshed analytics for post target {post_target_id}")

    except Exception as exc:
        logger.error(f"Error refreshing analytics: {exc}")

@shared_task
def refresh_all_analytics():
    """Refresh analytics for recent posts"""
    cutoff_date = timezone.now() - timedelta(days=7)
    recent_posts = PostTarget.objects.filter(
        status=PostStatus.POSTED,
        posted_at__gte=cutoff_date
    )

    tasks = [refresh_post_analytics.s(pt.id) for pt in recent_posts]
    group(tasks).apply_async()

    logger.info(f"Queued analytics refresh for {len(tasks)} posts")

@shared_task
def refresh_expired_tokens():
    """Refresh OAuth tokens that are about to expire"""
    expiring_soon = timezone.now() + timedelta(days=7)
    accounts = ConnectedAccount.objects.filter(
        is_active=True,
        token_expires_at__lte=expiring_soon
    )

    for account in accounts:
        try:
            platform = PlatformFactory.get_platform(account)
            platform.refresh_access_token()
            logger.info(f"Refreshed token for {account.platform} account {account.id}")
        except Exception as exc:
            logger.error(f"Failed to refresh token for account {account.id}: {exc}")
```

### 4.3 Redis Configuration

```python
# config/settings.py
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/1'
CELERY_CACHE_BACKEND = 'default'

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://localhost:6379/2',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}
```

### 4.4 Queue Strategy

- **posting**: High priority, post publishing tasks
- **analytics**: Lower priority, analytics fetching
- **media**: Medium priority, media processing
- **default**: Everything else

---

## 5. REST API Design

### 5.1 API Structure

**Base URL**: `/api/v1/`

**Endpoints**:

```
Authentication:
POST   /api/v1/auth/register/
POST   /api/v1/auth/login/
POST   /api/v1/auth/logout/
POST   /api/v1/auth/refresh/
GET    /api/v1/auth/me/

Social Accounts:
GET    /api/v1/accounts/
POST   /api/v1/accounts/connect/{platform}/
DELETE /api/v1/accounts/{id}/disconnect/
GET    /api/v1/accounts/{id}/
PATCH  /api/v1/accounts/{id}/

Posts:
GET    /api/v1/posts/
POST   /api/v1/posts/
GET    /api/v1/posts/{id}/
PATCH  /api/v1/posts/{id}/
DELETE /api/v1/posts/{id}/
POST   /api/v1/posts/{id}/schedule/
POST   /api/v1/posts/{id}/cancel/
GET    /api/v1/posts/{id}/analytics/

Media:
POST   /api/v1/media/upload/
GET    /api/v1/media/
DELETE /api/v1/media/{id}/

Queues:
GET    /api/v1/queues/
POST   /api/v1/queues/
GET    /api/v1/queues/{id}/
PATCH  /api/v1/queues/{id}/
DELETE /api/v1/queues/{id}/
POST   /api/v1/queues/{id}/add-post/
POST   /api/v1/queues/{id}/reorder/

Analytics:
GET    /api/v1/analytics/overview/
GET    /api/v1/analytics/posts/{id}/
GET    /api/v1/analytics/posts/{id}/history/
```

### 5.2 Serializers

```python
# social_media/api/serializers.py
from rest_framework import serializers
from social_media.models import Post, PostTarget, ConnectedAccount, MediaFile

class ConnectedAccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = ConnectedAccount
        fields = ['id', 'platform', 'platform_username', 'is_active', 'created_at']
        read_only_fields = ['id', 'created_at']

class MediaFileSerializer(serializers.ModelSerializer):
    class Meta:
        model = MediaFile
        fields = ['id', 'media_type', 'file', 'file_size', 'width', 'height', 'alt_text', 'created_at']
        read_only_fields = ['id', 'file_size', 'width', 'height', 'created_at']

class PostTargetSerializer(serializers.ModelSerializer):
    account = ConnectedAccountSerializer(read_only=True)

    class Meta:
        model = PostTarget
        fields = ['id', 'account', 'status', 'platform_post_url', 'error_message', 'posted_at']

class PostSerializer(serializers.ModelSerializer):
    targets = PostTargetSerializer(many=True, read_only=True)
    media_files = MediaFileSerializer(many=True, read_only=True)
    target_accounts = serializers.PrimaryKeyRelatedField(
        many=True,
        queryset=ConnectedAccount.objects.all(),
        write_only=True
    )

    class Meta:
        model = Post
        fields = [
            'id', 'content', 'status', 'scheduled_time', 'posted_at',
            'created_at', 'updated_at', 'targets', 'media_files', 'target_accounts'
        ]
        read_only_fields = ['id', 'status', 'posted_at', 'created_at', 'updated_at']

    def create(self, validated_data):
        target_accounts = validated_data.pop('target_accounts', [])
        post = Post.objects.create(**validated_data)

        # Create PostTarget entries
        for account in target_accounts:
            PostTarget.objects.create(post=post, account=account)

        return post
```

### 5.3 ViewSets

```python
# social_media/api/views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from django.utils import timezone

class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Post.objects.filter(user=self.request.user).prefetch_related('targets', 'media_files')

    def perform_create(self, serializer):
        serializer.save(user=self.request.user)

    @action(detail=True, methods=['post'])
    def schedule(self, request, pk=None):
        """Schedule a post for publishing"""
        post = self.get_object()
        scheduled_time = request.data.get('scheduled_time')

        if not scheduled_time:
            return Response({'error': 'scheduled_time is required'}, status=status.HTTP_400_BAD_REQUEST)

        post.scheduled_time = scheduled_time
        post.status = PostStatus.SCHEDULED
        post.save()

        return Response(self.get_serializer(post).data)

    @action(detail=True, methods=['post'])
    def cancel(self, request, pk=None):
        """Cancel a scheduled post"""
        post = self.get_object()

        if post.status != PostStatus.SCHEDULED:
            return Response({'error': 'Only scheduled posts can be cancelled'}, status=status.HTTP_400_BAD_REQUEST)

        post.status = PostStatus.CANCELLED
        post.save()

        return Response(self.get_serializer(post).data)

    @action(detail=True, methods=['get'])
    def analytics(self, request, pk=None):
        """Get analytics for a post"""
        post = self.get_object()

        analytics_data = []
        for target in post.targets.all():
            if hasattr(target, 'analytics'):
                analytics_data.append({
                    'platform': target.account.platform,
                    'likes': target.analytics.likes,
                    'shares': target.analytics.retweets,
                    'replies': target.analytics.replies,
                    'impressions': target.analytics.impressions,
                })

        return Response(analytics_data)
```

### 5.4 Authentication

**Strategy**: JWT (JSON Web Tokens)

```python
# config/settings.py
INSTALLED_APPS += [
    'rest_framework',
    'rest_framework_simplejwt',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}

from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(hours=1),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
}
```

---

## 6. Media Management

### 6.1 Storage Strategy

**Development**: Local filesystem
**Production**: AWS S3

```python
# config/settings.py
if DEBUG:
    # Local storage
    MEDIA_URL = '/media/'
    MEDIA_ROOT = BASE_DIR / 'media'
else:
    # S3 storage
    DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
    AWS_ACCESS_KEY_ID = os.environ.get('AWS_ACCESS_KEY_ID')
    AWS_SECRET_ACCESS_KEY = os.environ.get('AWS_SECRET_ACCESS_KEY')
    AWS_STORAGE_BUCKET_NAME = os.environ.get('AWS_STORAGE_BUCKET_NAME')
    AWS_S3_REGION_NAME = 'us-east-1'
    AWS_S3_FILE_OVERWRITE = False
    AWS_DEFAULT_ACL = 'private'
    AWS_QUERYSTRING_AUTH = True
    AWS_S3_SIGNATURE_VERSION = 's3v4'
```

### 6.2 Media Processing

```python
# social_media/media/processor.py
from PIL import Image
import ffmpeg
import os

class MediaProcessor:

    @staticmethod
    def process_image(file_path: str, max_size: tuple = (4096, 4096)) -> dict:
        """Process and optimize image"""
        with Image.open(file_path) as img:
            # Get dimensions
            width, height = img.size

            # Resize if too large
            if width > max_size[0] or height > max_size[1]:
                img.thumbnail(max_size, Image.Resampling.LANCZOS)
                img.save(file_path, optimize=True, quality=85)

            return {
                'width': img.width,
                'height': img.height,
                'format': img.format,
            }

    @staticmethod
    def process_video(file_path: str, max_duration: int = 140) -> dict:
        """Process video and extract metadata"""
        probe = ffmpeg.probe(file_path)
        video_info = next(s for s in probe['streams'] if s['codec_type'] == 'video')

        duration = float(probe['format']['duration'])
        width = int(video_info['width'])
        height = int(video_info['height'])

        # Trim if too long
        if duration > max_duration:
            output_path = file_path.replace('.mp4', '_trimmed.mp4')
            ffmpeg.input(file_path, t=max_duration).output(output_path).run()
            os.replace(output_path, file_path)
            duration = max_duration

        return {
            'duration': duration,
            'width': width,
            'height': height,
        }
```

### 6.3 File Upload API

```python
# social_media/api/views.py
from rest_framework.parsers import MultiPartParser
from rest_framework.views import APIView

class MediaUploadView(APIView):
    parser_classes = [MultiPartParser]

    def post(self, request):
        file = request.FILES.get('file')
        post_id = request.data.get('post_id')
        alt_text = request.data.get('alt_text', '')

        if not file:
            return Response({'error': 'No file provided'}, status=400)

        # Validate file type
        allowed_types = ['image/jpeg', 'image/png', 'image/gif', 'video/mp4']
        if file.content_type not in allowed_types:
            return Response({'error': 'Unsupported file type'}, status=400)

        # Determine media type
        if file.content_type.startswith('image'):
            media_type = MediaFile.MediaType.IMAGE
        elif file.content_type.startswith('video'):
            media_type = MediaFile.MediaType.VIDEO

        # Create media file
        media_file = MediaFile.objects.create(
            user=request.user,
            post_id=post_id,
            media_type=media_type,
            original_filename=file.name,
            file=file,
            file_size=file.size,
            mime_type=file.content_type,
            alt_text=alt_text
        )

        # Process in background
        from social_media.tasks import process_media_file
        process_media_file.delay(media_file.id)

        return Response(MediaFileSerializer(media_file).data, status=201)
```

---

## 7. Authentication & Security

### 7.1 Security Checklist

- [ ] **Environment Variables**: All secrets in `.env` file (never in code)
- [ ] **OAuth Token Encryption**: Use `django-cryptography` for storing tokens
- [ ] **HTTPS Only**: Force HTTPS in production
- [ ] **CSRF Protection**: Enabled for all state-changing operations
- [ ] **SQL Injection**: Use Django ORM (parameterized queries)
- [ ] **XSS Protection**: Sanitize user input, use Content Security Policy
- [ ] **Rate Limiting**: Use `django-ratelimit` or API throttling
- [ ] **CORS**: Whitelist frontend domain only

### 7.2 Sensitive Data Encryption

```python
# social_media/models.py
from django_cryptography.fields import encrypt

class ConnectedAccount(models.Model):
    access_token = encrypt(models.TextField())
    refresh_token = encrypt(models.TextField(null=True, blank=True))
```

### 7.3 API Rate Limiting

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour'
    }
}
```

### 7.4 Environment Variables

```bash
# .env.example
DEBUG=False
SECRET_KEY=your-secret-key-here
DATABASE_URL=postgresql://user:pass@localhost/dbname

# Social Media API Keys
TWITTER_API_KEY=
TWITTER_API_SECRET=
TWITTER_BEARER_TOKEN=

LINKEDIN_CLIENT_ID=
LINKEDIN_CLIENT_SECRET=

# AWS
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_STORAGE_BUCKET_NAME=

# Redis
REDIS_URL=redis://localhost:6379/0

# Celery
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/1
```

---

## 8. Deployment Architecture

### 8.1 Docker Setup

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: social_media_automation
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  web:
    build: .
    command: gunicorn config.wsgi:application --bind 0.0.0.0:8000
    volumes:
      - .:/app
      - static_volume:/app/staticfiles
      - media_volume:/app/media
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - db
      - redis

  celery:
    build: .
    command: celery -A config worker -l info
    volumes:
      - .:/app
    env_file:
      - .env
    depends_on:
      - db
      - redis

  celery-beat:
    build: .
    command: celery -A config beat -l info
    volumes:
      - .:/app
    env_file:
      - .env
    depends_on:
      - db
      - redis

volumes:
  postgres_data:
  redis_data:
  static_volume:
  media_volume:
```

**Dockerfile**:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    postgresql-client \
    gcc \
    python3-dev \
    musl-dev \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy project
COPY . .

# Collect static files
RUN python manage.py collectstatic --noinput

EXPOSE 8000

CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

### 8.2 AWS Deployment

**Services**:
- **ECS (Elastic Container Service)**: Run Docker containers
- **RDS (PostgreSQL)**: Managed database
- **ElastiCache (Redis)**: Managed Redis
- **S3**: Media storage
- **CloudFront**: CDN for static/media files
- **ALB (Application Load Balancer)**: Load balancing
- **Route 53**: DNS management
- **ACM**: SSL certificates

### 8.3 CI/CD Pipeline (GitHub Actions)

**.github/workflows/deploy.yml**:
```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Run tests
        run: pytest
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost/test_db

      - name: Run linting
        run: |
          flake8 .
          black --check .

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: social-media-automation
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

      - name: Update ECS service
        run: |
          aws ecs update-service --cluster production --service web --force-new-deployment
```

---

## 9. Testing Strategy

### 9.1 Testing Pyramid

```
     /\
    /  \  E2E Tests (5%)
   /____\
  /      \
 / Unit   \ Integration Tests (25%)
/  Tests   \
/___________\ Unit Tests (70%)
```

### 9.2 Unit Tests

```python
# social_media/tests/test_models.py
import pytest
from django.contrib.auth import get_user_model
from social_media.models import Post, ConnectedAccount, PostTarget

User = get_user_model()

@pytest.fixture
def user():
    return User.objects.create_user(username='testuser', password='testpass')

@pytest.fixture
def connected_account(user):
    return ConnectedAccount.objects.create(
        user=user,
        platform='twitter',
        platform_user_id='12345',
        platform_username='testuser',
        access_token='test_token'
    )

@pytest.mark.django_db
class TestPost:
    def test_create_post(self, user):
        post = Post.objects.create(
            user=user,
            content='Test post content'
        )
        assert post.status == PostStatus.DRAFT
        assert post.content == 'Test post content'

    def test_post_with_targets(self, user, connected_account):
        post = Post.objects.create(user=user, content='Test')
        PostTarget.objects.create(post=post, account=connected_account)

        assert post.targets.count() == 1
        assert post.targets.first().account == connected_account
```

### 9.3 Integration Tests

```python
# social_media/tests/test_api.py
import pytest
from rest_framework.test import APIClient
from django.contrib.auth import get_user_model

User = get_user_model()

@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def authenticated_client(api_client):
    user = User.objects.create_user(username='testuser', password='testpass')
    api_client.force_authenticate(user=user)
    return api_client, user

@pytest.mark.django_db
class TestPostAPI:
    def test_create_post(self, authenticated_client):
        client, user = authenticated_client

        response = client.post('/api/v1/posts/', {
            'content': 'Test post',
            'target_accounts': []
        })

        assert response.status_code == 201
        assert response.data['content'] == 'Test post'

    def test_list_posts(self, authenticated_client):
        client, user = authenticated_client

        # Create test posts
        Post.objects.create(user=user, content='Post 1')
        Post.objects.create(user=user, content='Post 2')

        response = client.get('/api/v1/posts/')

        assert response.status_code == 200
        assert len(response.data['results']) == 2
```

### 9.4 Platform Integration Tests (Mocked)

```python
# social_media/tests/test_platforms.py
import pytest
from unittest.mock import Mock, patch
from social_media.platforms import TwitterPlatform

@pytest.mark.django_db
class TestTwitterPlatform:
    @patch('social_media.platforms.twitter.tweepy.API')
    def test_post_text(self, mock_api, connected_account):
        platform = TwitterPlatform(connected_account)

        # Mock response
        mock_tweet = Mock()
        mock_tweet.id = 123456
        mock_tweet.user.screen_name = 'testuser'
        mock_api.return_value.update_status.return_value = mock_tweet

        result = platform.post_text('Test tweet')

        assert result['success'] == True
        assert result['platform_post_id'] == '123456'
```

### 9.5 Test Configuration

```python
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings_test
python_files = tests.py test_*.py *_tests.py
addopts = --cov=social_media --cov-report=html --cov-report=term

# requirements-dev.txt
pytest==7.4.3
pytest-django==4.7.0
pytest-cov==4.1.0
factory-boy==3.3.0
faker==20.1.0
```

---

## 10. Performance Considerations

### 10.1 Database Optimization

- **Connection Pooling**: Use `pgbouncer` or `django-db-pool`
- **Query Optimization**: Use `select_related()` and `prefetch_related()`
- **Indexes**: Add indexes on frequently queried fields
- **Database Replicas**: Read replicas for analytics queries

```python
# Example optimized query
posts = Post.objects.select_related('user').prefetch_related(
    'targets__account',
    'media_files',
    'targets__analytics'
).filter(user=request.user)
```

### 10.2 Caching Strategy

```python
# Cache API responses
from django.core.cache import cache

def get_user_posts(user_id):
    cache_key = f'user_posts_{user_id}'
    posts = cache.get(cache_key)

    if posts is None:
        posts = Post.objects.filter(user_id=user_id)
        cache.set(cache_key, posts, timeout=300)  # 5 minutes

    return posts
```

### 10.3 Celery Optimization

- **Task Routing**: Separate queues for different task types
- **Task Prioritization**: High priority for user-initiated actions
- **Batch Processing**: Group similar tasks together
- **Result Expiration**: Clean up old task results

### 10.4 API Rate Limiting

Implement per-user and per-platform rate limiting to prevent abuse and stay within API quotas.

---

## 11. Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
- [x] Django project setup
- [ ] Database models implementation
- [ ] User authentication (JWT)
- [ ] Basic REST API structure
- [ ] Twitter integration
- [ ] Bluesky integration

### Phase 2: Core Features (Weeks 3-4)
- [ ] LinkedIn integration
- [ ] Mastodon integration
- [ ] Media upload and processing
- [ ] Celery task queue setup
- [ ] Post scheduling functionality
- [ ] Basic error handling and retry logic

### Phase 3: Dashboard & UX (Week 5)
- [ ] React/Vue frontend setup
- [ ] Dashboard UI
- [ ] Post composer
- [ ] Account connection flow
- [ ] Calendar view for scheduled posts

### Phase 4: Analytics & Polish (Week 6+)
- [ ] Analytics integration
- [ ] Queue management
- [ ] Multi-account support
- [ ] Performance optimization
- [ ] Comprehensive testing
- [ ] Documentation

### Phase 5: Deployment (Week 7)
- [ ] Docker containerization
- [ ] AWS infrastructure setup
- [ ] CI/CD pipeline
- [ ] Monitoring and logging
- [ ] Production deployment

---

## Appendix

### A. Technology Versions

- Python: 3.11+ (3.12+ required for Django 6.0)
- Django: 5.1
- PostgreSQL: 15
- Redis: 7
- Celery: 5.3+
- Django REST Framework: 3.14+

### B. Useful Resources

- [Django Documentation](https://docs.djangoproject.com/)
- [Celery Documentation](https://docs.celeryproject.org/)
- [Twitter API v2 Docs](https://developer.twitter.com/en/docs/twitter-api)
- [Bluesky ATP Protocol](https://atproto.com/)
- [Django REST Framework](https://www.django-rest-framework.org/)

### C. Development Commands

```bash
# Start development server
python manage.py runserver

# Run migrations
python manage.py makemigrations
python manage.py migrate

# Create superuser
python manage.py createsuperuser

# Start Celery worker
celery -A config worker -l info

# Start Celery beat
celery -A config beat -l info

# Run tests
pytest

# Code formatting
black .
flake8 .
```

---

**End of Architecture Document**
